# Задание 2. Динамическое масштабирование

## Запуск

```bash
minikube start -p scaletest
minikube addons enable metrics-server -p scaletest
minikube stop -p scaletest
```

Все ресурсы в namespace `scaletest`. Образ `scaletestapp:local` собрал из исходников scaletestapp, поэтому `imagePullPolicy: Never`.

## Часть 1. Масштабирование по памяти
### Приложение и сервис

![Ответ приложения: идентификатор пода](artifacts/browser_localhost.png)
![Метрики приложения, есть http_requests_total](artifacts/browser_localhost_metrics.png)

Под без нагрузки занимает 11Mi:
![kubectl top pod без нагрузки](artifacts/top_pod_1.png)
![Применил service.yaml, запустил minikube tunnel](artifacts/apply_service.png)
![Сервис получил EXTERNAL-IP 127.0.0.1](artifacts/get_svc.png)
![Применил hpa-memory.yaml, 1 под](artifacts/apply_HPA.png)

### Нагрузка

1. Сначала запускал Locust локально через туннель (`http://127.0.0.1:8080`), 100 -> 1500 пользователей. Память выросла с 31% до 91%, HPA добавил второй под. Через туннель 26% ошибок, поэтому решил попробовать запустить Locust в кластере.

![Locust локально, до 1500 пользователей](artifacts/locus_dashboard.png)
![kubectl get hpa -w: 1 -> 2 пода](artifacts/watch_hpa_locus_600_user.png)

2. Locust в кластере (`http://scaletestapp:8080`), до 8000 пользователей, 0% ошибок. Память дошла до 104%, HPA увеличил число подов 2 -> 3 -> 4 -> 5 -> 7 -> 10 (максимум).

![Locust в кластере, до 8000 пользователей](artifacts/locus_dashboard_8000.png)
![kubectl get hpa -w: 2 -> 10 подов](artifacts/watch_hpa_locus_8000_user.png)

Вывод HPA: [hpa-memory-describe.txt](artifacts/hpa-memory-describe.txt), события: [hpa-memory-events.txt](artifacts/hpa-memory-events.txt).

3. Нагрузку выключил, а поды не удаляются: память после нагрузки не освобождается, в среднем около 105% от request. HPA считает, что поды все еще нужны.

![kubectl top pod: 10 подов после остановки нагрузки](artifacts/10_pods_after_locus.png)

Вывод: по памяти HPA масштабирует вверх, но не вниз. Для этого приложения память плохо отражает нагрузку.

## Часть 2. Масштабирование по RPS

- [kube-prometheus-stack-values.yaml](kube-prometheus-stack-values.yaml) - Prometheus + Grafana через helm.
- [servicemonitor.yaml](servicemonitor.yaml) - Prometheus собирает `/metrics` с каждого пода раз в 15 с
- [prometheus-adapter-values.yaml](prometheus-adapter-values.yaml) - адаптер превращает счетчик `http_requests_total` в `http_requests_per_second` (rate за 1 мин по каждому поду) и отдает его в Kubernetes через custom metrics API
- [hpa-rps.yaml](hpa-rps.yaml) - HPA: в среднем 10 RPS на под, от 1 до 10 подов; при спаде нагрузки ждет 60 с и убирает не больше 50% подов за 30 с

### Prometheus

Все 10 подов приложения в Targets в статусе UP:
![Prometheus Targets: serviceMonitor scaletestapp 10/10 up](artifacts/Prometheus_install.png)

Счетчик `http_requests_total` по каждому поду:
![Prometheus: запрос http_requests_total](artifacts/Prometheus_http_requests_total.png)

RPS по подам в Grafana под нагрузкой Locust (1000 пользователей):
![Grafana: RPS по подам](artifacts/Grafana_http_requests_total.png)
![Locust в кластере, 1000 пользователей](artifacts/locus_http_requests_total.png)

### Адаптер

API custom metrics доступен, в нем есть `pods/http_requests_per_second`:
![prometheus-adapter: apiservice доступен, метрика http_requests_per_second](artifacts/prometheus_adapter.png)

### Проверка HPA по RPS

До 20:13 работал HPA по памяти и держал 10 подов. После замены на hpa-rps число подов меняется вслед за нагрузкой, а после выключения нагрузки падает до 1:
![Grafana: RPS по подам и количество подов](artifacts/auto_kill_pods.png)

`10146m/10` в TARGETS - это 10.1 запроса в секунду на под при цели 10 (`m` - тысячные доли). Когда RPS падает до 0, HPA примерно за 2 минуты сокращает поды 10 -> 5 -> 4 -> 2 -> 1:
![kubectl get hpa -w: масштабирование по RPS и обратно до 1 пода](artifacts/kubectl_get_hpa_for_kill.png)
