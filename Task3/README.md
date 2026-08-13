# Task3 — проверка HPA на локальном Minikube

## Что настроено

- `deployment.yaml` запускает `ghcr.io/yandex-practicum/scaletestapp:latest`, стартует с одной реплики и задаёт лимит памяти `30Mi`.
- `service.yaml` открывает приложение внутри кластера на порту `8080`.
- `hpa.yaml` масштабирует deployment по памяти: целевая утилизация `80%`, диапазон от 1 до 10 реплик.
- `locustfile.py` содержит сценарий из задания: пользователь вызывает `GET /`.

`resources.requests.memory` выставлен в `8Mi`, а лимит оставлен `30Mi`. HPA считает процент утилизации от request, поэтому такой request позволяет увидеть рост раньше, чем контейнер упрётся в лимит памяти.

## Команды запуска

```bash
minikube start --cpus=4 --memory=6144
minikube addons enable metrics-server
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
kubectl get pods
kubectl top pods
kubectl get hpa -w
```

Для локального доступа к приложению использовался tunnel Minikube:

```bash
minikube service scaletestapp --url
```

В этом запуске команда выдала `http://127.0.0.1:60840`. При повторной проверке порт может быть другим.

Locust запускался из корня репозитория:

```bash
.venv/bin/locust -f Task3/locustfile.py \
  --headless \
  --host=http://127.0.0.1:60840 \
  --users 200 \
  --spawn-rate 40 \
  --run-time 1m \
  --csv Task3/evidence/locust-service \
  --html Task3/evidence/locust-service-report.html
```

Для проверки масштабирования также снимался watch HPA во время более тяжёлой нагрузки:

```bash
kubectl get hpa scaletestapp
kubectl get deployment scaletestapp -o wide
kubectl get pods -l app=scaletestapp -o wide
kubectl top pods -l app=scaletestapp
kubectl describe hpa scaletestapp
```

## Доказательства

Фактические артефакты лежат в `Task3/evidence/`:

- `before-load.log` — состояние перед нагрузкой: одна реплика.
- `hpa-watch-balanced.log` — динамика под нагрузкой: HPA увидел `memory: 367%/80%` и начал поднимать deployment с 1 до 2 и 4 pod-ов.
- `after-load.log`, `hpa-describe.log`, `events-after-load.log` — итоговое состояние после повторного прогона: deployment дошёл до 10 pod-ов, есть события `SuccessfulRescale`.
- `kubernetes-dashboard-scaletestapp-full.png` — реальный скриншот Minikube Dashboard: deployment `scaletestapp`, `Pods status: 10 / 10`.
- `locust-service-report.html` и `locust-service-report.png` — отчёт Locust: около 4 тысяч запросов, ошибок 0.
- `locust-service_*.csv` — CSV-выгрузка Locust.

## Проверка метрик приложения

```bash
curl http://localhost:8080/
curl http://localhost:8080/metrics | grep http_requests_total
```

## Замечание по локальной машине

Образ `ghcr.io/yandex-practicum/scaletestapp:latest` не содержит `linux/arm64` manifest. На Apple Silicon образ был заранее скачан как `linux/amd64` и загружен в Minikube, а в deployment указан `imagePullPolicy: IfNotPresent`. Также отключена инъекция Istio sidecar через `sidecar.istio.io/inject: "false"`, потому что в локальном default namespace уже был включён Istio, а sidecar искажал бы потребление памяти тестового контейнера.

HPA масштабирует pod-ы приложения, а не базу данных. В задании фраза про «реплики базы данных» относится не к этому тестовому образу; в evidence показано изменение числа реплик deployment `scaletestapp`.
