# Envoy Gateway Domains Helm Chart

Helm-чарт для управления доменами и маршрутами через Gateway API с использованием Envoy Gateway.

## Предварительные требования

- Kubernetes кластер с Gateway API CRD
- Установленный Envoy Gateway контроллер
- cert-manager для автоматических TLS сертификатов

## Установка Envoy Gateway

```bash
helm upgrade --install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.5.5 \
  -n envoy-gateway-system \
  --create-namespace
```

### Установка на выделенную ноду с taint

Если нода имеет taint (например, `dedicated=envoy`), используйте:

```bash
helm upgrade --install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.5.5 \
  -n envoy-gateway-system \
  --set deployment.envoyGateway.nodeSelector."kubernetes\.io/hostname"=ingress-01
```

## Установка чарта доменов

```bash
helm install envoy-gateway . -f values.yaml
```

или напрямую через kubectl:

```bash
helm template . | kubectl apply -f -
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
    routes:
      - name: dotops-lending
        path: /
        service:
          name: dotops-lending
          namespace: dotops-lending  # Обязательно указывать namespace
          port: 80
```

### Добавление нескольких доменов

```yaml
hosts:
  - host: dotops.ru
    routes:
      - name: web-service
        path: /
        service:
          name: web-service
          namespace: production
          port: 80
          
  - host: app.dotops.ru
    routes:
      - name: app-backend
        path: /
        service:
          name: app-backend
          namespace: production
          port: 3000
```

## TLS/HTTPS (cert-manager)

Чарт автоматически настраивает получение TLS сертификатов через cert-manager и Let's Encrypt.

### Настройка cert-manager

```yaml
# values.yaml
certManager:
  enabled: true
  email: your-email@example.com
```

Это создаст:
- `ClusterIssuer` для Let's Encrypt (production)
- `Certificate` ресурс для автоматического получения сертификата
- Автоматическое продление сертификата

### Важно для работы ACME HTTP-01 challenge

1. DNS домена должен указывать на IP адрес Gateway
2. Порт 80 должен быть доступен извне для проверки Let's Encrypt
3. CoreDNS в кластере должен иметь доступ к внешнему DNS

## Размещение на выделенной ноде

Если Envoy Gateway должен работать на конкретной ноде с taint, используйте EnvoyProxy CRD (уже включен в чарт):

```yaml
# templates/envoyproxy.yaml автоматически создаётся с:
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
spec:
  provider:
    kubernetes:
      envoyDeployment:
        pod:
          nodeSelector:
            kubernetes.io/hostname: ingress-01
          tolerations:
          - key: dedicated
            operator: Equal
            value: envoy
            effect: NoSchedule
      envoyService:
        type: LoadBalancer
        externalTrafficPolicy: Cluster
```

## Проверка статуса

```bash
# Gateway
kubectl get gateway -n envoy-gateway-system

# HTTPRoute
kubectl get httproute -n envoy-gateway-system

# Certificate
kubectl get certificate -n envoy-gateway-system

# Проверка сайта
curl -I https://dotops.ru
```

## Структура чарта

- `templates/gatewayclass.yaml` — GatewayClass с параметрами EnvoyProxy
- `templates/gateway.yaml` — Gateway с HTTP/HTTPS listeners
- `templates/httproute.yaml` — HTTPRoute для каждого хоста
- `templates/envoyproxy.yaml` — EnvoyProxy с nodeSelector и tolerations
- `templates/referencegrant.yaml` — ReferenceGrant для cross-namespace доступа
- `templates/cert-manager-issuer.yaml` — ClusterIssuer для Let's Encrypt
- `templates/certificate.yaml` — Certificate для автоматического TLS
