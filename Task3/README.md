# Task3 — запуск и доказательства работы HPA

## Предпосылки

Установлены Docker, `minikube`, `kubectl`, Python 3 и Locust (`python3 -m pip install locust`). HPA ориентируется на память, поэтому `metrics-server` обязателен.

## Запуск

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

Откройте сервис в отдельном терминале:

```bash
kubectl port-forward service/scaletestapp 8080:8080
```

В ещё одном терминале, из папки `Task3`, запускайте Locust:

```bash
locust --host=http://localhost:8080
```

Откройте `http://localhost:8089`. Для эксперимента начните с 100–300 пользователей и hatch rate 20–50. Если память не достигает 80%, плавно увеличьте число пользователей: конкретный порог зависит от ресурсов локального компьютера и версии образа.

## Что приложить после теста

Результат не создан искусственно: его нужно снять с вашего Minikube после реальной нагрузки. Сохраните в `evidence/`:

```bash
mkdir -p evidence
kubectl get hpa -w | tee evidence/hpa-watch.log
# после появления роста реплик остановите Ctrl+C
kubectl get deployment scaletestapp -o wide > evidence/deployment-after-load.log
kubectl get pods -l app=scaletestapp -o wide > evidence/pods-after-load.log
kubectl top pods -l app=scaletestapp > evidence/pods-memory-after-load.log
kubectl describe hpa scaletestapp > evidence/hpa-describe.log
```

Также можно положить скриншот `minikube dashboard` с количеством реплик и потреблением памяти. Для ревью важны признаки: `TARGETS` около/выше 80%, увеличение `REPLICAS` выше 1, несколько pod'ов приложения и события `SuccessfulRescale` в `describe hpa`.

## Проверка метрик приложения

```bash
curl http://localhost:8080/
curl http://localhost:8080/metrics | grep http_requests_total
```

## Ограничение эксперимента

`HPA` масштабирует **поды приложения**, а не базу данных. Формулировка задания о «репликах базы данных» вероятно является оговоркой. В этой работе доказательством служит изменение числа реплик Deployment `scaletestapp`.
