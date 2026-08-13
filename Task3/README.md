# Task3 — проверка HPA на локальном Minikube

## Что настроено

- `deployment.yaml` запускает `ghcr.io/yandex-practicum/scaletestapp:latest`, стартует с одной реплики и задаёт лимит памяти `30Mi`.
- `service.yaml` открывает приложение внутри кластера на порту `8080`.
- `hpa.yaml` масштабирует развёртывание по памяти: цель `80%`, от 1 до 10 реплик.
- `locustfile.py` содержит сценарий из задания: пользователь вызывает `GET /`.

`resources.requests.memory` выставлен в `8Mi`, лимит оставлен `30Mi`. HPA считает процент утилизации от запрошенной памяти, поэтому с таким значением видно масштабирование до того, как контейнер упирается в лимит.

## Команды запуска

```bash
minikube start --cpus=4 --memory=6144
minikube addons enable metrics-server
kubectl apply -f Task3/deployment.yaml
kubectl apply -f Task3/service.yaml
kubectl apply -f Task3/hpa.yaml
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
- `hpa-watch-balanced.log` — динамика под нагрузкой: HPA увидел `memory: 367%/80%` и начал увеличивать число подов с 1 до 2 и 4.
- `after-load.log` — состояние после повторного прогона: развёртывание дошло до 10 подов, внутри есть `kubectl describe hpa` и события `SuccessfulRescale`.
- `kubernetes-dashboard-scaletestapp-full.png` — скриншот Minikube Dashboard: развёртывание `scaletestapp`, `Pods status: 10 / 10`.
- `locust-service-report.png` — скриншот отчёта Locust: около 4 тысяч запросов, ошибок 0.

## Проверка метрик приложения

```bash
curl http://localhost:8080/
curl http://localhost:8080/metrics | grep http_requests_total
```

## Замечание по локальной машине

Образ `ghcr.io/yandex-practicum/scaletestapp:latest` не содержит сборку для `linux/arm64`. На Apple Silicon заранее загружался вариант `linux/amd64` в Minikube; в манифесте оставлен `imagePullPolicy: IfNotPresent`. Также отключён вспомогательный контейнер Istio через `sidecar.istio.io/inject: "false"`: в локальном пространстве имён `default` уже был включён Istio, а его контейнер добавлял бы свою память к тестовому приложению.

HPA масштабирует поды приложения, а не базу данных. Фраза в задании про «реплики базы данных» к этому тестовому образу не относится; в evidence показано изменение числа реплик развёртывания `scaletestapp`.
