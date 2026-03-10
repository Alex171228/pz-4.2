# Практическое задание 4
## Шишков А.Д. ЭФМО-02-22
## Тема
Метрики приложения, Prometheus, Grafana и интеграция с HTTP сервисом.

## Цель
Научиться собирать и визуализировать метрики сервиса: трафик, ошибки, задержки, активные запросы.

---

## 1. Описание метрик

В сервис Tasks добавлены 3 метрики через `github.com/prometheus/client_golang`:

### http_requests_total (Counter)

Общее количество HTTP-запросов. Увеличивается на 1 при каждом завершённом запросе.

| Label | Описание | Примеры значений |
|-------|----------|-----------------|
| `method` | HTTP-метод | GET, POST, PATCH, DELETE |
| `route` | Нормализованный маршрут | `/v1/tasks`, `/v1/tasks/:id` |
| `status` | Код ответа | 200, 201, 401, 404, 503 |

### http_request_duration_seconds (Histogram)

Длительность обработки запроса в секундах. Buckets: 0.01, 0.05, 0.1, 0.3, 1, 3.

| Label | Описание | Примеры значений |
|-------|----------|-----------------|
| `method` | HTTP-метод | GET, POST, PATCH, DELETE |
| `route` | Нормализованный маршрут | `/v1/tasks`, `/v1/tasks/:id` |

### http_in_flight_requests (Gauge)

Количество запросов, обрабатываемых в данный момент. Увеличивается при входе запроса, уменьшается при завершении.

Labels отсутствуют — одно глобальное значение для всего сервиса.

### Нормализация route

Для предотвращения взрыва кардинальности путь нормализуется:
- `/v1/tasks` → `/v1/tasks`
- `/v1/tasks/abc123` → `/v1/tasks/:id`

---

## 2. Пример вывода /metrics

Команда:

```bash
curl -s http://<SERVER_IP>:8082/metrics | head -30
```

<img width="947" height="495" alt="image" src="https://github.com/user-attachments/assets/36c00141-cd0f-40ce-bae6-8c0573eb54fb" /> 


---

## 3. Docker Compose и Prometheus

### docker-compose.yml

Файл `deploy/monitoring/docker-compose.yml` поднимает 3 контейнера:

| Сервис | Образ | Порт | Назначение |
|--------|-------|------|------------|
| `tasks` | сборка из Dockerfile | 8082 | Микросервис задач с endpoint `/metrics` |
| `prometheus` | `prom/prometheus:latest` | 9090 | Сбор метрик каждые 5 секунд |
| `grafana` | `grafana/grafana:latest` | 3000 | Визуализация (admin/admin) |

Все контейнеры объединены в сеть `monitoring`. Grafana автоматически провизионирует Prometheus как datasource и подключает готовый dashboard.

### prometheus.yml

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "tasks"
    static_configs:
      - targets: ["tasks:8082"]
```

Target — контейнер `tasks` на порту 8082. Prometheus обращается к `http://tasks:8082/metrics` каждые 5 секунд.

---

## 4. Графики в Grafana

Dashboard автоматически провизионируется при запуске. Содержит 3 панели:

### 4.1. RPS (Requests Per Second)

PromQL:

```promql
sum(rate(http_requests_total[1m])) by (route)
```

Показывает количество запросов в секунду с разбивкой по маршрутам.

<img width="488" height="475" alt="image" src="https://github.com/user-attachments/assets/c52bf110-07bc-4d17-b9b1-7fb36a4facff" /> 


### 4.2. Error Rate (4xx / 5xx)

PromQL:

```promql
sum(rate(http_requests_total{status=~"4..|5.."}[1m])) by (status)
```

Показывает частоту ошибочных ответов с разбивкой по кодам (401, 404, 503...).

<img width="477" height="461" alt="image" src="https://github.com/user-attachments/assets/961c1d89-fcdf-4a09-ad42-e9d14a562f62" /> 


### 4.3. Latency p95

PromQL:

```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m])) by (le, route))
```

Показывает 95-й перцентиль задержки — время, в которое укладываются 95% запросов.

<img width="480" height="464" alt="image" src="https://github.com/user-attachments/assets/096d8d40-b0d6-4b0f-a6f4-7678946e49d3" /> 


---

## 5. Инструкция запуска

### Предварительные требования

- Docker и Docker Compose установлены на сервере
- Auth-сервис запущен на порту 8081 (на хост-машине)

### Запуск Auth-сервиса (на хосте)

```bash
cd ~/pz4
export AUTH_PORT=8081
export AUTH_GRPC_PORT=50051
go run ./services/auth/cmd/auth &
```

### Запуск Tasks + Prometheus + Grafana

```bash
cd ~/pz4/deploy/monitoring
docker compose up -d --build
```

### Генерация нагрузки (для появления данных на графиках)

Успешные запросы:

```bash
for i in $(seq 1 50); do
  curl -s http://localhost:8082/v1/tasks \
    -H "Authorization: Bearer demo-token" > /dev/null
done
```

Ошибочные запросы (401):

```bash
for i in $(seq 1 20); do
  curl -s http://localhost:8082/v1/tasks \
    -H "Authorization: Bearer wrong" > /dev/null
done
```

### Проверка

| Что | URL |
|-----|-----|
| Метрики Tasks | http://localhost:8082/metrics |
| Prometheus targets | http://localhost:9090/targets |
| Grafana dashboard | http://localhost:3000 (admin / admin) |

---

## 6. Контрольные вопросы

**1. Зачем приложение отдаёт метрики на отдельном пути?**

Endpoint `/metrics` отделён от бизнес-логики, чтобы Prometheus мог собирать метрики без авторизации и не создавая нагрузку на основные хендлеры. Это также позволяет закрыть `/metrics` на уровне сети (firewall), не затрагивая API.

**2. Чем Counter отличается от Gauge?**

Counter — монотонно растущий счётчик (только увеличивается). Gauge — значение, которое может расти и падать (например, текущее число активных запросов). Counter подходит для подсчёта событий, Gauge — для текущего состояния.

**3. Почему latency лучше мерить histogram, а не средним?**

Среднее скрывает выбросы: если 99% запросов за 10ms, а 1% за 10s, среднее будет ~110ms — выглядит нормально, но 1% пользователей ждут 10 секунд. Histogram позволяет вычислять перцентили (p95, p99), которые показывают реальную картину.

**4. Что такое labels и почему их не должно быть слишком много?**

Labels — ключ-значение пары, по которым можно фильтровать и группировать метрики. Каждая уникальная комбинация labels создаёт отдельный time series в Prometheus. Слишком много labels (например, user_id) приводит к взрыву кардинальности — миллионы time series, что перегружает память и хранилище.

**5. Что значат p95/p99 и чем они полезны?**

p95 — значение, ниже которого находятся 95% наблюдений. p99 — 99%. Если p95 latency = 200ms, значит 95% запросов обрабатываются быстрее 200ms. Это стандартная метрика для SLA/SLO: «99% запросов должны обрабатываться быстрее 500ms».
