# Hotelio - переход от монолита к микросервисной архитектуре

Проект эволюции платформы бронирования отелей "Hotelio" из Spring-монолита в
микросервисную архитектуру: поэтапный вынос доменных сервисов по паттерну Strangler Fig,
GraphQL BFF, событийная интеграция, CI/CD, Kubernetes и service mesh.
Все решения оформлены как ADR, диаграммы подготовлены в нотации C4.

## Контекст

Текущее состояние системы и цели миграции описаны в [architecture/context.md](architecture/context.md).
Устройство монолита, модули и REST API - в [architecture/readme.md](architecture/readme.md),
схема зависимостей компонент - в [architecture/monolith.png](architecture/monolith.png).

## Результаты по этапам

### 1. Целевая архитектура и вынос первого сервиса

Проработка целевой архитектуры Hotelio и план первого шага миграции - выноса
read-ориентированного сервиса Hotels с минимальной связанностью.

| ADR | Тема | Артефакт |
|-----|------|----------|
| ADR-000 | Проработка целевой архитектуры Hotelio | [ADR-000.md](tasks/task1/results/ADR-000.md) |
| ADR-001 | Вынос микросервиса Hotels (первый шаг миграции) | [ADR-001.md](tasks/task1/results/ADR-001.md) |

| Диаграмма | PlantUML | PNG |
|-----------|----------|-----|
| Диаграмма контейнеров (ADR-000) | [ADR-000-container.puml](tasks/task1/results/ADR-000-container.puml) | [ADR-000-container.png](tasks/task1/results/ADR-000-container.png) |
| Диаграмма развёртывания (ADR-000) | [ADR-000-deployment.puml](tasks/task1/results/ADR-000-deployment.puml) | [ADR-000-deployment.png](tasks/task1/results/ADR-000-deployment.png) |
| Диаграмма контейнеров, основной сценарий (ADR-001) | [ADR-001-container-main.puml](tasks/task1/results/ADR-001-container-main.puml) | [ADR-001-container-main.png](tasks/task1/results/ADR-001-container-main.png) |
| Диаграмма контейнеров, альтернатива (ADR-001) | [ADR-001-container-alternative.puml](tasks/task1/results/ADR-001-container-alternative.puml) | [ADR-001-container-alternative.png](tasks/task1/results/ADR-001-container-alternative.png) |
| Диаграмма переключения трафика (ADR-001) | [ADR-001-migration.puml](tasks/task1/results/ADR-001-migration.puml) | [ADR-001-migration.png](tasks/task1/results/ADR-001-migration.png) |

### 2. Вынос сервиса бронирований и миграция данных

Выделение Booking Service с собственным REST- и gRPC-контрактом, отдельной БД
и поэтапной миграцией данных без остановки пользовательского сценария.

| Артефакт | Назначение | Ссылка |
|----------|-----------|--------|
| `booking.proto` | Контракт gRPC booking-сервиса | [booking.proto](tasks/task2/booking.proto) |
| `docker-compose.yml` | Запуск монолита и микросервиса | [docker-compose.yml](tasks/task2/docker-compose.yml) |
| Стратегия миграции данных | Пошаговый план переноса бронирований | [results/README.md](tasks/task2/results/README.md) |
| Фикстуры БД | Тестовые данные монолита и сервиса | [init-fixtures.sql](tasks/task2/results/init-fixtures.sql), [ms-init-fixtures.sql](tasks/task2/results/ms-init-fixtures.sql) |
| `regress.sh` | Регрессионные тесты REST API | [regress.sh](tasks/task2/results/regress.sh) |
| Логи проверок | REST/gRPC-вызовы, выборки, тест-лог | [rest-and-grpc-calls.txt](tasks/task2/results/rest-and-grpc-calls.txt), [test-log.txt](tasks/task2/results/test-log.txt) |

### 3. GraphQL Federation и BFF с ACL

API Gateway и GraphQL BFF: агрегация данных нескольких сервисов, получение только
нужных полей и проверка прав доступа (ACL) по пользователю.

