This K8s session assumes that vault is running and kubernetes auth is already enabled.

# vault-agent-injector
The Vault Agent Injector alters pods to include 2 Vault containers, that'll retreive secrets from vault and store the secrets in a shared volume, allowing the main application container access to retreive the vault secret. 

We're going to create the application pod in away that it is expected to fail. Then we'll address issues as they arise.  

**Kubernetes Objectives:**
- Distinguish between kubernetes pod container types
  - Main App Container
  - initContainer
  - ContainerSidecar
- Create a kubernetes deployment
- Check Container and pod status
- Investigate container logs
- Annotate a running deployment
- mutatingwebhookconfiguration - step pending
- Mutate pod to include initContainer and sideCar container
- kubectl cli: patch,get,logs and describe

**Vault Objectives:**
- Deploy an application that is able to retreive secrets from vault, using vault-k8s vault-injector
- Create a vault policy 
- Enable kv secrets engine
- Create a secret in kv secrets engine path
- Create a kubernetes auth method role


**Step 1.**
Clone the nginx-deployment git repo 
  
  CMD: 
  git clone https://github.com/ationghc/vault-agent-injector.git

**Step 2.**
Create an nginx-deployment with 1 replica using the nginx-deployment.yaml supplied in this repo.
 
 CMD: 
 cd vault-agent-injector; kubectl apply -f nginx-deployment.yaml

**Step 3.**
Check the status of the nginx-deployment:
  
  CMD: 
  kubectl get deploy nginx-deployment

**Output of kubectl get deploy**

  ```
  NAME               READY   UP-TO-DATE   AVAILABLE   AGE
  nginx-deployment   1/1     1            1          70m
 ```
The output above shows the nginx-deployment pod is running. But without the initContainer vault-agent-init and sideCar container vault-agent. 

The annotation below triggers vault agent injector to inject containers into pods.
   - vault.hashicorp.com/agent-inject: 'true'

The next step is to check if the deployment has the correct annotations set. 

**Step 4.**
Check deployment for annotations, you can use any one of the commands listed below to view a resources annotation.

CMD:
kubectl annotate --list=true pod nginx-deployment-585f4cdbf-lqklw

kubectl get deployment nginx-deployment -o json | jq -r .spec.template.metadata.annotations

kubectl describe deployment nginx-deployment and check for annotations

kubectl get po nginx-deployment-7b9477ddd8-v7z5x -o yaml


```
{
  "deployment.kubernetes.io/revision": "1",
  "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"apps/v1\",\"kind\":\"Deployment\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"nginx\"},\"name\":\"nginx-deployment\",\"namespace\":\"default\"},\"spec\":{\"replicas\":1,\"selector\":{\"matchLabels\":{\"app\":\"nginx\"}},\"template\":{\"metadata\":{\"annotations\":null,\"labels\":{\"app\":\"nginx\"}},\"spec\":{\"containers\":[{\"image\":\"nginx:1.14.2\",\"name\":\"nginx\",\"ports\":[{\"containerPort\":80}]}],\"serviceAccountName\":\"vault-auth\"}}}}\n"
}
```
The output above is missing annotations for vault.hashicorp.com/agent-inject:true. 
As a result the pod is being deploymed with only the application container nginx.

The next step is to update the deployment nginx-deployment with the proper annotations
**Step 5.**

You can update the deployment object nginx-deployment with the patch command, but it is not the recommended approach.
- The kubectl patch command takes a partial Kubernetes resource spec as input and merges it into the object. It is essentially an update command for kubernetes resources.

Patch CMD for single annotation:

kubectl patch deployment nginx-deployment -p '{"spec": {"template":{"metadata":{"annotations":{"vault.hashicorp.com/agent-inject":"true"}}}} }'

Patch CMD for multiple annotations:

kubectl patch deployment nginx-deployment -p '{"spec": {"template":{"metadata":{"annotations":{"vault.hashicorp.com/tls-skip-verify":"true","vault.hashicorp.com/agent-inject":"true","vault.hashicorp.com/agent-inject-secret-password.txt":"test/secret/super_secret"}}}} }' 

**Recommended Approach:**
kubectl replace command is the recommended approach. You can edit your current YAML in place or create a new YAML with your latest changes and then post it to the API with the recreate subcommand. Kubernetes will recreate the resource with the new values you defined in the YAML. 

The next step is to run the kubectl replace command, which is listed below to update the nginx-deployment with the vault-agent-injector annotations.

