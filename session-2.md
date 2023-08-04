This is session #2 of learning Kubernetes, while debugging vault-agent-injector.

<h2>Prerequisites</h2> 

* A Vault pod
  * Initalized & unsealed
  * KV secrets engine enabled
  * A Vault policy that references the KV secrets path 
  * k8s auth method enabled
  * A role configured for the k8s auth method
* An application pod

* Prior to starting lab, spin up a new Minikube cluster
`minikube start -p lab`
* Ensure that a Vault license has been exported to the shell in the `VAULT_LICENSE` variable
* Run the setup.sh [script](https://github.com/jeremyaranas-hashicorp/minikube/blob/main/vault/vault-raft/lab/vault_agent_injector/setup.sh)



**Step 6**

Update the deployment with vault-agent-injector annotations

`kubectl  replace -f nginx-deployment-annotated.yaml`

**Step 7**

Confirm the annotations were applied to the deployment object

`kubectl get deployment nginx-deployment -o json | jq -r .spec.template.metadata.annotations`

The output from the command shows the annotations were updated/patched successfully

```
{
  "vault.hashicorp.com/agent-inject": "true",
  "vault.hashicorp.com/agent-inject-secret-password.txt": "test/secret",
  "vault.hashicorp.com/role": "test-app"
}
```

**Step 8**

Confirm if the containers were injected into the nginx application pod

`kubectl get pods -l app=nginx`

```
NAME                               READY   STATUS     RESTARTS   AGE
<nginx-deployment>   0/2     Init:0/1   0          4s
```

Pod status:
  - Init:0/1 means none of the initContainers have completed so far.
  - _PodInitializing_ or _Running_ means the Pod has already finished executing the Init Containers.

**Step 9**

Check container logs:
* Check logs of vault-agent-injector pod
  * `kubectl logs <vault-agent-injector-pod> -f`
* Check logs of initContainer
  * `kubectl logs <nginx-deployment> -c vault-agent-init -f`
* Check logs of K8s API
  * `kubectl logs kube-apiserver-vault-test -n kube-system | grep inject`

The vault-agent-injector is reporting a 400 error invalid role name "test-app"

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

**Step 10**

Check K8s auth role

`kubectl exec vault-0 -- vault list auth/kubernetes/role`

**Step 11**

Update deployment to use the role that exists in Vault for the k8s auth method

Edit the _nginx-deployment-annotated.yaml_ file and update the `vault.hashicorp.com/role` annotation to use the `test-role` role

**Step 12** 

Redeploy the Ngnix deployment after updating annotations  

`kubectl replace -f nginx-deployment-annotated.yaml` 

**Step 13** 

Check logs from the Vault agent init container in the application pod

`kubectl logs -f <nginx-deployment> vault-agent-init`

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

**Step 14** 

Check k8s auth role for _bound_service_account_namespaces_

`kubectl exec vault-0 -- vault read auth/kubernetes/role/test-role`

```
bound_service_account_names         [test-sa]
bound_service_account_namespaces    [vault]
```

**Step 15** 

Check namespace that application pod is running in 

`kubectl get pods <nginx_pod> -o json | jq .metadata.namespace`

We see that the appplication pod is running in the _default_ namespace but the `bound_service_account_namespaces` is set to _vault_

**Step 16** 

Update `bound_service_account_namespaces` to _default_

`kubectl exec vault-0 -- vault write auth/kubernetes/role/test-role bound_service_account_namespaces=default`

**Step 17** 

Confirm that namespace has been updated 

`kubectl exec vault-0 -- vault read auth/kubernetes/role/test-role`

**Step 18** 

Check logs from the Vault agent init container in the application pod

`kubectl logs -f <nginx-deployment> vault-agent-init`

```
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
```

**Step 19** 

Check application pod for service account

`kubectl get pods <nginx-pod> -o json | jq .spec.serviceAccount`

We see that the application pod is using the _vault_ service account

**Step 20**

Check k8s auth role config for `bound_service_account_names`

`kubectl exec vault-0 -- vault read auth/kubernetes/role/test-role`

```
bound_service_account_names         [test-sa]
bound_service_account_namespaces    [default]
```

We see that the _bound_service_account_names_ for the role is set to _test-sa_ but the application pod is using the _vault_ service account

**Step 21**

Update `bound_service_account_namespaces` for the k8s auth role

`kubectl exec vault-0 -- vault write auth/kubernetes/role/test-role bound_service_account_names=vault`

**Step 22**

Confirm that service account has been updated 

`kubectl exec vault-0 -- vault read auth/kubernetes/role/test-role`

**Step 23**

Check logs from the Vault agent init container in the application pod

`kubectl logs -f <nginx-deployment> vault-agent-init`

```
kubectl logs <nginx-deployment>  -c vault-agent-init
2023-08-01T19:27:06.607Z [WARN] (view) vault.read(test/secret): vault.read(test/secret): Error making API request.

URL: GET http://vault.default.svc:8200/v1/test/secret
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

**Step 24**

Check policy in Vault

`kubectl exec -ti vault-0 -- vault policy list`

Check k8s auth role to see what policies are set

`kubectl exec vault-0 -- vault read auth/kubernetes/role/test-role`

Write k8s auth role to reference _test-policy_ policy

`kubectl exec vault-0 -- vault write auth/kubernetes/role/test-role policies=test-policy`

**Step 25**

Check pod status

```
kubectl get po -l app=nginx
```

```
NAME                               READY   STATUS    RESTARTS   AGE
<nginx-deployment>   2/2     Running   0          114m
```

**Step 26**

Re-Check logs of initContainer:

```
kubectl logs <nginx-deployment> -c vault-agent-init

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

**Step 27**

Confirm secret was written to the main app container

```
kubectl exec -ti <nginx-deployment> -c nginx -- cat /vault/secrets/password.txt
```

<h3>Things to Note</h3>

vault-agent-init

```
* initContainer created sink file at configured location path=/home/vault/.vault-token
* initContainer successfully authenticated vault K8s auth method
* initContainer successfully retrieved a token from vault 
* initContainer wrote a token to the sink location
* initContainer retrieved and rendered a secret to /vault/secrets/password.txt
* initContainer terminated after completing its job
```

nginx:

```
* Main app container nginx state transitioned from pending to running
* Main app is now able to retrieve a token from the mount path /vault/secrets
```

vault-agent

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
