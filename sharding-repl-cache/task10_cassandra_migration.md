# Задание 10. Миграция на Cassandra: модель данных, стратегии репликации и шардирования

---

## Задание 10.1. Анализ данных: что переносить в Cassandra

### Почему MongoDB не справилась при пике

Range-based шардирование MongoDB при добавлении нового шарда запускает
**балансировщик**, который физически перемещает чанки между узлами. При 50 000
запросов/сек. это означает одновременно: обслуживать трафик + копировать
гигабайты данных - просадка latency в пик.

Cassandra использует **consistent hashing**: новый узел забирает часть токен-диапазона
у соседей, перемещая только свою долю данных. Остальные узлы продолжают работу без
изменений - без глобального перераспределения.

### Анализ сущностей

| Сущность | Тип нагрузки | Требование к консистентности | Cassandra? |
|---|---|---|---|
| **Корзины (carts)** | Высокая запись/чтение, TTL | Eventual - устаревший товар в корзине некритичен | **Да** |
| **Сессии пользователей** | Очень высокая запись, TTL | Eventual - потеря сессии = повторный логин | **Да** |
| **История заказов** | Append-only, высокое чтение | Eventual - просмотр истории не требует строгой свежести | **Да** |
| **События статусов заказов** | Высокая запись, временные ряды | Eventual с периодическим repair | **Да** |
| **Остатки товаров (stock)** | Очень высокая запись при покупках | Строгая - риск overselling | Частично* |
| **Каталог товаров** | Низкая запись, высокое чтение | Eventual - изменение описания некритично | Опционально |
| **Создание заказа (транзакция)** | Средняя запись | Строгая - финансовые данные, атомарность | **Нет** |

> \* Остатки: Cassandra поддерживает легковесные транзакции (LWT / `IF` условия),
> но они в 4-5x медленнее обычных операций. Остатки лучше вести в MongoDB,
> а в Cassandra хранить денормализованную копию для отображения на странице товара.

### Итог: что переносим

1. **carts** - высокая запись, TTL, eventual consistency приемлем
2. **user_sessions** - высокая запись, TTL, простой доступ по ключу
3. **orders_by_customer** (история) - append-only, временной ряд по пользователю
4. **order_events** (статусы) - append-only, временной ряд по заказу
5. **product_stock_display** - денормализованная копия остатков для страницы товара

---

## Задание 10.2. Модель данных в Cassandra

> **Правило Cassandra:** модель строится под запросы, а не под сущности.
> Один запрос - одна таблица. Денормализация - норма.

### 2.1 Корзины: `cart_items`

**Запросы:**
- Получить все товары корзины по `cart_id`
- Добавить / обновить количество конкретного товара
- Удалить товар из корзины

```cql
CREATE TYPE IF NOT EXISTS cart_item_meta (
    name     TEXT,
    price    DECIMAL
);

CREATE TABLE cart_items (
    cart_id    UUID,
    product_id TEXT,
    quantity   INT,
    meta       FROZEN<cart_item_meta>,
    added_at   TIMESTAMP,
    PRIMARY KEY ((cart_id), product_id)
) WITH default_time_to_live = 604800
  AND comment = 'Товары в корзине. Каждый product_id - отдельная строка для атомарных обновлений';
```

Поиск корзины по `user_id` / `session_id` - через отдельные lookup-таблицы:

```cql
CREATE TABLE carts_by_user (
    user_id TEXT,
    status  TEXT,
    cart_id UUID,
    PRIMARY KEY ((user_id), status)
) WITH default_time_to_live = 604800;

CREATE TABLE carts_by_session (
    session_id TEXT,
    status     TEXT,
    cart_id    UUID,
    PRIMARY KEY ((session_id), status)
) WITH default_time_to_live = 86400;
```

**Partition key:** `cart_id` - UUID, высокая кардинальность, равномерный хеш.

**Почему не `LIST` для items:** конкурентные обновления списка создают tombstone-проблемы
и требуют read-before-write. Отдельная строка на товар позволяет атомарно обновить
количество без чтения.

**Горячие партиции:** исключены - каждый `cart_id` уникален, корзина одного
пользователя содержит десятки строк, не миллионы.

---

### 2.2 Сессии пользователей: `user_sessions`

**Запросы:**
- Получить сессию по `session_id`
- Обновить время последней активности
- Автоматически удалить истёкшие сессии

```cql
CREATE TABLE user_sessions (
    session_id  TEXT,
    user_id     TEXT,
    created_at  TIMESTAMP,
    last_active TIMESTAMP,
    data        MAP<TEXT, TEXT>,
    PRIMARY KEY (session_id)
) WITH default_time_to_live = 86400
  AND comment = 'Пользовательские сессии. TTL автоматически удаляет истёкшие';
```

**Partition key:** `session_id` - уникальный токен (UUID), идеально равномерное распределение.