CMD:
kubectl  replace -f nginx-deployment-annotated.yaml

Next step is to confirm the annotations were applied to the deployment object.
**Step 6.**

CMD:
kubectl get deployment nginx-deployment -o json | jq -r .spec.template.metadata.annotations

The output from the command above shows the annotations were updated/patched successfully.
```
{
  "vault.hashicorp.com/agent-inject": "true",
  "vault.hashicorp.com/agent-inject-secret-password.txt": "test/secret/super_secret",
  "vault.hashicorp.com/role": "test-app"
}
```

Since the annotations were applied then the containers should be injected. 

The next step is to execute kubectl get pods -l app=nginx to confirm if the containers were injected into the application pod.
**Step 7.**

CMD: 
kubectl get pods -l app=nginx

The output from the command above shows that the vault-agent-injector webhook server injected the vault agent containers into the application pod.
Because the ready column shows `0/2` container ready. 
```
NAME                               READY   STATUS     RESTARTS   AGE
nginx-deployment-585f4cdbf-6p4pw   0/2     Init:0/1   0          4s
```
```
 - Pod status
    - Init:0/1 means none of the initContainers have completed so far.
    - PodInitializing or Running means the Pod has already finished executing the Init Containers.
```

Next step determine why the pods are stuck in an initializing state.
The next step is to check the logs:
**Step 8.**
If the containers are injected but not running, then you'll need to check the logs
- vault-agent-injector logs
  CMD:
  kubectl logs vault-agent-injector-8496df644f-kfnbb -f

- application pod initContainer logs
  CMD:
  kubectl logs nginx-deployment-585f4cdbf-6p4pw -c vault-agent-init -f

- kubernetes API logs
  CMD: kubectl logs kube-apiserver-vault-test -n kube-system | grep inject

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
The next step is to check if the kubernetes auth role test-app is configured
**Step 9.**

**The error above means that the role defined in the deployment annotation is incorrect or doesn't exist.**

The output from the kubectl logs command tells us alot about the vault-agent-injector and the reason for the failure.
 ```
  - The error shows the vault agent is attempting to login @ a VAULT URL http://vault.default.svc:8200/v1/auth/kubernetes/login
  - The error shows a kubernetes auth role test-app is invalid.
  - The error shows the kubernetes authentication path is set to auth/kubernetes.
  - The error shows the fqdn of the vault server http://vault.default.svc:8200 or vault-agent-injector.svc service fqdn.
    - This value can be updated by editing the env AGENT_INJECT_VAULT_ADDR in the vault-agent-injector pod spec.
  ```
The next step is to check the kubernetes config and auth role.

**Step 10.**
  - exec into vault pod

    CMD:
    kubectl exec -it vault-0 /bin/sh

List the Vaults k8s auth roles at the auth/kubernetes/role endpoint, to determine if the test-app role exist.

  CMD:
  vault read auth/kuberntes/config
  
  CMD: 
  vault list auth/kubernetes/role

The output from the list command above, doesn't include the test-app role.
  ```
   - The test-app role will need to be created
  ```
The next step create the kubernetes auth test-app role.

**Step 11.**

  CMD: 
  vault write auth/kubernetes/role/test-app bound_service_account_names=vault-auth bound_service_account_namespaces=default token_policies=test-app
```
The next step is to check the logs from the initContainer vault-agent-init
```

**Step 12.**

  CMD: 
  kubectl logs nginx-deployment-585f4cdbf-rrxdk  -c vault-agent-init

**Example of new ERROR in application pod initContainer vault-agent-init:**

```
2023-07-21T00:33:45.274Z [WARN] (view) vault.read(test/secret/super_secret): vault.read(test/secret/super_secret): Error making API request.

URL: GET http://vault.default.svc:8200/v1/test/secret/super_secret
Code: 403. Errors:

* 1 error occurred:
	* permission denied
```

The new error initially seems to be related to the token not having a policy that allows a read action on the kv secret path test/secret/super_secret. 
```
  - But we're encountering this error
    - Because we didn't enable the secret engine or create a secret at that path.
```

The next step is to enable the kv secrets engine and create a secret at path test/secret/super_secret 

**Step 13.**
Enable kv secrets engine at path test
 
 CMD: 
 vault secrets enable -path=test kv
 
```
The next step is to create a secret at path
```
**Step 14.**
 
 CMD: 
 vault kv put test/secret/super_secret password=p@ssword
 
```
Next step make sure you're able to retreive the secret.
```
**Step 15.**
Retrieve kv secret super_secret

