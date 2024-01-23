## Fetch vault secrets using VSO

### Add vault helm repository
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

### Deploy vault server
```
helm install vault hashicorp/vault -n vault --create-namespace --set server.dev.enabled=true
```

### Add secrets
First, exec into the vault pod.
1. Enable kvv2 secrets engine
    ```
    vault secrets enable kv-v2
    ```
2. Add secret for the fakeapp
    ```
    vault kv put kv-v2/fakeapp/mysecret username=foo password=bar
    ```
3. Add policy to read the secret
    ```
    vault policy write mysecret - << EOF
        path "kv-v2/data/fakeapp/mysecret" {
        capabilities = ["read"]
        }
    EOF
    ```

### Configure kubernetes auth

1. Enable kubernetes auth backend
    ```
    vault auth enable kubernetes
    ```
2. Configure kubernetes endpoint
    ```
    vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443”
    ```
3. Create a role to enable access to the secret
    ```
    vault write auth/kubernetes/role/fakeapp \
      bound_service_account_names=default \
      bound_service_account_namespaces=fakeapp \
      policies=default,mysecret \
      audience=vault \
      ttl=24h
    ```

### Deploy VSO
```
helm install vault-secrets-operator hashicorp/vault-secrets-operator -n vault-secrets-operator-system \
    --create-namespace --values vso-values.yaml
```

### Deploy manifests
1. Create new namespace
    ```
    kubectl create ns fakeapp
    ```
2. Apply VaultAuth
    ```
    kubectl apply -f vault-auth.yaml
    ```
3. Apply VaultStaticSecret
    ```
    kubectl apply -f kv-secret.yaml
    ```
4. Deploy application
    ```
    kubectl apply -f deployment.yaml
    ```
