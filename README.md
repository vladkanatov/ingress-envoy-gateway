# Envoy Gateway Helm Chart

Helm chart для развертывания Envoy Gateway с Envoy Proxy и интеграцией cert-manager.

## Установка

```bash
helm install envoy-gateway . -f values.yaml
```

## Конфигурация

### Основные параметры

- **hosts** — список доменов и маршрутов к сервисам
- **acmeEmail** — email для получения ACME-сертификатов
- **dedicatedNode** — настройка для выделенных нод под Envoy
- **gateway.className** — имя Gateway класса (по умолчанию: `envoy`)
- **certManager.version** — версия cert-manager

### Пример добавления доменов и роутов

```yaml
hosts:
  - host: dotops.ru
    tlsSecret: envoy-dotops-ru-cert
    routes:
      - name: api-service
        path: /api
        service:
          name: api-service
          port: 8080
      - name: web-service
        path: /
        service:
          name: web-service
          port: 80
  - host: app.dotops.ru
    tlsSecret: envoy-app-cert
    routes:
      - name: app-backend
        path: /
        service:
          name: app-backend
          port: 3000
```

## Проверка шаблонов

```bash
helm template envoy-gateway . -f values.yaml
```

## Структура

- `templates/gateway.yaml` — Gateway ресурс (HTTP/HTTPS listeners)
- `templates/httproute.yaml` — HTTPRoute для каждого хоста
- `templates/envoy-proxy-daemonset.yaml` — DaemonSet с Envoy Proxy
- `templates/envoy-gateway-deploy.yaml` — Deployment Envoy Gateway
- `templates/gatewayclass.yaml` — GatewayClass ресурс