CMD: 
vault kv get test/secret/super_secret

```
Next step is to create a vault policy to allow read access to the kv secret.
```
**Step 16.**
Create a vault policy named test-app with read permissions on kv secrets path "test/secret/super_secret"
```
CMD:

vault policy write test-app - <<EOF

path "test/secret/super_secret" {

capabilities = ["read"]

}

EOF
```

**Step 17.**
Read vault policy test-app.

CMD: 
vault policy read test-app
```
Next step check the status of the pods 
```

**Step 18.**
At this point the pods initContainer should've successfully retreived a token from vault and the main app container nginx and sideCar container vault-agent should be running now.

CMD: 
kubectl get po -l app=nginx
```
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-68f94459b-6fx4z   2/2     Running   0          114m
```

**IGNORE INFORMATION BELOW:**

The next step is to execute kubectl describe deploy cmd on the nginx-deployment object, to understand state of deployment. 

**Step 4.**
Check output of kubectl describe deploy cmd for the fields listed below.
  
  CMD: 
  kubectl describe deploy nginx-deployment
  ```
- containers name field
- replicas field
- image field
- serviceAccount field
- conditions field
- namespace field
- annotations field
- deployment events
```

The output below from the kubectl describe deploy cmd didn't state why the pods weren't running. But, it did provide the state of the deployment.

**Output of kubectl describe deployment**
```
Name:                   nginx-deployment
Namespace:              default
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
Pod Template:
  Annotations:      
  Service Account:  vault-auth
  Containers:
   nginx:
    
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    False   ProgressDeadlineExceeded
NewReplicaSet:   nginx-deployment-585f4cdbf (3/3 replicas created)
Events:          <none>
```
Since we've checked the state of the deployment, the next step is to check the state of the pods.

**Step 5.**
List only the nginx application pods. Use the -l argument to specify only the pods that have the label apps=nginx. 
  
  CMD: 
  kubectl get po -l app=nginx 

        **POD STATUS is running and the main app container has a STATUS of READY**
```
vault-agent-injector % kubectl get po -l app=nginx           
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7b9477ddd8-vg7kc   1/1     Running   0          3s



```
  - Pod status
    - Init:0/1 means none of the initContainers have completed so far.
    - PodInitializing or Running means the Pod has already finished executing the Init Containers.

Next step determine why the pods are stuck in an initializing state.

**Step 6.**
Execute kubectl describe on an nginx application pod
```
  - Check for the name and state of each container in the pod. 
  - In the output of the kubectl describe command, there are two fields that identify the containers of a pod.
  - Init Containers
  - Container
```
**The nginx-deployment pods should have 3 containers defined in the pod spec**

**Init Container:** 
```
  - vault-agent-init: container runs until it completes its job.
    - Successfully retreives a secret from vault and store the secret in a mount point defined on a shared volume.
    - This job occurs before any other containers are started.
    - In fact, usually the other containers cannot start unless the init container completes succesfully first.
```
**Main App Container:** 
  - nginx:
    ```
    Main app container consumes the secret that the vault-agent-init initContainer stored to a mount point in the pod spec.
    ```
**SideCar Container:** 
  - vault-agent:
    ```
    container runs through out the lifecycle of the pod and continously checks for updated secrets.
    	- whereas the init container terminates after it executes and retreives the initial secret from vault successfully.  
    ```

  CMD: 
  kubectl describe po nginx-deployment-585f4cdbf-529qg

The kubectl describe po command
```
  - Describes the state of the pod and each container.
  - Notice only the vault-agent-init container is running.  
```

Below are two fields that define the pods containers from the output of the kubectl describe po command.
 ```
  - Init Containers
  - Containers
  - The containers field defines the Main app container nginx and the sideCar container vault-agent.
  - The Main app container and sideCar container depend on the initContainer vault-agent-init to complete its job successfully before they can start.
```
   
```
Init Containers:
  vault-agent-init:
    State:          Running
      Started:      Wed, 19 Jul 2023 17:31:58 -0700
    Ready:          False
```
```
Containers:
  nginx:
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
```
```
vault-agent:
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
```
The output of the describe command above didn't show the cause of the pods stuck initializing. But it did provide the state of the pod and state of its defined containers.

The next step is to check the logs of one of the application pod initContainer vault-agent-init.

