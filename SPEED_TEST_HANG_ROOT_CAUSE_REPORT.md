# ОТЧЁТ: ДИАГНОСТИКА ЗАВИСАНИЯ ТЕСТА SPEED OK

**Дата исследования:** 2025-01-10  
**Проблема:** Тест SPEED OK зависает и не завершается  
**Статус:** ✅ ROOT CAUSE НАЙДЕН

---

## 🔍 СИМПТОМЫ

1. При запуске теста SPEED OK на узлах со статусом `ping_ok`, тест **не выполняется**
2. Progress остаётся на `0/N` и не двигается
3. Через 60+ секунд тест всё ещё не завершён
4. В логах backend: **"Skipping node X (dedupe speed)"** для ВСЕХ узлов
5. Итоговый результат: **"0 processed, 0 failed"**

---

## 🐛 ROOT CAUSE: АГРЕССИВНАЯ ДЕДУПЛИКАЦИЯ

### Обнаруженная проблема в `/app/backend/server.py`

**Строка 181:**
```python
TEST_DEDUPE_TTL = 180  # seconds
```

**Строки 185-192:**
```python
def test_dedupe_should_skip(node_id: int, mode: str) -> bool:
    now = datetime.utcnow().timestamp()
    exp = _test_recent.get((node_id, mode))
    if exp and exp > now:
        return True  # ❌ ПРОБЛЕМА: Блокирует повторное тестирование
    if node_id in _test_inflight:
        return True
    return False
```

**Строки 194-197:**
```python
def test_dedupe_mark_enqueued(node_id: int, mode: str):
    now = datetime.utcnow().timestamp()
    _test_recent[(node_id, mode)] = now + TEST_DEDUPE_TTL  # ❌ Блокировка на 180 секунд
    _test_inflight.add(node_id)
```

---

## 📊 ЧТО ПРОИСХОДИТ

### Сценарий зависания:

1. **T=0**: Узел `node_id=6` тестируется через PING OK тест
2. **T=0**: Система вызывает `test_dedupe_mark_enqueued(6, "speed")` 
3. **T=0**: В `_test_recent` сохраняется: `(6, "speed") -> timestamp + 180 секунд`
4. **T=120 секунд (2 минуты)**: Пользователь запускает SPEED OK тест на узле 6
5. **T=120**: Система проверяет `test_dedupe_should_skip(6, "speed")`
6. **T=120**: Обнаруживает что `exp = timestamp + 180` > `now = 120`
7. **T=120**: ❌ **УЗЕЛ ПРОПУСКАЕТСЯ**: "Skipping node 6 (dedupe speed)"
8. **Результат**: 0 узлов обработано, тест "зависает" (на самом деле завершается мгновенно с 0 результатов)

---

## 🔬 ПОДТВЕРЖДЕНИЕ

### Лог backend при запуске SPEED OK теста:

```
INFO:server:🚀 Testing Batch: Starting 5 nodes in batches of 50, mode: speed_only
INFO:server:📦 Testing batch 1: nodes 1-5
INFO:server:⏭️ Testing: Skipping node 6 (dedupe speed)
INFO:server:⏭️ Testing: Skipping node 98 (dedupe speed)
INFO:server:⏭️ Testing: Skipping node 180 (dedupe speed)
INFO:server:⏭️ Testing: Skipping node 239 (dedupe speed)
INFO:server:⏭️ Testing: Skipping node 252 (dedupe speed)
INFO:server:✅ Testing batch 1 completed: 5 nodes scheduled
INFO:server:✅ Testing batch 1 completed: 5 nodes processed
INFO:server:📊 Testing batch processing completed: 0 processed, 0 failed
```

### Проверка времени последнего обновления узлов:

| Node ID | Last Update         | Прошло времени | Dedupe TTL | Результат  |
|---------|---------------------|----------------|------------|------------|
| 6       | 05:07:03            | 2 минуты       | 3 минуты   | ❌ Blocked |
| 98      | 05:07:05            | 2 минуты       | 3 минуты   | ❌ Blocked |
| 180     | 05:07:04            | 2 минуты       | 3 минуты   | ❌ Blocked |
| 239     | 05:07:05            | 2 минуты       | 3 минуты   | ❌ Blocked |
| 252     | 05:07:05            | 2 минуты       | 3 минуты   | ❌ Blocked |

**Вывод:** Все узлы заблокированы дедупликацией, так как прошло меньше 180 секунд с момента последнего теста.

---

## ⚠️ КРИТИЧЕСКИЕ ПРОБЛЕМЫ ЛОГИКИ

### Проблема 1: Неправильное использование dedupe для разных типов тестов

**Текущее поведение:**
- Когда запускается PING OK тест, в `_test_recent` сохраняется `(node_id, "speed")`
- Затем при запуске SPEED OK теста, система проверяет ту же самую пару `(node_id, "speed")`
- **Результат**: SPEED OK тест блокируется из-за предыдущего PING OK теста

**Строка 3704:**
```python
mode_key = "ping" if testing_mode in ["ping_only", "ping_speed"] else ("speed" if testing_mode in ["speed_only"] else testing_mode)
```

**Проблема:** 
- `testing_mode = "ping_speed"` → `mode_key = "ping"`  ✅
- `testing_mode = "speed_only"` → `mode_key = "speed"` ✅
- НО: оба режима могут конфликтовать, если `mode_key` одинаковый!

### Проблема 2: TTL слишком длинный (180 секунд)

**Текущее значение:** `TEST_DEDUPE_TTL = 180 секунд (3 минуты)`

