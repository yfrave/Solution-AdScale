# Схема взаимодействия и выбор протоколов

## 1. Задача

Нужно определить, как новая DSP-площадка будет взаимодействовать с AdScale через Bidding Service.

Основной сценарий — обработка bid request. Это критичный путь, поэтому он должен быть быстрым 
и не должен зависеть от медленных синхронных операций с базой данных.

Также нужно описать, какие протоколы используются между сервисами AdScale:

* для внешней интеграции с DSP;
* для внутреннего вызова Bidding Service;
* для обновления read-моделей;
* для событий;
* для управления кампаниями, бюджетами и доступами.

---

## 2. Основной поток bid request

```text
DSP
  -> DSP Gateway
  -> Bidding Service
  -> Redis
  -> Bidding Service
  -> DSP Gateway
  -> DSP
```

Описание потока:

1. DSP отправляет bid request.
2. DSP Gateway проверяет запрос и применяет ограничения.
3. Gateway передаёт подготовленный внутренний запрос в Bidding Service.
4. Bidding Service читает быстрые данные из Redis.
5. Bidding Service рассчитывает ставку.
6. Ответ возвращается обратно в DSP.

В этом сценарии Bidding Service не обращается к PostgreSQL.

Данные для проверки DSP-партнёра подготавливаются отдельно через Auth Service и используются на уровне Gateway.

---

## 3. Внешний протокол для DSP

Для интеграции с DSP используется:

```text
HTTPS / OpenRTB JSON
```

Почему так:

* DSP-площадки обычно интегрируются через HTTP API;
* OpenRTB подходит для bid request и bid response;
* JSON проще поддерживать для внешней интеграции;
* HTTPS обеспечивает защищённую передачу данных.

Пример внешнего маршрута:

```text
POST /openrtb/v1/bid
```

Внешний запрос не должен напрямую попадать в Bidding Service. Сначала он проходит через DSP Gateway.

---

## 4. DSP Gateway

Для интеграции с новой DSP нужен отдельный входной слой — **DSP Gateway**.

Он нужен для:

* маршрутизации bid request в Bidding Service;
* проверки DSP-партнёра;
* ограничения частоты запросов;
* контроля таймаутов;
* защиты Bidding Service от лишней нагрузки;
* сбора метрик по времени ответа;
* применения Circuit Breaker;
* преобразования OpenRTB-запроса во внутренний формат.

DSP Gateway скрывает внутреннюю архитектуру AdScale от внешней DSP.

После проверки Gateway передаёт в Bidding Service уже подготовленный внутренний `BidRequest`, 
где есть `request_id`, `dsp_id`, placement-данные, параметры пользователя и контекст показа.

---

## 5. Взаимодействие Gateway и Auth Service

Auth Service отвечает за пользователей, роли, JWT, refresh tokens и DSP credentials.

Для административных и пользовательских сценариев Gateway обращается в Auth Service по REST:

```text
Advertiser Dashboard / Admin
  -> Gateway
  -> Auth Service
```

Примеры таких сценариев:

* login;
* refresh token;
* управление пользователями;
* управление ролями;
* создание или ротация DSP API keys;
* блокировка DSP-партнёра.

Для RTB-нагрузки данные доступа обновляются асинхронно:

```text
Auth Service
  -> Kafka
  -> Auth Cache Updater
  -> Gateway credentials cache
```

Так DSP Gateway получает актуальную read-модель для быстрых проверок внешних bid request.

---

## 6. Внутреннее взаимодействие Gateway и Bidding Service

Для связи между DSP Gateway и Bidding Service предлагается использовать:

```text
gRPC
```

Почему gRPC:

* это критичный по времени путь;
* gRPC обычно быстрее и компактнее REST;
* есть строгий контракт запроса и ответа;
* проще контролировать таймауты;
* удобно передавать структурированные данные bid request.

Пример внутреннего метода:

```text
CalculateBid(BidRequest) returns (BidResponse)
```

---

## 7. Взаимодействие Bidding Service и Redis

Bidding Service читает из Redis данные, которые нужны для расчёта ставки:

* активные кампании;
* ставки;
* таргетинг;
* доступные бюджеты;
* лимиты показов;
* статусы кампаний.

Протокол:

```text
Redis protocol
```

Redis используется потому, что Bidding Service не должен ходить в PostgreSQL во время обработки bid request.

---

## 8. Взаимодействие с Campaign Service

Bidding Service не должен синхронно вызывать Campaign Service на каждый bid request.

Правильный поток обновления кампаний:

```text
Campaign Service
  -> Kafka
  -> Cache Updater
  -> Redis
  -> Bidding Service
```

