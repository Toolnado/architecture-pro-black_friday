# Задание 8. Выявление и устранение «горячих» шардов

## Описание проблемы

Коллекция `products` шардирована по ключу `{ category: 1, _id: 1 }`.  
70% запросов приходится на категорию «Электроника», все эти запросы
направляются на один и тот же шард, **query hotspot**.

Важно различать два вида нагрузки:

| Вид | Причина | Симптом |
|---|---|---|
| **Data hotspot** | Чанки одной категории не влезают на один шард | Неравномерный объём данных |
| **Query hotspot** | Все запросы по «Электронике» идут на один шард | Перегрузка CPU/IO при равномерных данных |

В нашем случае - **query hotspot**: даже если балансировщик равномерно
распределил чанки, запросы `{ category: "electronics" }` всегда попадают
на те же шарды, где лежат эти чанки.

---

## 1. Метрики мониторинга

### 1.1 Распределение данных и чанков

```js
// Общий статус шардирования: число чанков на каждом шарде
sh.status()

// Детальное распределение данных коллекции по шардам
db.products.getShardDistribution()
```

Проверить дисбаланс можно с помощью команды `getShardDistribution()`.

**Пороговое значение:** разница в объёме данных между шардами > 20% - сигнал к действию.

### 1.2 Операционная нагрузка на каждый шард

```js
// Выполнить на каждом шарде отдельно - счётчики операций
db.adminCommand({ serverStatus: 1 }).opcounters

// Очередь операций (признак перегрузки - currentQueue > 0)
db.adminCommand({ serverStatus: 1 }).globalLock

// Активные операции прямо сейчас
db.adminCommand({ currentOp: 1, active: true })
```

**Ключевые метрики из `serverStatus`:**

| Метрика | Путь | Порог тревоги |
|---|---|---|
| Запросов в секунду | `opcounters.query` | Разница между шардами > 3× |
| Записей в секунду | `opcounters.update` | Разница между шардами > 3× |
| Очередь чтения | `globalLock.currentQueue.readers` | > 10 |
| Очередь записи | `globalLock.currentQueue.writers` | > 5 |
| Connections | `connections.current` | > 80% от `available` |

### 1.3 Jumbo-чанки

Jumbo-чанк - чанк, который не может быть разделён, потому что все
документы в нём имеют одинаковое значение шард-ключа (например, тысячи
товаров с `category: "electronics"` и разными `_id` - это не jumbo,
но category-only ключ создал бы такую проблему).

```js
// Найти jumbo-чанки в коллекции
db.getSiblingDB("config").chunks.find({
  ns: "somedb.products",
  jumbo: true
})
```

### 1.4 Метрики инфраструктуры (вне MongoDB)

Собирать через **Prometheus + MongoDB Exporter** или **Ops Manager**:

| Метрика | Значение |
|---|---|
| `mongodb_mongod_op_latencies_latency` | Задержка операций на каждом шарде |
| `mongodb_mongod_wiredtiger_cache_bytes_currently_in_cache` | Давление на кеш WiredTiger |
| CPU utilization per node | Неравномерная загрузка процессора |
| Disk IOPS per node | Перегрузка дискового ввода-вывода |
| Network bytes in/out per node | Асимметрия сетевого трафика |

---

## 2. Механизмы устранения дисбаланса

### 2.1 Проверка и настройка балансировщика

Балансировщик MongoDB перемещает чанки между шардами автоматически,
но только по числу чанков - он не знает о нагрузке запросов.

```js
// Проверить состояние балансировщика
sh.getBalancerState()

// Включить балансировщик
sh.startBalancer()

// Задать окно работы балансировщика (ночные часы, меньше влияния на прод)
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  {
    $set: {
      activeWindow: {
        start: "02:00",
        stop: "06:00"
      }
    }
  },
  { upsert: true }
)

// Вручную разделить горячий чанк
sh.splitAt("somedb.products", { category: "electronics", _id: ObjectId("...") })

// Вручную переместить чанк на другой шард
sh.moveChunk("somedb.products", { category: "electronics", _id: ObjectId("...") }, "shard2")
```

### 2.2 Zone Sharding - выделить «Электронику» на отдельные шарды

Добавить дополнительные шарды, которые будут обслуживать только
категорию «Электроника». Балансировщик сам перенесёт чанки в зону.

