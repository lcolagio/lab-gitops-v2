# #############
# HVAULT
# #############

## Install client hvault
```shell
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install
vault --version   
```

##  Initialise hvault
```shell
oc project openshift-gitops
oc exec -it hvault-0 -- vault operator init -key-shares=1 -key-threshold=1
```

Unseal Key 1: fPNtKgPYwXPTsP+E6pIZx4Am7n3SK2/aH+kfaR/es0U=
Initial Root Token: hvs.LbYoA16vX96aVnVz3G6Mr1YL


# Replace Vault UnsealKey
```shell
UnsealKey=fPNtKgPYwXPTsP+E6pIZx4Am7n3SK2/aH+kfaR/es0U=
RootToken=hvs.LbYoA16vX96aVnVz3G6Mr1YL

cd /Users/lcolagio/git/lab-gitops-v2/cluster-XXXX/lab-vault-demo/vault-demo/hvault/templates
cat secret-key.yaml | sed -e "/^data:/{n;s/KEY:.*/KEY: $(echo ${UnsealKey} | base64)/;}" >secret-key.yaml

oc delete pod hvault-0
sleep 15
oc exec -it hvault-0 -- vault status
```
# configure Vault
```shell

oc rsh hvault-0
vault login hvs.xY2GP2wxHGHr4XqLC3MXIxba
   
## créer le kv engine dédié aux secrets w6
vault secrets enable -description="vault OCP4 pour w6" -path w6 -version=1 kv
 
## création des permissions d'accès aux secrets
 
# policy pour mettre à jour les secrets
vault policy write w6 - << EOF
path "w6/*" {
  capabilities = ["create", "read", "update", "patch", "delete", "list"]
}
EOF
 
# policy utilisé par le plugin argocd
vault policy write w6-ro - << EOF
path "w6/*" {
  capabilities = ["read"]
}
EOF

# Creation des tokens
vault token create -display-name=w6 -metadata=name=w6 -policy=w6
vault token create -display-name=w6-ro -metadata=name=w6-ro -policy=w6-ro
 
## Si utilisation de l'authentification k8s
vault auth enable kubernetes
vault write auth/kubernetes/config kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
 
vault write auth/kubernetes/role/avp \
    bound_service_account_names=argocd-vault-plugin \
    bound_service_account_namespaces=openshift-gitops \
    policies=w6-ro \
    ttl=24h
```