Почему так:

* Campaign Service отвечает за создание и изменение кампаний;
* Bidding Service отвечает только за быстрый расчёт ставки;
* синхронный вызов Campaign Service замедлил бы bid request;
* Redis хранит уже готовую модель данных для быстрого чтения.

Для обычного управления кампаниями используется REST API через Gateway, потому что это не критичный по задержке административный сценарий.

---

## 9. Взаимодействие с Billing Service

Billing Service отвечает за бюджеты, списания и финансовые операции.

В критическом пути Bidding Service использует только данные бюджета из Redis.

Обновление бюджетов происходит через события:

```text
Billing Service
  -> Kafka
  -> Cache Updater
  -> Redis
```

Финансовые операции должны быть идемпотентными. Для этого у операции должен быть уникальный идентификатор, например `operation_id`.

Для пополнения бюджета и просмотра финансовой истории используется REST API через Gateway.

---

## 10. События показов, кликов и ставок

Для событий используется Kafka.

В Kafka отправляются:

* показы;
* клики;
* конверсии;
* результат обработки bid request;
* изменения кампаний;
* изменения бюджетов;
* финансовые события;
* события изменения доступов.

Пример потока статистики:

```text
Bidding Service / Tracking Service
  -> Kafka
  -> Statistics Worker
  -> ClickHouse
```

Почему Kafka:

* события не должны замедлять ответ DSP;
* поток показов и кликов может быть большим;
* события можно обрабатывать асинхронно;
* Kafka позволяет сглаживать пики нагрузки;
* при временной проблеме обработчика события не теряются сразу.

---

## 11. Взаимодействие с Analytics Service

Analytics Service отвечает за отчёты.

Он не должен читать данные напрямую из рабочих PostgreSQL-баз Campaign Service, Billing Service или Auth Service.

Основной поток:

```text
Kafka
  -> Statistics Worker / Analytics Loader
  -> ClickHouse
  -> Analytics Service
```

Для чтения отчётов используется:

```text
SQL
```

Для обращения личного кабинета к Analytics Service используется REST API через Gateway.

---

## 12. Административные сценарии

Административные и пользовательские сценарии не находятся в критическом пути bid request.

К ним относятся:

* создание кампании;
* изменение ставки;
* изменение таргетинга;
* пополнение бюджета;
* просмотр отчётов;
* login;
* refresh token;
* управление пользователями;
* управление DSP-партнёрами.

Для этих сценариев подходит REST API:

```text
Dashboard
  -> Gateway
  -> нужный сервис
```

REST здесь допустим, потому что требования по задержке ниже, чем в RTB-пути.

---

## 13. Итоговая таблица протоколов

| Взаимодействие                                  | Протокол             | Почему                                                                  |
| ----------------------------------------------- | -------------------- | ----------------------------------------------------------------------- |
| DSP -> DSP Gateway                              | HTTPS / OpenRTB JSON | Стандартный внешний формат для RTB-интеграции                           |
| DSP Gateway -> Bidding Service                  | gRPC                 | Быстрый внутренний вызов в критическом пути                             |
| DSP Gateway -> Gateway credentials cache        | local cache lookup   | Быстрая проверка DSP credentials без сетевого вызова в критическом пути |
| Bidding Service -> Redis                        | Redis protocol       | Быстрое чтение данных для аукциона                                      |
| Gateway -> Auth Service                         | REST                 | Login, refresh token, управление пользователями и DSP credentials       |
| Auth Service -> Kafka                           | Kafka                | Асинхронное обновление данных доступа                                   |
| Auth Cache Updater -> Gateway credentials cache | cache update         | Обновление read-модели для Gateway                                      |
| Campaign Service -> Kafka                       | Kafka                | Асинхронное обновление данных кампаний                                  |
| Billing Service -> Kafka                        | Kafka                | Асинхронное обновление бюджетов                                         |
| Bidding / Tracking -> Kafka                     | Kafka                | События показов, кликов и ставок не тормозят ответ DSP                  |
| Statistics Worker -> ClickHouse                 | SQL / batch insert   | Запись подготовленных событий для аналитики                             |
| Analytics Service -> ClickHouse                 | SQL                  | Быстрое чтение данных для отчётов                                       |
| Dashboard -> Gateway                            | HTTPS / REST         | Внешний доступ к личному кабинету                                       |
| Gateway -> Campaign Service                     | REST                 | Управление кампаниями                                                   |
| Gateway -> Billing Service                      | REST                 | Управление балансом и финансовыми операциями                            |
| Gateway -> Analytics Service                    | REST                 | Получение отчётов                                                       |
