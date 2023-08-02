This is session #2 of learning Kubernetes, while debugging vault-agent-injector.

**Step 6.**

**Update the deployment with vault-agent-injector annotations, using kubectl replace.**
```
kubectl  replace -f nginx-deployment-annotated.yaml
```
**Step 7.**

**The next step is to confirm the annotations were applied to the deployment object.**
```
kubectl get deployment nginx-deployment -o json | jq -r .spec.template.metadata.annotations
```
**The output from the command shows the annotations were updated/patched successfully.**
```
{
  "vault.hashicorp.com/agent-inject": "true",
  "vault.hashicorp.com/agent-inject-secret-password.txt": "test/secret",
  "vault.hashicorp.com/role": "test-app"
}
```
**Step 8.**

**Confirm if the containers were injected into the nginx application pod.**
```
kubectl get pods -l app=nginx
```
```
NAME                               READY   STATUS     RESTARTS   AGE
nginx-deployment-585f4cdbf-6p4pw   0/2     Init:0/1   0          4s
```
**Pod status:**
  - Init:0/1 means none of the initContainers have completed so far.
  - PodInitializing or Running means the Pod has already finished executing the Init Containers.

**Step 9.**

**Check container logs:**
* Check logs of vault-agent-injector pod
* Check logs of initContainer 
* Check logs of K8s API
 
```
kubectl logs vault-agent-injector-8496df644f-kfnbb -f
kubectl logs nginx-deployment-585f4cdbf-6p4pw -c vault-agent-init -f
kubectl logs kube-apiserver-vault-test -n kube-system | grep inject
```

**The vault-agent-injector is reporting a 400 error invalid role name "test-app"**
```
2023-07-28T18:28:21.751Z [ERROR] auth.handler: error authenticating:
  error=
  | Error making API request.
  | 
  | URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
  | Code: 400. Errors:
  | 
  | * invalid role name "test-app"
   backoff=4m25.27s
```
**Step 10.**

**Check K8s auth config and auth role:**

`kubectl exec vault-0 -- vault read auth/kubernetes/config`
`kubectl exec vault-0 -- vault list auth/kubernetes/role`

Update deployment to use role `test-role` that exists in Vault.
Go to `nginx-deployment-annotated.yaml` and update annotation for `vault.hashicorp.com/role` to use the `test-role` role in Vault
Run `kubectl replace -f nginx-deployment-annotated.yaml` to redeploy ngnix deployment using updated role
Check logs from the init container
`kubectl logs -f nginx-deployment-74887f67b9-pml5r vault-agent-init`

```
2023-08-02T19:38:22.990Z [INFO]  agent.auth.handler: authenticating
2023-08-02T19:38:22.993Z [ERROR] agent.auth.handler: error authenticating:
  error=
  | Error making API request.
  |
  | URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
  | Code: 403. Errors:
  |
  | * namespace not authorized
   backoff=44.61s
```
Check k8s auth role config for authorized namespaces
`kubectl exec vault-0 -- vault read auth/kubernetes/role/test-role`

```
bound_service_account_names         [test-sa]
bound_service_account_namespaces    [vault]
```

Check namespace that application pod is running in 
`kubectl get pods vault-0 -o json | jq .metadata.namespace`
We see that the app pod is running in the default namespace but the bound_service_account_namespaces is set to vault
Update bound_service_account_namespaces to default
`kubectl exec vault-0 -- vault write auth/kubernetes/role/test-role bound_service_account_namespaces=default`

Confirm that namespace has been updated 
`kubectl exec vault-0 -- vault read auth/kubernetes/role/test-role`

Check logs from init container

`kubectl logs -f nginx-deployment-74887f67b9-pml5r vault-agent-init`

2023-08-02T19:42:42.216Z [INFO]  agent.auth.handler: authenticating
2023-08-02T19:42:42.234Z [ERROR] agent.auth.handler: error authenticating:
  error=
  | Error making API request.
  |
  | URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
  | Code: 403. Errors:
  |
  | * service account name not authorized
   backoff=4m37.45s

Check app pod for service account
`kubectl get pods vault-0 -o json | jq .spec.serviceAccount`

We see that the app pod is using the vault service account
Check k8s auth role config for service accounts
`kubectl exec vault-0 -- vault read auth/kubernetes/role/test-role`

```
bound_service_account_names         [test-sa]
bound_service_account_namespaces    [default]
```

We see that the bound_service_account_names for the role is set to test-sa but the app pod is using the vault service account

Update bound_service_account_namespaces for the role
`kubectl exec vault-0 -- vault write auth/kubernetes/role/test-role bound_service_account_names=vault`

Confirm that service account has been updated 
`kubectl exec vault-0 -- vault read auth/kubernetes/role/test-role`

Check logs from init container