**Горячие партиции:** невозможны - каждая сессия независима.

---

### 2.3 История заказов: `orders_by_customer`

**Запросы:**
- Получить историю заказов пользователя (последние N, с пагинацией)
- Получить конкретный заказ по `order_id`

```cql
CREATE TYPE IF NOT EXISTS order_item (
    product_id TEXT,
    name       TEXT,
    quantity   INT,
    price      DECIMAL
);

CREATE TABLE orders_by_customer (
    customer_id  TEXT,
    created_at   TIMESTAMP,
    order_id     UUID,
    status       TEXT,
    total_amount DECIMAL,
    geo_zone     TEXT,
    items        LIST<FROZEN<order_item>>,
    PRIMARY KEY ((customer_id), created_at, order_id)
) WITH CLUSTERING ORDER BY (created_at DESC, order_id ASC)
  AND comment = 'История заказов. created_at DESC - новые заказы первыми';

-- Поиск заказа по order_id (для страницы заказа)
CREATE TABLE orders_by_id (
    order_id     UUID,
    customer_id  TEXT,
    created_at   TIMESTAMP,
    status       TEXT,
    total_amount DECIMAL,
    geo_zone     TEXT,
    items        LIST<FROZEN<order_item>>,
    PRIMARY KEY (order_id)
);
```

**Partition key:** `customer_id` - все заказы пользователя в одной партиции,
запрос истории = один partition read без scatter-gather.

**Clustering key:** `(created_at DESC, order_id)` - сортировка на диске,
пагинация через `WHERE created_at < :last_seen_ts`.

**Горячие партиции:** в B2C-магазине у одного покупателя сотни заказов - это
ничтожный объём для Cassandra. Если бы это была B2B-платформа с миллионами заказов
на клиента, добавили бы `year_month` в partition key:
`PRIMARY KEY ((customer_id, year_month), created_at, order_id)`.

---

### 2.4 События статусов заказов: `order_events`

**Запросы:**
- Получить все события по заказу (полная история изменений статуса)
- Получить последний статус заказа

```cql
CREATE TABLE order_events (
    order_id    UUID,
    event_time  TIMESTAMP,
    event_type  TEXT,
    status      TEXT,
    actor       TEXT,
    details     TEXT,
    PRIMARY KEY ((order_id), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC)
  AND comment = 'Лог событий заказа. Append-only, события никогда не удаляются';
```

**Partition key:** `order_id` - один заказ = одна партиция. Количество событий
на заказ ограничено (5-20 статусных переходов) - партиция никогда не вырастет
до проблемных размеров.

---

### 2.5 Остатки для отображения: `product_stock_display`

**Запросы:**
- Показать остаток товара в геозоне на странице товара

```cql
CREATE TABLE product_stock_display (
    product_id TEXT,
    geo_zone   TEXT,
    quantity   INT,
    updated_at TIMESTAMP,
    PRIMARY KEY ((product_id), geo_zone)
) WITH comment = 'Денормализованная копия остатков для чтения. Источник правды - MongoDB';
```

**Partition key:** `product_id` - запрос по конкретному товару всегда попадает
на один узел. Количество строк на партицию = количество геозон (~10).

**Горячие партиции:** возможны для суперпопулярного товара в «Чёрную пятницу».
Компенсируется кешированием в Redis (TTL 10-30 сек для отображения остатков).

---

### Итог: распределение данных

| Таблица | Partition Key | Ключевое свойство |
|---|---|---|
| `cart_items` | `cart_id` (UUID) | Уникален, равномерный хеш |
| `carts_by_user` | `user_id` | Активных корзин 1-2 на пользователя |
| `carts_by_session` | `session_id` | Уникален, равномерный хеш |
| `user_sessions` | `session_id` | Уникален, TTL |
| `orders_by_customer` | `customer_id` | B2C: сотни заказов = небольшая партиция |
| `orders_by_id` | `order_id` (UUID) | Уникален, равномерный хеш |
| `order_events` | `order_id` (UUID) | Ограниченный размер партиции |
| `product_stock_display` | `product_id` | Малое число геозон, Redis-кеш поверх |

---

## Задание 10.3. Стратегии обеспечения целостности данных

### Механизмы Cassandra

| Механизм | Как работает | Когда срабатывает |
|---|---|---|
| **Hinted Handoff** | Координатор сохраняет «подсказку» для недоступного узла и доставляет запись, когда узел возвращается | Узел временно недоступен во время записи |
| **Read Repair** | При чтении координатор сравнивает ответы реплик; если есть расхождение - исправляет устаревшие реплики | Во время каждого чтения (foreground или background) |
| **Anti-Entropy Repair** | Полное сравнение данных между узлами через деревья Меркла (`nodetool repair`); находит все расхождения | По расписанию (cron), вручную |

---

