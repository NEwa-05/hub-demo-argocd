# hub-demo-argocd

This repository contains a service that generates installation manifests for installing Hub in a GitOps-compliant fashion with ArgoCD

## Deploying the stack locally

Login to the Traefik Hub UI, add a new agent and copy your token within the env file.

### Source your variables (tokens, dns, ...)

```bash
source .env
```

### Create K3d cluster

```bash
sudo k3d cluster create demo-argo --port 80:80@loadbalancer --port 443:443@loadbalancer --k3s-arg "--disable=traefik@server:0" --k3s-arg="--cluster-domain=${CLUSTERNAME}.${DOMAINNAME}@server:0"
```

generate certificate from variables:

```bash
LEGO_DISABLE_CNAME_SUPPORT=true lego -a --path .lego/. --email mageekbox@gmail.com --dns gandiv5 -d "${CLUSTERNAME}.${DOMAINNAME}" -d "*.${CLUSTERNAME}.${DOMAINNAME}" run
```

Set new branch for dedicated env on demand:

```bash
git checkout -b $CLUSTERNAME
```

Set kustomozided domain:

```bash
for i in $(grep -Rl --exclude "*.md" --exclude ".env" --exclude ".kubeconfig" '${CLUSTERNAME}'); do gsed -i 's/${CLUSTERNAME}/'$CLUSTERNAME'/g' $i; done
for i in $(grep -Rl --exclude "*.md" --exclude ".env" --exclude ".kubeconfig" '${DOMAINNAME}'); do gsed -i 's/${DOMAINNAME}/'$DOMAINNAME'/g' $i; done
```

### Secrets

Let's use sops and age to make secret hidden and managed by argocd.

Install age and sops

```bash
brew install sops age
```

Generate key:

```bash
age-keygen -o age.agekey
```

create sops config

```bash
cat <<EOF | tee .sops.yaml
creation_rules:
  - path_regex: .*.yml
    encrypted_regex: '^(data|stringData)$'
    age: $(awk -F"key:" '{print $2}' age.agekey |tr -d '\n'' ')
EOF
```

Configure key info

```bash
export SOPS_AGE_KEY_FILE=age.agekey
```

Untrack secret values changes

```bash
git update-index --assume-unchanged .env
git update-index --assume-unchanged tools/argocd/repository/repo-secret.yaml
git update-index --assume-unchanged age.agekey
git update-index --assume-unchanged .sops.yaml
```

generate secret from variables:

```bash
gsed -i "s|\${TLSCRT}|$(cat .lego/certificates/$CLUSTERNAME.$DOMAINNAME.crt|base64)|g" traefik/base/wildcard-cert.yaml
gsed -i "s|\${TLSKEY}|$(cat .lego/certificates/$CLUSTERNAME.$DOMAINNAME.key|base64)|g" traefik/base/wildcard-cert.yaml
sops -e -i traefik/base/wildcard-cert.yaml
gsed -i 's/${HUB_TOKEN}/'$HUB_TOKEN'/g' traefik/base/hub-secret.yaml 
sops -e -i traefik/base/hub-secret.yaml
gsed -i 's/${GANDIV5_API_KEY}/'$GANDIV5_API_KEY'/g' traefik/base/dns-secret.yaml
sops -e -i traefik/base/dns-secret.yaml
gsed -i 's/${GH_TOKEN}/'$GH_TOKEN'/g' tools/base/observability/gh-secret.yaml
sops -e -i tools/base/observability/gh-secret.yaml
gsed -i 's/${KEYCLOAK_PASSWORD}/'$KEYCLOAK_PASSWORD'/g' tools/base/keycloak/keycloak-secret.yaml
sops -e -i tools/base/keycloak/keycloak-secret.yaml
gsed -i 's/${USER_TOKEN}/'$USER_TOKEN'/g' ingresses/3-hubmgt/base/traffic/user-token-secret.yaml
gsed -i 's/${USER2_TOKEN}/'$USER2_TOKEN'/g' ingresses/3-hubmgt/base/traffic/user-token-secret.yaml
sops -e -i ingresses/3-hubmgt/base/traffic/user-token-secret.yaml
gsed -i 's/${TREBLLE_APIKEY}/'$TREBLLE_APIKEY'/g' middlewares/base/hub/treblle-middleware.yaml
gsed -i 's/${TREBLLE_PROJECTID}/'$TREBLLE_PROJECTID'/g' middlewares/base/hub/treblle-middleware.yaml
```

Update git repository with overlay values:

```bash
git add . && git commit -m "${CLUSTERNAME}" && git push origin ${CLUSTERNAME}
```

Set the private key and known host secret within clusters/demo/repo-secret.yaml

Deploy ArgoCD for api gateway demo:

```bash
export ARGO_PWD_ENCRYPTED=$(htpasswd -nbBC 10 "" $ARGO_PWD | tr -d ':\n' | sed 's/$2y/$2a/')
helm upgrade --install redis bitnami/redis --namespace redis --values tools/redis/values.yaml --create-namespace
helm upgrade --install argocd argo-cd/argo-cd -f tools/argocd/values.yaml --namespace argocd --create-namespace --set configs.secret.argocdServerAdminPassword=${ARGO_PWD_ENCRYPTED}
cat age.agekey | kubectl create secret generic sops-age --namespace=argocd --from-file=age.agekey=/dev/stdin
envsubst < tools/argocd/repository.yaml | kubectl apply -f -
kubectl apply -f tools/argocd/applications.yaml
```

### Add GatewayAPI CRDs if needed

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/experimental-install.yaml
```

```bash
k apply -f tools/argocd/ingress.yaml
```