`k logs -f nginx-deployment-74887f67b9-pml5r vault-agent-init`

`2023-08-02T19:59:28.132Z [INFO]  agent.auth.handler: auth handler stopped` means the init container rendered the secret then terminated, this means the init container has completed its job.





**Step 12.**
   
**Check initContainer logs:**
```
kubectl logs nginx-deployment-585f4cdbf-rrxdk  -c vault-agent-init
2023-08-01T19:27:06.607Z [WARN] (view) vault.read(test/secret): vault.read(test/secret): Error making API request.

URL: GET http://vault.default.svc:8200/v1/test/secret
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```
**Step 13.**

Check policy in Vault
`kubectl exec -ti vault-0 -- vault policy list`

Write k8s auth config role to reference test-policy
`kubectl exec vault-0 -- vault write auth/kubernetes/role/test-role policies=test-policy`

Reschedule nginx pod 


**Step 14.**

**Check pod status:**
```
kubectl get po -l app=nginx
```
```
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-68f94459b-6fx4z   2/2     Running   0          114m
```

**Re-Check logs of initContainer:**
```
kubectl logs nginx-deployment-585f4cdbf-4bxwk -c vault-agent-init

2023-08-01T16:46:56.895Z [INFO]  sink.file: creating file sink
2023-08-01T16:46:56.895Z [INFO]  sink.file: file sink configured: path=/home/vault/.vault-token mode=-rw-r-----
2023-08-01T16:46:56.895Z [INFO]  template.server: starting template server
2023-08-01T16:46:56.895Z [INFO] (runner) creating new runner (dry: false, once: false)
2023-08-01T16:46:56.895Z [INFO]  sink.server: starting sink server
2023-08-01T16:46:56.896Z [INFO]  auth.handler: starting auth handler
2023-08-01T16:46:56.896Z [INFO]  auth.handler: authenticating
2023-08-01T16:46:56.896Z [INFO] (runner) creating watcher
2023-08-01T16:46:56.923Z [INFO]  auth.handler: authentication successful, sending token to sinks
2023-08-01T16:46:56.924Z [INFO]  auth.handler: starting renewal process
2023-08-01T16:46:56.924Z [INFO]  template.server: template server received new token
2023-08-01T16:46:56.924Z [INFO] (runner) stopping
2023-08-01T16:46:56.924Z [INFO] (runner) creating new runner (dry: false, once: false)
2023-08-01T16:46:56.924Z [INFO]  sink.file: token written: path=/home/vault/.vault-token
2023-08-01T16:46:56.924Z [INFO] (runner) creating watcher
2023-08-01T16:46:56.924Z [INFO] (runner) starting
2023-08-01T16:46:56.924Z [INFO]  sink.server: sink server stopped
2023-08-01T16:46:56.924Z [INFO]  sinks finished, exiting
2023-08-01T16:46:56.930Z [INFO]  auth.handler: renewed auth token
2023-08-01T16:46:57.027Z [INFO] (runner) rendered "(dynamic)" => "/vault/secrets/password.txt"
2023-08-01T16:46:57.027Z [INFO] (runner) stopping
2023-08-01T16:46:57.027Z [INFO]  template.server: template server stopped
2023-08-01T16:46:57.027Z [INFO] (runner) received finish
2023-08-01T16:46:57.027Z [INFO]  auth.handler: shutdown triggered, stopping lifetime watcher
2023-08-01T16:46:57.027Z [INFO]  auth.handler: auth handler stopped
```
**Confirm secret was written to the main app container**
```
kubectl exec -ti nginx-deployment-585f4cdbf-z795l -c nginx -- cat /vault/secrets/password.txt
```
**Vault-agent-init:**
```
* initContainer created sink file at configured location path=/home/vault/.vault-token
* initContainer successfully authenticated vault K8s auth method
* initContainer successfully retrieved a token from vault 
* initContainer wrote a token to the sink location
* initContainer retrieved and rendered a secret to /vault/secrets/password.txt
* initContainer terminated after completing its job
```
**Nginx:**
```
* Main app container nginx state transitioned from pending to running
* Main app is now able to retrieve a token from the mount path /vault/secrets
```
**Vault-agent**
```
* sideCar container vault-agent transitioned from pending to running
* sideCar container is periodically renewing tokens
* sideCar container is periodically checking for updated secrets
```
**Misc Info:**

Log entry below indicates the vault injector cert is expired
```
E0720 23:17:06.739490       1 dispatcher.go:185] failed calling webhook "vault.hashicorp.com": failed to call webhook: Post "https://vault-agent-injector-svc.default.svc:443/mutate?timeout=30s": 
x509: certificate has expired or is not yet valid: current time 2023-07-20T23:17:06Z is after 2023-07-20T18:49:51Z
```
