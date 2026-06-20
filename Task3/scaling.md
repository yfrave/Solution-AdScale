# Масштабирование баз данных

## 1. Задача

Нужно описать, как масштабировать хранилища данных в AdScale после перехода к микросервисной архитектуре.

Основная цель — убрать общую PostgreSQL как узкое место и дать каждому сервису подходящую стратегию масштабирования.

Также нужно учитывать, что в целевой архитектуре появляется отдельный **Auth Service**. 
Он отвечает за пользователей, роли, JWT, refresh tokens и DSP credentials. 
При этом Gateway не должен синхронно обращаться в Auth Service на каждый bid request. 
Для быстрой проверки DSP используется локальный кэш Gateway.

---

## 2. Общий подход

Для AdScale используется несколько способов масштабирования:

* репликация PostgreSQL для рабочих данных;
* Redis Cluster для быстрых данных аукциона;
* локальный кэш DSP credentials на Gateway;
* Kafka partitions для событий;
* ClickHouse для аналитики;
* разделение чтения и записи там, где это полезно.

Не все сервисы нужно масштабировать одинаково. Выбор зависит от типа нагрузки и критичности данных.

---

## 3. Стратегия по сервисам

| Сервис                        | Хранилище                | Стратегия масштабирования                                                         |
| ----------------------------- | ------------------------ | --------------------------------------------------------------------------------- |
| Auth Service                  | PostgreSQL `auth_db`     | Master-Slave репликация, read replicas для чтения, кэш DSP credentials на Gateway |
| Bidding Service               | Redis                    | Redis Cluster с репликами                                                         |
| Campaign Service              | PostgreSQL `campaign_db` | Master-Slave репликация, read replicas для чтения                                 |
| Tracking / Statistics Service | Kafka + ClickHouse       | Партиционирование Kafka и ClickHouse по времени и campaign_id                     |
| Analytics Service             | ClickHouse               | Шардирование и репликация ClickHouse                                              |
| Billing Service               | PostgreSQL `billing_db`  | Master-Slave репликация, запись только в master                                   |

---

## 4. Auth Service

Auth Service хранит:

* пользователей;
* роли;
* права доступа;
* refresh tokens;
* DSP API keys;
* allowlist IP для DSP;
* настройки DSP-партнёров.

Для него используется:

```text
PostgreSQL master + read replicas
```

Как распределяется нагрузка:

* запись пользователей, ролей и DSP credentials идёт только в master;
* чтение можно направлять на read replicas;
* изменения прав доступа и DSP credentials публикуются в Kafka;
* Gateway обновляет локальный кэш credentials по событиям.

Основной поток обновления DSP credentials:

```text
Auth Service
  -> auth_db
  -> Kafka auth-events
  -> Auth Cache Updater
  -> Gateway credentials cache
```

Это нужно, чтобы Gateway мог проверять DSP-партнёра быстро и не вызывал Auth Service на каждый bid request.

Шардирование `auth_db` на первом этапе не требуется. Если данных станет много, можно рассмотреть шардирование по:

```text
tenant_id
advertiser_id
```

Для DSP credentials можно использовать ключ:

```text
dsp_id
```

Но на первом этапе достаточно PostgreSQL с репликацией и кэша на Gateway.

---

## 5. Bidding Service

Bidding Service находится в критическом пути bid request.

Он не должен обращаться к PostgreSQL во время расчёта ставки.

Для масштабирования используется:

```text
Redis Cluster
```

Что это даёт:

* быстрый доступ к кампаниям, ставкам и бюджетам;
* возможность распределить данные по нескольким узлам;
* реплики для отказоустойчивости;
* низкую задержку для Bidding Service.

Основной ключ шардирования Redis:

```text
campaign_id
```

Дополнительно можно использовать:

```text
advertiser_id
dsp_id
```

Bidding Service масштабируется горизонтально: можно запускать несколько экземпляров сервиса,
и все они будут читать данные из Redis.

---

## 6. Campaign Service

Campaign Service хранит кампании, ставки, таргетинг и статусы кампаний.

Для него подходит PostgreSQL с репликацией:

```text
PostgreSQL master + read replicas
```

Как распределяется нагрузка:

* запись идёт только в master;
* чтение можно направлять на read replicas;
* изменения кампаний публикуются в Kafka;
* Redis обновляется по событиям.

Это полезно, потому что кампании часто читаются, но изменяются реже, чем события показов и кликов.

Шардирование на первом этапе не требуется. Если данных станет слишком много, можно рассмотреть шардирование по:

```text
advertiser_id
```