### Выбор стратегий по сущностям

#### Сессии (`user_sessions`) и Корзины (`cart_items`)

**Стратегия: Hinted Handoff + фоновый Read Repair**

```cql
-- Запись с консистентностью ONE (максимальная скорость)
-- Если узел упал - координатор сохранит hint и доставит запись при восстановлении
CONSISTENCY ONE;
INSERT INTO user_sessions (session_id, user_id, ...) VALUES (...);

CONSISTENCY LOCAL_ONE;
SELECT * FROM cart_items WHERE cart_id = ?;
```

**Обоснование:**
- TTL данных (24 ч / 7 дней) короче, чем окно хранения хинтов (по умолчанию 3 ч)
- Потеря сессии = повторный логин. Потеря корзины = редкий крайний случай
- Hinted Handoff восстанавливает пропущенные записи автоматически при кратковременных сбоях
- Read Repair в фоне не блокирует запрос и постепенно выравнивает реплики
- Anti-Entropy Repair нецелесообразен: данные быстро истекают по TTL

**Настройка:**

```yaml
# cassandra.yaml
hinted_handoff_enabled: true
max_hint_window_in_ms: 10800000   # 3 часа
hinted_handoff_throttle_in_kb: 1024

dc_local_read_repair_chance: 0.1  # 10% фоновых чтений триггерят repair
read_repair_chance: 0.0           # foreground repair отключён
```

---

#### История заказов (`orders_by_customer`, `order_events`)

**Стратегия: QUORUM + Anti-Entropy Repair по расписанию**

```cql
-- Запись с QUORUM: большинство реплик подтверждает
CONSISTENCY LOCAL_QUORUM;
INSERT INTO orders_by_customer (...) VALUES (...);
INSERT INTO order_events (...) VALUES (...);

-- Чтение с QUORUM: всегда актуальные данные
CONSISTENCY LOCAL_QUORUM;
SELECT * FROM orders_by_customer WHERE customer_id = ?;
```

**Обоснование:**
- Данные о заказах - финансовая информация. Пользователь не должен видеть
  «пропавший» заказ из-за временного расхождения реплик
- LOCAL_QUORUM сохраняет низкую latency внутри дата-центра
- Anti-Entropy Repair запускается еженедельно в ночное время - гарантирует,
  что все узлы имеют идентичную копию исторических данных
- Hinted Handoff продолжает работать как первый уровень защиты при сбоях

```bash
# Запуск Anti-Entropy Repair по расписанию (cron, еженедельно)
# Инкрементальный режим - только изменения с прошлого repair
0 3 * * 0 nodetool repair --incremental somedb orders_by_customer
0 3 * * 0 nodetool repair --incremental somedb order_events
```

---

#### Остатки для отображения (`product_stock_display`)

**Стратегия: ONE запись + Read Repair foreground**

```cql
-- Запись: скорость важнее строгости (источник правды - MongoDB)
CONSISTENCY ONE;
UPDATE product_stock_display SET quantity = ?, updated_at = ?
WHERE product_id = ? AND geo_zone = ?;

-- Чтение: foreground read repair выравнивает реплики при расхождении
CONSISTENCY LOCAL_QUORUM;
SELECT quantity FROM product_stock_display WHERE product_id = ? AND geo_zone = ?;
```

**Обоснование:**
- Cassandra хранит **копию** остатков для отображения, MongoDB - источник правды
- Расхождение в 10-30 секунд допустимо (Redis-кеш в любом случае добавляет задержку)
- Foreground Read Repair при QUORUM-чтении исправляет реплики в реальном времени
  без отдельного scheduled job

---

### Итоговая таблица

| Сущность | Consistency Write | Consistency Read | Hinted Handoff | Read Repair | Anti-Entropy Repair |
|---|---|---|---|---|---|
| `user_sessions` | ONE | LOCAL_ONE | Да (3 ч) | Фоновый 10% | Нет (TTL) |
| `cart_items` | ONE | LOCAL_ONE | Да (3 ч) | Фоновый 10% | Нет (TTL) |
| `orders_by_customer` | LOCAL_QUORUM | LOCAL_QUORUM | Да | Отключён | Еженедельно |
| `order_events` | LOCAL_QUORUM | LOCAL_QUORUM | Да | Отключён | Еженедельно |
| `product_stock_display` | ONE | LOCAL_QUORUM | Да (1 ч) | Foreground | Нет |

### Компромиссы

| Приоритет | Consistency | Механизм | Сущности |
|---|---|---|---|
| Скорость (latency) | ONE | Hinted Handoff + фоновый Read Repair | sessions, carts |
| Баланс | LOCAL_QUORUM | Hinted Handoff + Anti-Entropy Repair | orders, order_events |
| Скорость записи + надёжность чтения | ONE write / LOCAL_QUORUM read | Foreground Read Repair | product_stock_display |
