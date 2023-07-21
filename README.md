# vault-agent-injector
The Vault Agent Injector alters pods to include 2 Vault containers, that'll retreive secrets from vault and store the secrets in a shared volume, allowing the main application container access to retreive the vault secret. 

We're going to create the application pod in away that it is expected to fail. Then we'll address the issues as they arise.  

**Step 1.**
Clone the nginx-deployment git repo 
  CMD: git clone https://github.com/ationghc/vault-agent-injector.git

**Step 2.**
Create an nginx-deployment with 3 replicas using the nginx-deployment.yaml supplied in this repo.
 CMD: cd vault-agent-injector; kubectl apply -f nginx-deployment.yaml

**Step 3.**
Check the status of the nginx-deployment:
  CMD: kubectl get deploy nginx-deployment

**Output of k get deploy**
  NAME               READY   UP-TO-DATE   AVAILABLE   AGE
  nginx-deployment   0/3     3            0           70m

Output above shows that none of the nginx pods are running. Next step execute kubectl describe deploy cmd on the nginx-deployment object, to understand state of deployment. 

**Step 4.**
Check output of kubectl describe deploy cmd for the fields listed below.
  CMD: kubectl describe deploy nginx-deployment
  
- containers name field
- replicas field
- image field
- serviceAccount field
- conditions field
- namespace field
- annotations field
- deployment events

The output below from the kubectl describe deploy cmd didn't state why the pods weren't running. But, it did provide the state of the deployment.

**Output of k describe deployment**
Name:                   nginx-deployment
Namespace:              default
Replicas:               3 desired | 3 updated | 3 total | 0 available | 3 unavailable
Pod Template:
  Annotations:      vault.hashicorp.com/agent-inject: true
                    vault.hashicorp.com/agent-inject-secret-password.txt: test/secret/super_secret
                    vault.hashicorp.com/role: test-app
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

Since we've checked the state of the deployment, the next step is to check the state of the pods.
**Step 5.**

List only the nginx application pods. Use the -l argument to specify only the pods that have the label apps=nginx. 
  CMD: kubectl get po -l app=nginx 

                                        PODS Are Stuck In An Initialization Status
NAME                               READY   STATUS     RESTARTS   AGE
nginx-deployment-585f4cdbf-529qg   0/2     Init:0/1   0          3m34s
nginx-deployment-585f4cdbf-rrxdk   0/2     Init:0/1   0          3m34s
nginx-deployment-585f4cdbf-svwhg   0/2     Init:0/1   0          3m34s

  - Pod status
    - Init:0/1 means zero out of one initContainers have completed so far.
    - PodInitializing or Running means the Pod has already finished executing the Init Containers.

Next step determine why the pods are stuck in an initializing state.

**Step 6.**
Execute kubectl describe on an nginx application pod
  - Check for the name and state of each container in the pod. 
  - In the output of the kubectl describe command, there are two fields that identify the containers of a pod.
  - Init Containers
  - Container

**The nginx-deployment pods should have 3 containers defined in the pod spec**

**Init Container:** 
  - vault-agent-init: container runs until it completes its job.
    - Successfully retreives a secret from vault and store the secret in a mount point defined on a shared volume.
    - This job occurs before any other containers are started.
    - In fact, usually the other containers cannot start unless the init container completes succesfully first.
    
**Main App Container:** 
  - nginx: Main app container consumes the secret that the vault-agent-init initContainer stored to a mount point in the pod spec.

**SideCar Container:** 
  - vault-agent: container runs through out the lifecycle of the pod and continously checks for updated secrets.
    - whereas the init container terminates after it executes and retreives the initial secret from vault successfully.  

  CMD: kubectl describe po nginx-deployment-585f4cdbf-529qg

The kubectl describe po command
  - Describes the state of the pod and each container.
  - Notice only the vault-agent-init container is running.  

Below are two fields that define the pods containers from the output of the kubectl describe po command.
  - Init Containers
  - Containers
  - The containers field defines the Main app container nginx and the sideCar container vault-agent.
  - The Main app container and sideCar container depend on the initContainer vault-agent-init to complete its job successfully before they can start.
   
  
Init Containers:
  vault-agent-init:
    State:          Running
      Started:      Wed, 19 Jul 2023 17:31:58 -0700
    Ready:          False

