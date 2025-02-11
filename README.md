# hub-gateway-ai
Testing Hub Gateway AI capabilities

## deploy Redis

### For distributed features

```bash
helm upgrade --install redis bitnami/redis --namespace redis --values redis/values.yaml --create-namespace
```

## Deploy Hub

### Create Hub namespace

```bash
kubectl create ns traefik
```

### Create Hub token secret

```bash
kubectl create secret generic hub-license --from-literal=token="${HUB_TOKEN}" -n traefik
```

### deploy Hub

```bash
helm upgrade --install traefik traefik/traefik --create-namespace --namespace traefik --values hub/hub-values.yaml
```

### Set DNS entry

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "*.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/oss-traefik -n oss-traefik --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

```bash
curl --location --request POST "https://cf.infra.traefiklabs.tech/dns/env-on-demand" \
--header "X-TraefikLabs-User: ${CLUSTERNAME}" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer ${DNS_BEARER}" \
--data-raw "{
    \"Value\": \"$(kubectl get svc/traefik -n traefik --no-headers | awk {'print $4'})\"
}"
```

### Deploy dashboard ingress

```bash
envsubst < hub/ingress.yaml | kubectl apply -f -
```

## Deploy Monitoring

```bash
helm upgrade --install loki grafana/loki --create-namespace --namespace observability --values observability/loki/values.yaml
kubectl apply -f observability/jaeger
helm upgrade --install promtail grafana/promtail --create-namespace --namespace observability --values observability/promtail/values.yaml
kubectl create configmap grafana-traefik-dashboards --from-file=observability/prometheus-stack/traefik.json -o yaml --dry-run=client -n observability | kubectl apply -f -
helm upgrade -i prometheus-stack prometheus-community/kube-prometheus-stack -f observability/prometheus-stack/values.yaml --namespace=observability
envsubst < observability/prometheus-stack/ingress.yaml | kubectl apply -f -
```

## Test AI

### Deploy AI services

```bash
envsubst < ai/aiservice.yaml | kubectl apply -f -
```

### Through Gateway

```bash
envsubst < ai/gateway/middlewares.yaml | kubectl apply -f -
envsubst < ai/gateway/ingress.yaml | kubectl apply -f -
```

### test gateway

```bash
curl --location https://ai.${CLUSTERNAME}.${FULLDOMAINNAME} \
  -H "Authorization: Bearer ${JWT_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "messages": [
      {
        "role": "user",
        "content": "Do you know what Traefik Hub is?"
      }
    ]
  }' | jq
```

### Through Management

```bash
envsubst < ai/management/oas.yaml | kubectl apply -f -
kubectl apply -f ai/management/api.yaml
kubectl apply -f ai/management/api-sub.yaml
envsubst < ai/management/ingress.yaml | kubectl apply -f -
envsubst < ai/management/portal-ingress.yaml | kubectl apply -f -
envsubst < ai/management/portal.yaml | kubectl apply -f -
```

### test management

```bash
curl --location https://ai.${CLUSTERNAME}.${FULLDOMAINNAME} \
  -H "Authorization: Bearer ${HUBAPIKEY4AI}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "messages": [
      {
        "role": "user",
        "content": "Do you know what Traefik Hub is?"
      }
    ]
  }' | jq
```