**Step 7.**
Check the logs for the vault-agent-init container to understand why the other containers are stuck initializing and not running.
```
  - Usually when you run kubectl logs, it'll output the logs for the main container.
  - Since the main container didn't start, specify the running initContainer vault-agent-init by using the -c flag.
```
  CMD: 
  kubectl logs nginx-deployment-585f4cdbf-rrxdk  -c vault-agent-init

** 2 Examples of possible log errors in the application pods initContainer vault-agent-init.**

```
2023-07-21T17:00:03.034Z [INFO]  agent.auth.handler: authenticating
2023-07-21T17:00:03.037Z [ERROR] agent.auth.handler: error authenticating:
  error=
  | Error making API request.
  |
  | URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
  | Code: 403. Errors:
  |
  | * permission denied
   backoff=3m41.79s
```

```
2023-07-20T23:56:59.952Z [ERROR] auth.handler: error authenticating:
  error=
  | Error making API request.
  | 
  | URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
  | Code: 400. Errors:
  | 
  | * invalid role name "test-app"
   backoff=4m9.83s
```


**Misc Insight:**
Multiple errors can generate 403 errors, its up to you to determine where the errors are coming from. Sometimes you'll need to check the logs of the kube-system api pod for more insight on webhook issues.

CMD:
kubectl logs kube-apiserver-vault-test -n kube-system | grep injec
```
   E0720 23:17:06.739490       1 dispatcher.go:185] failed calling webhook "vault.hashicorp.com": failed to call webhook: Post "https://vault-agent-injector-svc.default.svc:443/mutate?timeout=30s": x509: certificate has expired or is not yet valid: current time 2023-07-20T23:17:06Z is after 2023-07-20T18:49:51Z
```
**If you encounter any x509 errors you can either specify the SSL details or skip-tls.**
 ```
  - To skip tls add an annotation to the deployment object nginx-deployment.
    - You have two options update the YAML file with the new annotation and re-POST it to the API or use the kubectl patch command. 
```
Update YAML Option:
```
CMD:
kubectl apply -f nginx-deployment.yaml
```
**Use k8s verb PATCH instead of updating YAML manually:**
  ```
  CMD: 
  kubectl patch deployment nginx-deployment -p '{"spec": {"template":{"metadata":{"annotations":{"vault.hashicorp.com/tls-skip-verify":"true"}}}} }'
 ```

**Quick Summary on vault-agent-injector troubleshooting steps. This should help diagnose causes of injection failure of initContainer and sideCar container into an application pod via kubernetes annotations.**
- Ensure the vault auth kubernetes method works first
  `cd /run/secrets/kubernetes.io/serviceaccount ; vault write auth/kubernetes/login jwt=@token role=test`
- Ensure that the kubernetes_host is reachable from the application pod
- Ensure that the vault-agent-injector service address is reachable from the application pod on port 8080
- Ensure that the vault-agent-injector endpoints are reachable on port 8080
- Ensure the kubernetes role has the correct serviceAccount and namespace
- Ensure application pod has the serviceAccount and is in the namespace defined in the kubernetes auth role
- Ensure there are no issues accessing the tokenreview API
- Inspect application pod mounted token, you can use a 3rd party site called jwt.io
- If a customer has aggregated logging like splunk, have them check their logs for
 	- kubernetes auth accessor id
   	- ServiceAccount
    	- The strings: unauthorized, x509, vault.hashicorp.com, "permission denied", mutate, injector


Misc:
**Command to select names of containers in pods**

CMD: 
kubectl get po nginx-deployment-585f4cdbf-fttp8 -o json | jq '.spec.containers[].name'
kubectl get po nginx-deployment-585f4cdbf-fttp8 -o json | jq '.spec.initContainers[].name'


**Logs aren't telling the full story:**
Sometimes the vault-agent-injector and the kubernetes-api log_levels are set to INFO and as a result they do not provide adequate information.
The next step would be to increase log verbosity on API server and vault-agent-injector. 

**Change vault-agent-injector log_level:**
 - Edit the deployment vault-agent-injector and change env $AGENT_INJECT_LOG_LEVEL from INFO to DEBUG
   - CMD kubectl edit deployment vault-agent-injector 

**Change Kubernetes API log_level:**
- Create SA clusterrolebinding for clusterrole edit-debug-flags-v
- Create SA default a TOKEN CMD: kubectl create token default
- Update log_level with curl CMD: curl -k -X PUT -d '4' https://127.0.0.1:50119/debug/flags/v --header "Authorization: Bearer $TOKEN"
	- Your cluster port is probably different, check with CMD: kubect cluster-info