```js
// Добавить новые шарды в кластер
sh.addShard("shard3/shard3:27018")
sh.addShard("shard4/shard4:27018")

// Создать зону для «Электроники»
sh.addShardToZone("shard3", "electronics-zone")
sh.addShardToZone("shard4", "electronics-zone")

// Назначить диапазон «Электроники» на эту зону
sh.updateZoneKeyRange(
  "somedb.products",
  { category: "electronics", _id: MinKey },
  { category: "electronics", _id: MaxKey },
  "electronics-zone"
)

// Остальные категории - на основные шарды
sh.addShardToZone("shard1", "general-zone")
sh.addShardToZone("shard2", "general-zone")

sh.updateZoneKeyRange(
  "somedb.products",
  { category: MinKey, _id: MinKey },
  { category: "electronics", _id: MinKey },
  "general-zone"
)

sh.updateZoneKeyRange(
  "somedb.products",
  { category: "electronics", _id: MaxKey },
  { category: MaxKey, _id: MaxKey },
  "general-zone"
)
```

**Результат:** горячая категория изолирована на выделенных шардах
с бо́льшими ресурсами. Другие категории не страдают.

### 2.3 Решардинг коллекции (MongoDB 5.0+)

Если изоляция зонами недостаточна - полностью сменить шард-ключ.
Переход на хешированный `_id` даёт равномерное распределение запросов,
но теряется преимущество targeted-запросов по категории.

```js
// Решардирование - онлайн, без простоя (MongoDB 5.0+)
db.adminCommand({
  reshardCollection: "somedb.products",
  key: { _id: "hashed" }
})

// Отслеживать прогресс
db.getSiblingDB("admin").aggregate([
  { $currentOp: { allUsers: true, localOps: false } },
  { $match: { type: "op", "originatingCommand.reshardCollection": { $exists: true } } }
])
```

После решардинга поиск по категории становится scatter-gather -
компенсировать через кеширование.

### 2.4 Кеширование горячих данных в Redis

Поскольку в стеке уже есть Redis, правильная стратегия - снять нагрузку
с MongoDB для операций чтения по популярным категориям.

**Стратегия cache-aside для листинга категории:**

```
1. GET redis:products:category:electronics:page:1
2. Если hit  → вернуть из Redis
3. Если miss → запросить MongoDB → записать в Redis с TTL → вернуть
```

**Рекомендуемые TTL:**

| Данные | TTL |
|---|---|
| Листинг категории (страница товаров) | 5 минут |
| Карточка товара (описание, атрибуты) | 30 минут |
| Остатки (`stock`) | Не кешировать - критично актуальное значение |

---

## 3. Итоговая стратегия

```
Шаг 1. Диагностика
  └─ sh.status() + getShardDistribution() + serverStatus на каждом шарде
  └─ Найти шард с opcounters.query в 3× и более выше среднего

Шаг 2. Быстрое облегчение (без даунтайма)
  └─ Zone sharding: вынести "electronics" на выделенные шарды
  └─ Включить Redis-кеш для листинга категорий

Шаг 3. Долгосрочное решение (если зоны не справляются)
  └─ reshardCollection на { _id: "hashed" }
  └─ Полностью полагаться на Redis для category-запросов

Шаг 4. Мониторинг
  └─ Prometheus + MongoDB Exporter
  └─ Алерт: разница opcounters.query между шардами > 3×
  └─ Алерт: globalLock.currentQueue.readers > 10
  └─ Еженедельная проверка getShardDistribution()
```

---

## Сводная таблица метрик и порогов

| Метрика | Команда / источник | Порог тревоги | Действие |
|---|---|---|---|
| Разброс числа чанков | `sh.status()` | > 2× между шардами | Проверить балансировщик |
| Разброс объёма данных | `getShardDistribution()` | > 20% | Запустить балансировщик вручную |
| Разброс запросов/сек | `serverStatus.opcounters` | > 3× | Zone sharding или решардинг |
| Очередь чтения | `serverStatus.globalLock` | `readers > 10` | Масштабировать реплику, Redis |
| Jumbo-чанки | `config.chunks` | Любой > 0 | `sh.splitAt()` |
| CPU шарда | Prometheus / Ops Manager | > 80% sustained | Добавить шард в зону |
