## Usefull Link
* wiki: https://wiki.net.extra.xxxx.fr/confluence/display/ITAAS/Sealed+secrets
* Create certificat: https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/bring-your-own-certificates.md

## Exemple Helm Installation

### Install helm Cli
```shell
wget https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64
mv helm-linux-amd64 helm
chmod +x helm
sudo mv helm /usr/bin/helm
helm version
```

### Install sealed-secrets
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm install sealed-secrets -n default --set-string fullnameOverride=sealed-secrets-controller bitnami/sealed-secrets

# Ne fonctionne pas pour l'instant
# # Test 1
# # https://github.com/bitnami/charts/tree/main/bitnami/sealed-secrets/#installing-the-chart 
# helm repo add bitnami https://charts.bitnami.com/bitnami
# helm repo update
# helm repo list
# helm install sealed-secrets -n ops-kubeseal --set-string fullnameOverride=sealed-secrets-controller --set containerSecurityContext.enabled=false bitnami/sealed-secrets 

# # Test 2
# helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
# helm install sealed-secrets -n ops-kubeseal --set-string fullnameOverride=sealed-secrets-controller bitnami/sealed-secrets
```

## create certificate
```shell
export NAMESPACE="default"
export PRIVATEKEY="sealed-secrets-from-gitops.key"
export PUBLICKEY="sealed-secrets-from-gitops.crt"
export SECRETNAME="sealed-secrets-from-gitops"

openssl req -x509 -nodes -newkey rsa:4096 -keyout "$PRIVATEKEY" -out "$PUBLICKEY" -subj "/CN=sealed-secret/O=sealed-secret"
```

## Configure Kubeseal
```shell
kubectl -n "$NAMESPACE" create secret tls "$SECRETNAME" --cert="$PUBLICKEY" --key="$PRIVATEKEY"
kubectl -n "$NAMESPACE" label secret "$SECRETNAME" sealedsecrets.bitnami.com/sealed-secrets-key=active

kubectl -n "$NAMESPACE" delete pod -l app.kubernetes.io/name=sealed-secrets
sleep 10
kubectl -n "$NAMESPACE" logs -l app.kubernetes.io/name=sealed-secrets
```

### Install kubeseal CLI
* Download kubeseal: https://github.com/bitnami-labs/sealed-secrets/tags

```shell
version=0.19.1
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v${version}/kubeseal-${version}-linux-amd64.tar.gz
tar xfz kubeseal-${version}-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

## Test Sealed Secret
```shell
oc new-project lab-sealedsecret

cat << EOF >my-secret.yaml
kind: Secret
apiVersion: v1
metadata:
  name: my-secret
data:
  password: $(echo Hello | base64)
type: Opaque
EOF

kubeseal --cert "./${PUBLICKEY}" --scope cluster-wide < my-secret.yaml | kubectl apply -f-
oc get sealedsecret my-secret
oc get secret my-secret

oc get sealedsecret my-secret -o go-template --template="{{.spec.encryptedData.password}}"
oc get secret my-secret -o go-template --template="{{.data.password|base64decode}}"
```



