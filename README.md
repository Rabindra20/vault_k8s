Clone this repo for the manifest file<br>

git clone : https://github.com/Rabindra20/vault_k8s.git - Connect your Github account <br>

Note:-change parameter a/c to requirement then<br>

cd into the vault-manifests directory of the cloned repository and execute the following command

```
kubectl apply -f .
```
Use same command for vault-agent-manifest after cloned repository from github and<br>

 Note please check all value in deployment and change namespace 

```
cd vault-injector-manifests
kubectl apply -f .
```

Initialize the vault to get unseal keys nad token and write it in keys.json file
```
kubectl exec vault-0 -- vault operator init -key-shares=3 -key-threshold=3 -format=json > keys.json
``` 

Getting the unseal key in env

```
VAULT_UNSEAL_KEY=$(cat keys.json | jq -r ".unseal_keys_b64[]")
echo $VAULT_UNSEAL_KEY
```
 

Getting the unseal key in the token

```
VAULT_ROOT_KEY=$(cat keys.json | jq -r ".root_token")
echo $VAULT_ROOT_KEY
```
 

Unseal the vault with this cmd

```
kubectl exec vault-0 -n namespace -- vault operator unseal ffffffffddddddssss
```

Login into the vault give us the token
```
kubectl exec vault-0 -- vault login $VAULT_ROOT_KEY
```
```
kubectl exec vault-0 -n namespace -- vault login xxxxxxxxxxxxxxx
```

(Just Go command Enable the vault server to make API calls to Kubernetes using the token, Kubernetes URL and the cluster API CA certificate for unsealed)<br>

Port-forward the vault to access the UI

```
kubectl port-forward -n namespace svc/vault 8200:8200
```

Enter to the pod

```
kubectl exec -it vault-0 -n namespace -- /bin/sh
```

Enabling secrets using the key-value engine v2 (for other stage change path=stage)

```
vault secrets enable -version=2 -path="dev" kv
``` 

Put the secrets inthe leave path(for other stage change vault kv put stage/apps/module name)

```
vault kv put dev/apps/demo-api/  \
DB_HOST=" "\
xxxxx="xx"
``` 

Get and view the secrets value

```
vault kv get dev/apps/auth/
```

Create the policy for the secrets

```
vault policy write demo-api-policy - <<EOH
path "dev/data/apps/demo-api" {
  capabilities = ["read"]
}
EOH
```
List the policy

```
vault policy list
```

Enter into the vault-0 pod

```
vault auth enable kubernetes
```
![j](https://github.com/Rabindra20/vault_k8s/assets/53372486/d3b401e9-f8ef-4058-bafb-0be770b221ff)
<br>
 
Enable the vault server to make API calls to Kubernetes using the token, Kubernetes URL and the cluster API CA certificate

```
vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt disable_iss_validation=true disable_local_ca_jwt=true
```

To verify iss_validation
```
vault read auth/kubernetes/config
```

Create the role to bound policy and Service Account
```
vault write auth/kubernetes/role/demo-api bound_service_account_names=vault bound_service_account_namespaces=namespace policies=demo-api-policy ttl=72h
``` 

Add Annotation and Service Account to deployment file<br>

After deploying
```
kubectl describe pod pod-name
```

Initalizing the pod (INIT_POD)

```
kubectl logs podname  -c vault-agent-init
``` 

 

### Troubleshoot
For fix unseal key error<br>

![Screenshot 2023-06-04 072149](https://github.com/Rabindra20/vault_k8s/assets/53372486/50ab8e55-9524-4832-9717-b13385b4dff8)


<br>
https://stackoverflow.com/questions/56471530/how-to-fix-vault-error-failed-to-read-request-counters-invalid-character-x00   
<br>    

https://www.vaultproject.io/api-docs/system/internal-counters


### For fix error authenticating<br>

If you get this type of error then Check the vault URL, policy, role and service account mapping.<br>

This happens when the vault agent is not able to connect to the vault server or the service account doest have permissions to read the secrets.<br>
 
![Screenshot 2023-06-04 07220](https://github.com/Rabindra20/vault_k8s/assets/53372486/cc179617-705a-431c-b989-d67aeb8d0d46)
 

### For QA env

Deploy SA with name space<br>

Deploy cluster role binding with name space