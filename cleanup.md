source .env
for i in $(grep -Rl --exclude "*.md" --exclude ".env" --exclude ".kubeconfig" ${CLUSTERNAME}); do gsed -i 's/'${CLUSTERNAME}'/${CLUSTERNAME}/g' $i; done
for i in $(grep -Rl --exclude "*.md" --exclude ".env" --exclude ".kubeconfig" ${DOMAINNAME}); do gsed -i 's/'${DOMAINNAME}'/${DOMAINNAME}/g' $i; done
export SOPS_AGE_KEY_FILE=age.agekey
sops -d -i traefik/base/wildcard-cert.yaml
gsed -i 's/tls.crt:.*$/tls.crt: ${TLSCRT}/g' traefik/base/wildcard-cert.yaml
gsed -i 's/tls.key:.*$/tls.key: ${TLSKEY}/g' traefik/base/wildcard-cert.yaml
sops -d -i traefik/base/hub-secret.yaml
gsed -i 's/token:.*$/token: ${HUB_TOKEN}/g' traefik/base/hub-secret.yaml
sops -d -i traefik/base/dns-secret.yaml
gsed -i 's/token:.*$/token: ${GANDIV5_API_KEY}/g' traefik/base/dns-secret.yaml
sops -d -i tools/base/observability/gh-secret.yaml
gsed -i 's/GH_TOKEN:.*$/GH_TOKEN: ${GH_TOKEN}/g' tools/base/observability/gh-secret.yaml
sops -d -i tools/base/keycloak/keycloak-secret.yaml
gsed -i 's/password:.*$/password: \"${KEYCLOAK_PASSWORD}\"/g' tools/base/keycloak/keycloak-secret.yaml
sops -d -i ingresses/3-hubmgt/base/traffic/user-token-secret.yaml
gsed -i 's/token1:.*$/token1: ${USER_TOKEN}/g' ingresses/3-hubmgt/base/traffic/user-token-secret.yaml
gsed -i 's/token2:.*$/token2: ${USER2_TOKEN}/g' ingresses/3-hubmgt/base/traffic/user-token-secret.yaml
gsed -i 's/ProjectId:.*$/ProjectId: ${TREBLLE_PROJECTID}/g' middlewares/base/hub/treblle-middleware.yaml
gsed -i 's/ApiKey:.*$/ApiKey: ${TREBLLE_APIKEY}/g' middlewares/base/hub/treblle-middleware.yaml
