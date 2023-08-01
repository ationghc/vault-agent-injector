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

**The next step is to check container logs:**
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
```
kubectl exec vault-0 -- vault read auth/kubernetes/config
kubectl exec vault-0 -- vault list auth/kubernetes/role
```
**Step 11.**

**Create Vault Kubernetes Auth role test-app:**
```
vault write auth/kubernetes/role/test-app bound_service_account_names=vault bound_service_account_namespaces=default token_policies=test-app
```
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

**Create a vault policy with read access for kv secret path test/secret:**
```
vault policy write test-app - <<EOF
path "test/secret" {
capabilities = ["read"]
}
EOF
```
**Step 14.**

**Check pod status:**
```
kubectl get po -l app=nginx
```
```
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-68f94459b-6fx4z   2/2     Running   0          114m
```
**Step 15.**

**Check initContainer status:**
```
kubectl get po nginx-deployment-585f4cdbf-wjt7g -o json | jq '.status.initContainerStatuses'

"containerID": "docker://18c25eac4d55aa374419f18b8b38442ebcf617ad73fb4f9521fea414e539a4a2",
    "image": "hashicorp/vault:1.12.0",
    "imageID": "docker-pullable://hashicorp/vault@sha256:0723b1e86ef211de2f63aba10bc540956a771a6ae202f776c92bb7dfd32e6267",
    "lastState": {},
    "name": "vault-agent-init",
    "ready": true,
    "restartCount": 0,
    "state": {
      "terminated": {
        "containerID": "docker://18c25eac4d55aa374419f18b8b38442ebcf617ad73fb4f9521fea414e539a4a2",
        "exitCode": 0,
        "finishedAt": "2023-08-01T16:46:57Z",
        "reason": "Completed",
        "startedAt": "2023-08-01T16:46:56Z"
```
**Check sideCar and main app container status:**
```
kubectl get po nginx-deployment-585f4cdbf-wjt7g -o json | jq '.status.containerStatuses'

containerID": "docker://272f29527ed89aa26fbc74e9508490eddff0c9dbe096e3dcd83425747bbd0fd4",
    "image": "nginx:1.14.2",
    "imageID": "docker-pullable://nginx@sha256:f7988fb6c02e0ce69257d9bd9cf37ae20a60f1df7563c3a2a6abe24160306b8d",
    "lastState": {},
    "name": "nginx",
    "ready": true,
    "restartCount": 0,
    "started": true,
    "state": {
      "running": {
        "startedAt": "2023-08-01T16:46:57Z"
   -------------------------------------------
    "containerID": "docker://4b7dec247ad0ad190381805fcb857c3119b5dbd54fb7e3067254a616c9a721a5",
    "image": "hashicorp/vault:1.12.0",
    "imageID": "docker-pullable://hashicorp/vault@sha256:0723b1e86ef211de2f63aba10bc540956a771a6ae202f776c92bb7dfd32e6267",
    "lastState": {},
    "name": "vault-agent",
    "ready": true,
    "restartCount": 0,
    "started": true,
    "state": {
      "running": {
        "startedAt": "2023-08-01T16:46:57Z"
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
Log entry below indicates the application pod serviceAccount isn't scoped to a k8s auth role:
```
2023-08-01T21:49:09.160Z [ERROR] auth.handler: error authenticating:
  error=
  | Error making API request.
  | 
  | URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
  | Code: 403. Errors:
  | 
  | * service account name not authorized
```
Log entry below indicates the application pod namespace isn't scoped to a k8s auth role:
```
2023-08-01T22:02:46.410Z [ERROR] auth.handler: error authenticating:
  error=
  | Error making API request.
  | 
  | URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
  | Code: 403. Errors:
  | 
  | * namespace not authorized
```
Log entry below indicates the vault injector cert is expired
```
E0720 23:17:06.739490       1 dispatcher.go:185] failed calling webhook "vault.hashicorp.com": failed to call webhook: Post "https://vault-agent-injector-svc.default.svc:443/mutate?timeout=30s": 
x509: certificate has expired or is not yet valid: current time 2023-07-20T23:17:06Z is after 2023-07-20T18:49:51Z
```