**Проблема:**
- Пользователь не может запустить повторный тест раньше чем через 3 минуты
- Для диагностики и отладки это создаёт "зависание" системы
- Пользователь не видит сообщения об ошибке, только отсутствие прогресса

### Проблема 3: Нет обратной связи пользователю

**Текущее поведение:**
- Все узлы пропускаются молча (только в логах backend)
- Progress API показывает `status: "completed"` с `0 processed`
- Frontend не видит причину (dedupe blocking)

**Результат:** Пользователь думает что система зависла, хотя на самом деле тест завершился мгновенно

---

## 📋 ДОПОЛНИТЕЛЬНЫЕ НАБЛЮДЕНИЯ

### Некорректное определение mode_key

**Строка 3704:**
```python
mode_key = "ping" if testing_mode in ["ping_only", "ping_speed"] else ("speed" if testing_mode in ["speed_only"] else testing_mode)
```

**Анализ:**
- `testing_mode = "ping_only"` → `mode_key = "ping"` ✅
- `testing_mode = "ping_speed"` → `mode_key = "ping"` ⚠️ (должно быть `"ping"` и `"speed"`)
- `testing_mode = "speed_only"` → `mode_key = "speed"` ✅
- `testing_mode = "no_test"` → `mode_key = "no_test"` ✅

**Проблема:** Режим `"ping_speed"` блокирует только по ключу `"ping"`, но не по `"speed"`

### Очистка dedupe registry

**Строки 3724-3727:**
```python
try:
    test_dedupe_cleanup()
except Exception:
    pass
```

**Функция cleanup:**
```python
def test_dedupe_cleanup():
    now = datetime.utcnow().timestamp()
    to_del = [k for k, exp in _test_recent.items() if exp <= now]
    for k in to_del:
        _test_recent.pop(k, None)
```

**Проблема:** Cleanup вызывается только после каждого batch, но если все узлы пропущены (как в нашем случае), то `_test_recent` не очищается достаточно быстро

---

## 🎯 ИТОГОВЫЕ ВЫВОДЫ

### ❌ Основная причина зависания:

**АГРЕССИВНАЯ ДЕДУПЛИКАЦИЯ С TTL=180 СЕКУНД**

1. При любом тесте узла, он блокируется на 3 минуты
2. Если пользователь пытается запустить SPEED OK тест раньше чем через 3 минуты после PING OK теста - ВСЕ узлы пропускаются
3. Система завершается мгновенно с результатом "0 processed"
4. Frontend/пользователь видит это как "зависание", так как нет feedback

### ❌ Вторичные проблемы:

1. **Неправильная логика mode_key**: не различает ping и speed в режиме `ping_speed`
2. **Нет обратной связи**: пользователь не знает о dedupe blocking
3. **Слишком большой TTL**: 180 секунд - чрезмерно для production системы
4. **Конфликт между типами тестов**: PING OK блокирует SPEED OK

---

## 🔧 РЕКОМЕНДАЦИИ ПО ИСПРАВЛЕНИЮ

### 1. Уменьшить TEST_DEDUPE_TTL

**Рекомендация:** `TEST_DEDUPE_TTL = 30` секунд (вместо 180)

**Обоснование:**
- 30 секунд достаточно чтобы предотвратить случайные дубликаты
- Не блокирует пользователя на длительное время
- Позволяет быструю диагностику и отладку

### 2. Раздельная дедупликация для ping и speed

**Текущая проблема:** Один узел - один ключ блокировки

**Решение:** Использовать раздельные ключи:
- `(node_id, "ping")` для PING тестов
- `(node_id, "speed")` для SPEED тестов

**Преимущество:** PING OK и SPEED OK не блокируют друг друга

### 3. Добавить feedback для dedupe

**Решение:** Изменить прогресс tracking:
```python
if test_dedupe_should_skip(node_id, mode_key):
    progress_increment(session_id, f"⏭️ Узел {node_id} недавно тестировался, пропускаем (ожидайте {remaining}с)")
```

### 4. Добавить force_retest параметр

**Решение:** Добавить опцию `force_retest=True` в API, которая игнорирует dedupe

**Преимущество:** Админ может принудительно перезапустить тест

### 5. Исправить логику mode_key для ping_speed

**Текущее:**
```python
mode_key = "ping" if testing_mode in ["ping_only", "ping_speed"] else ("speed" if testing_mode in ["speed_only"] else testing_mode)
```

**Правильное:**
```python
if testing_mode == "ping_only":
    mode_keys = ["ping"]
elif testing_mode == "speed_only":
    mode_keys = ["speed"]
elif testing_mode == "ping_speed":
    mode_keys = ["ping", "speed"]  # Блокировать по обоим ключам
else:
    mode_keys = [testing_mode]

for mode_key in mode_keys:
    if test_dedupe_should_skip(node_id, mode_key):
        # skip
```

---

## 📌 ЗАКЛЮЧЕНИЕ

**Зависание теста SPEED OK вызвано НЕ багом в коде измерения скорости, а АГРЕССИВНОЙ системой дедупликации, которая блокирует повторное тестирование узлов на 180 секунд.**

**Это не настоящее зависание** - тест завершается мгновенно, но с результатом "0 processed", что выглядит как зависание для пользователя.

**Исправления требуют:**
1. Уменьшения TTL до 30 секунд
2. Раздельной дедупликации для ping/speed
3. Добавления feedback о dedupe blocking
4. Опциональной возможности force_retest

**Без этих исправлений:** Система будет продолжать "зависать" при любой попытке протестировать узлы раньше чем через 3 минуты после предыдущего теста.
