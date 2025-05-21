# Demo API Gateway

show dashboard and test access to api/whoami

## Upgrade from OSS to Hub

```bash
gsed -i "s/path: ingresses\/3-proxy\/overlays/path: ingresses\/1-hubgtw\/overlays/g" tools/argocd/applications.yaml
gsed -i "s/path\: middlewares\/overlays\/proxy/path\: middlewares\/overlays\/hub/g" tools/argocd/applications.yaml
gsed -i "s/path\: traefik\/overlays\/proxy/path\: traefik\/overlays\/hub-gtw/g" tools/argocd/applications.yaml
kubectl apply -f tools/argocd/applications.yaml
```

show dashboard and test access to api/whoami

## apply oidc to whoami

```bash
kubectl apply -f ingresses/3-hubgtw/overlays/whoami
```

Test access to frontend

## secure api with jwt and api key

```bash
kubectl apply -f ingresses/3-hubgtw/overlays/customers
```

Test access to api with get and post

## add rate-limit and publish external api

```bash
kubectl apply -f ingresses/3-hubgtw/overlays/employees
kubectl apply -f ingresses/3-hubgtw/overlays/external-graphql
```

test rl and show headers

## observability

show some metrics and logs