| Артефакт | Назначение | Ссылка |
|----------|-----------|--------|
| Gateway | Роутинг и проброс заголовков в подграфы | [gateway/index.js](tasks/task3/gateway/index.js) |
| Hotel subgraph | Получение данных об отелях из монолита | [hotel-subgraph/index.js](tasks/task3/hotel-subgraph/index.js) |
| Booking subgraph | Получение бронирований из сервиса (gRPC) с ACL | [booking-subgraph/index.js](tasks/task3/booking-subgraph/index.js) |
| Контракт booking | Proto-файл booking-сервиса | [booking.proto](tasks/task3/booking-subgraph/protos/booking.proto) |
| docker-compose | Общая сеть и запуск всех сервисов | [docker-compose.yml](tasks/task3/docker-compose.yml) |
| Отчёт | Описание внесённых изменений | [results/report.md](tasks/task3/results/report.md) |

| Результат | PNG |
|-----------|-----|
| Успешный запрос (ACL разрешает) | [request-success.png](tasks/task3/results/request-success.png) |
| Отказ по ACL | [request-deny.png](tasks/task3/results/request-deny.png) |

### 4. Сборка, Helm и CI/CD, Service Discovery

Docker-образ сервиса с health-эндпоинтами, Helm-чарт с пробами и конфигурацией
по средам, пайплайн CI/CD и проверка DNS service discovery внутри кластера.

| Артефакт | Назначение | Ссылка |
|----------|-----------|--------|
| Dockerfile | Сборка образа booking-service (`/ping`) | [booking-service/Dockerfile](tasks/task4/booking-service/Dockerfile) |
| Helm-чарт | Deployment, Service, пробы | [helm/booking-service](tasks/task4/helm/booking-service) |
| Значения окружений | staging и prod | [values-staging.yaml](tasks/task4/helm/booking-service/values/values-staging.yaml), [values-prod.yaml](tasks/task4/helm/booking-service/values/values-prod.yaml) |
| CI/CD пайплайн | build, test, deploy, tag | [.gitlab-ci.yml](tasks/task4/.gitlab-ci.yml) |
| Проверки | DNS внутри кластера и статус деплоя | [check-dns.sh](tasks/task4/check-dns.sh), [check-status.sh](tasks/task4/check-status.sh) |
| Отчёт | Описание внесённых изменений | [results/report.md](tasks/task4/results/report.md) |

| Результат | PNG |
|-----------|-----|
| Проверка DNS в кластере | [check-dns.png](tasks/task4/results/check-dns.png) |
| Статус деплоя и сервисов | [check-status.png](tasks/task4/results/check-status.png) |
| Проверка через curl | [curl.png](tasks/task4/results/curl.png) |

### 5. Istio: canary, feature-флаги и fallback

Развёртывание двух версий сервиса (v1/v2), управление трафиком через Istio
(канареечный релиз, маршрутизация по заголовку), фича-флаги и откат.

| Артефакт | Назначение | Ссылка |
|----------|-----------|--------|
| Istio-манифесты | Gateway, VirtualService, DestinationRule, EnvoyFilter | [istio](tasks/task5/istio) |
| Значения версий | v1 и v2 | [values-v1.yaml](tasks/task5/helm/booking-service/values/values-v1.yaml), [values-v2.yaml](tasks/task5/helm/booking-service/values/values-v2.yaml) |
| Проверки | canary, feature-флаг, fallback, istio | [check-canary.sh](tasks/task5/check-canary.sh), [check-feature-flag.sh](tasks/task5/check-feature-flag.sh), [check-fallback.sh](tasks/task5/check-fallback.sh), [check-istio.sh](tasks/task5/check-istio.sh) |
| Отчёт | Описание внесённых изменений | [results/report.md](tasks/task5/results/report.md) |

## Как читать артефакты

Каждая задача содержит каталоги с исходниками и каталог `results/` с
результатами и отчётами. ADR содержат контекст, требования, решение,
рассмотренные альтернативы и компромиссы. Диаграммы представлены исходником
PlantUML (`.puml`) и готовым рендером (`.png`), а результаты проверок
зафиксированы в виде логов (`.txt`) и скриншотов (`.png`).
