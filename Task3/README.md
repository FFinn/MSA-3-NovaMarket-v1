# Task3 — проверка HPA на локальном Minikube

## Что настроено

- `deployment.yaml` запускает `ghcr.io/yandex-practicum/scaletestapp:latest`, стартует с одной реплики и задаёт лимит памяти `30Mi`.
- `service.yaml` открывает приложение внутри кластера на порту `8080`.
- `hpa.yaml` масштабирует deployment по памяти: цель `80%`, от 1 до 10 реплик.
- `locustfile.py` содержит сценарий из задания: пользователь вызывает `GET /`.

`resources.requests.memory` выставлен в `8Mi`, лимит оставлен `30Mi`. HPA считает процент утилизации от request, поэтому с таким request видно масштабирование до того, как контейнер упирается в лимит памяти.

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

Для локального доступа к приложению использовался туннель Minikube:

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

Во время более тяжёлой нагрузки отдельно снималось состояние HPA:

```bash
kubectl get hpa scaletestapp
kubectl get deployment scaletestapp -o wide
kubectl get pods -l app=scaletestapp -o wide
kubectl top pods -l app=scaletestapp
kubectl describe hpa scaletestapp
```

## Доказательства

Что лежит в `Task3/evidence/`:

- `before-load.log` — состояние перед нагрузкой: одна реплика.
- `hpa-watch-balanced.log` — динамика под нагрузкой: HPA увидел `memory: 367%/80%` и начал увеличивать число pod-ов с 1 до 2 и 4.
- `after-load.log`, `hpa-describe.log`, `events-after-load.log` — состояние после повторного прогона: deployment дошёл до 10 pod-ов, есть события `SuccessfulRescale`.
- `kubernetes-dashboard-scaletestapp-full.png` — скриншот Minikube Dashboard: deployment `scaletestapp`, `Pods status: 10 / 10`.
- `locust-service-report.html` и `locust-service-report.png` — отчёт Locust: около 4 тысяч запросов, ошибок 0.
- `locust-service_*.csv` — CSV-выгрузка Locust.

## Проверка метрик приложения

```bash
curl http://localhost:8080/
curl http://localhost:8080/metrics | grep http_requests_total
```

## Замечание по локальной машине

Образ `ghcr.io/yandex-practicum/scaletestapp:latest` не содержит `linux/arm64` manifest. На Apple Silicon пришлось заранее скачать вариант `linux/amd64`, загрузить его в Minikube и оставить в deployment `imagePullPolicy: IfNotPresent`. Ещё отключена инъекция Istio sidecar через `sidecar.istio.io/inject: "false"`: в локальном `default` namespace уже был включён Istio, и sidecar добавлял бы свою память к тестовому контейнеру.

HPA масштабирует pod-ы приложения, а не базу данных. Фраза в задании про «реплики базы данных» к этому тестовому образу не относится; в evidence показано изменение числа реплик deployment `scaletestapp`.