Containers:
  nginx:
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False

vault-agent:
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False

The output of the describe command above didn't show the cause of the pods stuck initializing. But it did provide the state of the pod and its defined containers.

The next step is to check the logs of one of the application pod initContainer vault-agent-init.

**Step 7.**
Check the logs for the vault-agent-init container to understand why the other containers are stuck initializing and not running.
  - Usually when you run kubectl logs, it'll output the logs for the main container.
  - Since the main container didn't start, specify the running initContainer vault-agent-init by using the -c flag.

  CMD: kubectl logs nginx-deployment-585f4cdbf-rrxdk  -c vault-agent-init

**Example of log errors in the application pods initContainer vault-agent-init.**

2023-07-20T23:56:59.952Z [ERROR] auth.handler: error authenticating:
  error=
  | Error making API request.
  | 
  | URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
  | Code: 400. Errors:
  | 
  | * invalid role name "test-app"
   backoff=4m9.83s

This error means that the role defined in the deployment annotation is incorrect or doesn't exist.

The output from the kubectl logs command tells us alot about the vault-agent-injector and the reason for the failure.
  - The error shows the vault agent is attempting to login @ a VAULT URL http://vault.default.svc:8200/v1/auth/kubernetes/login
  - The error shows a kubernetes auth role test-app is invalid.
  - The error shows the kubernetes authentication path is set to auth/kubernetes.
  - The error shows the fqdn of the vault server http://vault.default.svc:8200 or service fqdn.
    - This value can be updated by editing the env AGENT_INJECT_VAULT_ADDR in the vault-agent-injector pod spec.
  
The next step is to check the kubernetes auth role.

**Step 8.**
  - exec into vault pod
    CMD: kubectl exec -it vault-0 /bin/sh

List the Vaults k8s auth roles at the auth/kubernetes/role endpoint.
  CMD: vault list auth/kubernetes/role

The output from the list command above, doesn't include the test-app role.
   - The test-app role will need to be created

The next step create the kubernetes auth test-app role.

**Step 9.**
  CMD: vault write auth/kubernetes/role/test-app bound_service_account_names=vault-auth bound_service_account_namespaces=default

The next step is to check the logs from the initContainer vault-agent-init

**Step 10.**
  CMD: kubectl logs nginx-deployment-585f4cdbf-rrxdk  -c vault-agent-init

**Example of new ERROR in application pod initContainer vault-agent-init:**
  2023-07-21T00:33:45.274Z [WARN] (view) vault.read(test/secret/super_secret): vault.read(test/secret/super_secret): Error making API request.

URL: GET http://vault.default.svc:8200/v1/test/secret/super_secret
Code: 403. Errors:

* 1 error occurred:
	* permission denied


The new error initially seems to be related to the token not having a policy that allows a read action on the kv secret path test/secret/super_secret. 
  - But we're encountering this error
    - Because we didn't enable the secret engine or create a secret at that path.

Dealing with ERRORS: 
 - x509: certificate has expired 

W0720 22:40:40.320475       1 dispatcher.go:181] Failed calling webhook, failing open vault.hashicorp.com: failed calling webhook "vault.hashicorp.com": failed to call webhook: Post "https://vault-agent-injector-svc.default.svc:443/mutate?timeout=30s": x509: certificate has expired or is not yet valid: current time 2023-07-20T22:40:40Z is after 2023-07-20T18:49:51Z
E0720 22:40:40.320514       1 dispatcher.go:185] failed calling webhook "vault.hashicorp.com": failed to call webhook: Post "https://vault-agent-injector-svc.default.svc:443/mutate?timeout=30s": x509: certificate has expired or is not yet valid: current time 2023-07-20T22:40:40Z is after 2023-07-20T18:49:51Z

If you encounter any x509 errors you can either specify the SSL details or skip-tls.
  - To skip tls add an annotation to the deployment object nginx-deployment.
    - You can either update the YAML file with the new annotation and re-POST it to the API or use the kubectl patch command. 
      CMD: kubectl apply -f nginx-deployment.yaml

  Use k8s verb PATCH instead of updating YAML:
  CMD: kubectl patch deployment nginx-deployment -p '{"spec": {"template":{"metadata":{"annotations":{"vault.hashicorp.com/tls-skip-verify":"true"}}}} }'

