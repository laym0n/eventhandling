# AS IS

## Логическая модель event

| Поле          | Тип                                | Описание                                                                                |
| ------------- | ---------------------------------- | --------------------------------------------------------------------------------------- |
| id            | UUID, PK                           |                                                                                         |
| create_date   | TIMESTAMP WITH TIME ZONE, NOT NULL |                                                                                         |
| code          | VARCHAR(100), NOT NULL             | название события                                                                        |
| payload       | TEXT                               | payload для обработки                                                                   |
| group_id      | VARCHAR(100)                       | идентификатор группы в рамках, которого необходимо обрабатывать события последовательно |
| attempt_count | SMALLINT, NOT NULL                 | количество неудачных попыток обработки                                                  |
| max_attempts  | SMALLINT                           | максимальное количество неудачных попыток обработки                                     |

## Обработка

1. Поиск событий для обработки

```sql
SELECT *
FROM events
WHERE attempt_count < max_attempts
ORDER BY create_date
LIMIT :batch_size;
```

2. Сгруппировать события по `group_id` и каждую группу отсортировать по `create_date `
3. Для каждого события:
   1. Найти обработчик по `code`
   2. Передать на обработку событие
   3. Синхронно дождаться завершения обработки события
   4. Удалить событие по `id`
4. Если обработчик по `code` не найден
   1. установить `attempt_count` равным `max_attempts`
   2. Логирование, мониторинг
5. Если обработка события закончилась ошибкой
   1. увеличить `attempt_count` на 1
   2. Логирование, мониторинг (как финальных, так и нет попыток обработки)

## Очистка

1. Запрос для очистки старых записей

```sql
DELETE FROM events
WHERE attempt_count >= max_attempts and create_date <= :cleanup_before_date
```

## Общая схема

![Итоговая обработка](./out/result_processing/processing.svg)

# TO BE (Доработки приоритетов и дедлайна обработки)

## Концептуальная модель event

![Концептуальная модель](./out/events/events.svg)

## Логическая модель event

| Поле           | Тип                                | Описание                                                                                |
| -------------- | ---------------------------------- | --------------------------------------------------------------------------------------- |
| id             | UUID, PK                           |                                                                                         |
| create_date    | TIMESTAMP WITH TIME ZONE, NOT NULL |                                                                                         |
| code           | VARCHAR(100), NOT NULL             | название события                                                                        |
| payload        | TEXT                               | payload для обработки                                                                   |
| group_id       | VARCHAR(100)                       | идентификатор группы в рамках, которого необходимо обрабатывать события последовательно |
| attempt_count  | SMALLINT, NOT NULL                 | количество неудачных попыток обработки                                                  |
| max_attempts   | SMALLINT                           | максимальное количество неудачных попыток обработки                                     |
| deadline_until | TIMESTAMP WITH TIME ZONE           | Время дедлайна, когда событие становится просроченным                                   |
| locked_until   | TIMESTAMP WITH TIME ZONE           | Время блокировки события, после которого событие считается неуспешно обработанным       |
| prioritet      | SMALLINT                           | Приоритет события                                                                       |
| status         | VARCHAR(50)                        | Статусы события (WAIT, PROCESSING, ERROR)                                               |

## Конфигурация scheduler'ов

```yml
event-handler:
  clean-up:
    enabled:
    schedule-interval:
    lock-schedule-at-most-for:
    record-max-age:
  pulling:
    default: # дефолтный шедулер для обработки событий без приоритетов
      enabled:
      schedule-interval:
      lock-schedule-at-most-for:
      batch-size:
      event-lock-duration: # Длительность блокирования события для обработки; дефолтное значение + по кодам событий значения (опционально)
      pool:
        pool-size:
        max-pool-size:
        queue-capacity:
        keep-alive-seconds:
        execution-timeout:
    priority: # дефолтный шедулер для обработки событий c приоритетами
      enabled:
      schedule-interval:
      lock-schedule-at-most-for:
      batch-size:
      event-lock-duration: # Длительность блокирования события для обработки; дефолтное значение + по кодам событий значения (опционально)
      pool:
        pool-size:
        max-pool-size:
        queue-capacity:
        keep-alive-seconds:
        execution-timeout:
    events: # шедулеры для обработки отдельных событий
      [event_code]:
        enabled:
        schedule-interval:
        lock-schedule-at-most-for:
        batch-size:
        max-processing-count: # Максимальное количество одновременно обрабатываемых событий
        event-lock-duration: # Длительность блокирования события для обработки; дефолтное значение
        order-by: # PRIORITY | CREATE_DATE
        pool:
          pool-size:
          max-pool-size:
          queue-capacity:
          keep-alive-seconds:
          execution-timeout:
```

## Запрос поиска событий для обработки по конфигурации шедулера

### Базовый запрос

```sql
SELECT *
FROM events
WHERE status <> 'PROCESSING'
AND (deadline_until IS NULL OR now() <= deadline_until)
AND (max_attempts IS NULL OR attempt_count < max_attempts)
```

### Дефолтные шедулеры или для отдельного события

#### Для отдельного события

```sql
AND code = :code
```

#### Дефолтные шедулеры

```sql
AND code NOT IN (:codes) # codes - коды событий, для которых сформирован отдельный шедулер
```

### PRIORITY или CREATE_DATE шедулеры

#### PRIORITY шедулер

```sql
AND priority IS NOT NULL
ORDER BY priority, create_date
```

#### CREATE_DATE шедулер

```sql
AND priority IS NULL
ORDER BY create_date
```

### batch-size или max-processing-count шедулеры

#### batch-size шедулер

```sql
LIMIT :batch_size;
```

#### max-processing-count шедулер

```sql
LIMIT :max_processing_count - :processing_count;
```

## Общая логика обработки событий

![Общая логика обработки событий](./out/sheduler_event_processing/sheduler_event_processing.svg)

## Доработка запроса удаления старых событий в шедулере очистки

```sql
DELETE FROM events
WHERE (now() >= deadline_until OR attempt_count >= max_attempts) and create_date <= :cleanup_before_date
```

## Логика IDR обработчика

![Логика IDR обработчика](./out/process_idr/one_instance_processing_eager_cleanup.svg)

## Задачи

- Обновить ER: добавить (deadline_until, locked_until, prioritet, status) и убрать NOT NULL с max_attempts
- Обновить логику шедулера обработки событий
- Добавить конфигурацию шедулеров обработки событий
- Сконфигурировать два дефолтных шедулера обработки событий
- Обновить шедулер удаления старых записей
