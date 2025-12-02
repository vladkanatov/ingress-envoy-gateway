# Envoy Gateway Domains Helm Chart

Helm-чарт для управления доменами и маршрутами через Gateway API с уже установленным Envoy Gateway контроллером.

## Предварительные требования

- Kubernetes кластер с Gateway API CRD
- Установленный Envoy Gateway контроллер (например, через официальный Helm-чарт или манифесты)
- GatewayClass `envoy` должен быть создан и управляться Envoy Gateway контроллером

## Установка

```bash
helm install envoy-domains . -f values.yaml
```

## Обновление

```bash
helm upgrade envoy-domains . -f values.yaml
```

## Конфигурация

### Основные параметры в `values.yaml`

#### Gateway настройки

```yaml
gateway:
  className: envoy                    # Имя GatewayClass
  namespace: envoy-gateway-system     # Namespace где развернут Gateway
  name: envoy-gateway                 # Имя Gateway ресурса
  tlsSecretName: envoy-domains-cert   # Секрет с TLS сертификатом
```

#### Хосты и маршруты

```yaml
hosts:
  - host: dotops.ru
    tlsSecret: envoy-dotops-ru-cert  # Опционально: индивидуальный TLS секрет
    routes:
      - name: dotops-lending
        path: /
        service:
          name: dotops-lending
          port: 80
```

### Добавление нескольких доменов

```yaml
hosts:
  - host: dotops.ru
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
    routes:
      - name: app-backend
        path: /
        service:
          name: app-backend
          port: 3000
```

## Проверка шаблонов

```bash
helm template envoy-domains . -f values.yaml
```

## Применение изменений

```bash
helm template . | kubectl apply -f -
```

## Структура чарта

- `templates/gatewayclass.yaml` — GatewayClass ресурс (опционально, если еще не создан)
- `templates/gateway.yaml` — Gateway ресурс с HTTP/HTTPS listeners
- `templates/httproute.yaml` — HTTPRoute для каждого хоста из `values.yaml`

## TLS/HTTPS

### Автоматическое получение сертификатов (cert-manager)

Чарт включает поддержку cert-manager для автоматического получения сертификатов от Let's Encrypt.

**Требования:**
- Установленный cert-manager в кластере
- Рабочий DNS в кластере (для доступа к ACME API)
- Внешний доступ к Gateway для прохождения HTTP-01 challenge

**Включение cert-manager:**

```yaml
# values.yaml
certManager:
  enabled: true
  email: your-email@example.com
```

**Проблемы с DNS:**

Если в кластере не работает внешний DNS (как в текущем случае), cert-manager не сможет обратиться к Let's Encrypt API. В этом случае используйте ручное создание сертификата или настройте DNS01 challenge.

### Ручное создание TLS сертификата

Для HTTPS необходимо создать Kubernetes Secret с TLS сертификатом:

```bash
kubectl create secret tls envoy-domains-cert \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n envoy-gateway-system
```

Или использовать cert-manager для автоматического получения сертификатов:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: envoy-domains-cert
  namespace: envoy-gateway-system
spec:
  secretName: envoy-domains-cert
  dnsNames:
    - dotops.ru
    - "*.dotops.ru"
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```