Так кампании одного рекламодателя будут находиться в одном шарде.

---

## 7. Tracking / Statistics Service

Tracking / Statistics Service работает с большим потоком событий:

* bid events;
* показы;
* клики;
* конверсии.

Для записи событий используется Kafka.

Основные топики нужно партиционировать, чтобы распределить нагрузку между обработчиками.

Возможные ключи партиционирования:

| Событие             | Ключ партиционирования    |
| ------------------- | ------------------------- |
| auth events         | `user_id` или `dsp_id`    |
| bid events          | `request_id` или `dsp_id` |
| показы              | `campaign_id`             |
| клики               | `campaign_id`             |
| конверсии           | `campaign_id`             |
| изменения кампаний  | `campaign_id`             |
| изменения бюджета   | `advertiser_id`           |
| финансовые операции | `operation_id`            |

Для хранения подготовленной статистики используется ClickHouse.

В ClickHouse данные лучше партиционировать по дате:

```text
event_date
```

А шардировать по:

```text
campaign_id
```

Так отчёты по кампаниям будут читаться быстрее.

---

## 8. Analytics Service

Analytics Service строит отчёты по ClickHouse.

Для аналитики нужны:

* быстрые агрегирующие запросы;
* хранение большого объёма событий;
* чтение за периоды;
* отчёты по кампаниям и рекламодателям.

Для масштабирования ClickHouse используется:

* партиционирование по дате;
* шардирование по `campaign_id` или `advertiser_id`;
* репликация для отказоустойчивости.

Read replicas для PostgreSQL здесь не нужны, потому что Analytics Service не должен читать отчёты из рабочей PostgreSQL.

Основной источник данных для аналитики — ClickHouse.

---

## 9. Billing Service

Billing Service отвечает за деньги:

* балансы;
* пополнения;
* списания;
* счета;
* историю операций.

Для него используется PostgreSQL.

Стратегия масштабирования:

```text
PostgreSQL master + replica
```

Правила:

* все финансовые записи выполняются только в master;
* критичные проверки баланса читаются из master;
* read replica можно использовать только для не критичных отчётов и просмотра истории;
* финансовые операции должны быть идемпотентными.

Шардирование Billing Service на первом этапе лучше не делать, потому что финансовые данные требуют строгой согласованности.

Если нагрузка сильно вырастет, можно рассмотреть шардирование по:

```text
advertiser_id
```

Но это усложнит финансовые операции и сверку данных.

---

## 10. CQRS

CQRS можно применить там, где чтение и запись имеют разный профиль нагрузки.

В AdScale это подходит для:

* Auth Service;
* Campaign Service;
* Billing Service;
* Analytics Service.

### Auth Service

Запись пользователей, ролей и DSP credentials идёт в PostgreSQL.

Быстрое чтение DSP credentials для bid request идёт не из PostgreSQL, а из локального кэша Gateway.

```text
Command: Auth Service -> auth_db
Query: DSP Gateway -> Gateway credentials cache
```

Так Gateway не обращается в Auth Service на каждый bid request.

---

### Campaign Service

Запись кампаний идёт в PostgreSQL.

Чтение для аукциона идёт не из PostgreSQL, а из Redis.

```text
Command: Campaign Service -> campaign_db
Query: Bidding Service -> Redis
```

---

### Billing Service

Финансовые операции записываются в PostgreSQL.

Данные о доступных бюджетах для аукциона читаются из Redis.

```text
Command: Billing Service -> billing_db
Query: Bidding Service -> Redis
```

---

### Analytics Service

События записываются через Kafka и обработчики.

Отчёты читаются из ClickHouse.

```text
Command/Event: Kafka -> обработчики -> ClickHouse
Query: Analytics Service -> ClickHouse
```

---

## 11. Итоговая стратегия

| Хранилище                 | Стратегия                                                                                |
| ------------------------- | ---------------------------------------------------------------------------------------- |
| PostgreSQL `auth_db`      | Master-Slave, read replicas для чтения, события для обновления Gateway credentials cache |
| PostgreSQL `campaign_db`  | Master-Slave, read replicas для чтения                                                   |
| PostgreSQL `billing_db`   | Master-Slave, запись и критичные чтения через master                                     |
| Gateway credentials cache | Локальный кэш DSP credentials, обновление по `auth-events`                               |
| Redis                     | Redis Cluster с репликами                                                                |
| Kafka                     | Партиционирование топиков по ключам событий                                              |
| ClickHouse                | Партиционирование по дате, шардирование по campaign_id или advertiser_id                 |
