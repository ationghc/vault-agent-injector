# vault-agent-injector
The Vault Agent Injector alters pod to include 2 Vault containers, that'll retreive secrets from vault and store the secrets in a shared volume, where the main application container can retreive the secret. 

This demonstration configures the vault agent from the outside in. Meaning, we're going to create the application pod in away that it is expected to fail. So that we can address the issues and have a better understanding of vault-agent dependencies. 

Step 1.
Clone the nginx-deployment git repo 
- git clone https://github.com/ationghc/vault-agent-injector.git

Step 2.
Create an nginx-deployment with 3 replicas using the nginx-deployment.yaml supplied in this repo.
- cd vault-agent-injector folder
- kubectl apply -f nginx-deployment.yaml

Step 3.
Check the status of the nginx-deployment:
- kubectl get deploy nginx-deployment
  
  NAME               READY   UP-TO-DATE   AVAILABLE   AGE
  nginx-deployment   0/3     3            0           70m

The output from the above command, shows that none of the nginx pods are running. You can also drill down more on the deployment by running kubectl describe on the nginx-deployment object.

Step 4.
List only the nginx application pods. Use the -l argument to specify only the pods that have the label apps=nginx
- kubectl get po -l app=nginx 


NAME                               READY   STATUS     RESTARTS   AGE
nginx-deployment-585f4cdbf-529qg   0/2     Init:0/1   0          3m34s
nginx-deployment-585f4cdbf-rrxdk   0/2     Init:0/1   0          3m34s
nginx-deployment-585f4cdbf-svwhg   0/2     Init:0/1   0          3m34s


Determine why the pods are stuck in an initializing state, by running kubectl describe on the nginx application pod and then check for the name and state of each container in the pod. 
In the output of the kubectl describe command, there are two sections that reference the Init Containers: and regular Containers: Below are the names of the containers defined in the pod spec.

- kubectl describe po nginx-deployment-585f4cdbf-529qg

The nginx-deployment pod should have 3 containers

1. Init Container: vault-agent-init container runs until it completes its job. Which is to retreive a secret from vault and store it to a mount point defined on a shared volume. This job needs to occur before any of the other containers are started. In fact, the other containers cannot start unless the init container completes succesfully. Which is to successfully retreive a secret from vault.
2. Main App Container: nginx container consumes the secret that the vault-agent-init container stores to its defined mount point
3. SideCar Container: vault-agent container runs through out the lifecycle of the pod and continously checks for updated secrets, whereas the init container terminates as soon as it retreives the initial secret from vault.  


Below are sections from the kubectl describe deployment command that describe the state of each container. Notice only the vault-agent-init container is running. 

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

Step 5. 
Check the logs for the vault-agent-init container to understand why the other containers are stuck initializing and not running.
Usually when you run kubectl logs, it'll output the logs for the main container. Since the main container didn't start, specify the initContainer vault-agent-init by using the -c flag.

-kubectl logs nginx-deployment-585f4cdbf-rrxdk  -c vault-agent-init

2023-07-20T02:56:28.224Z [ERROR] auth.handler: error authenticating:
  error=
  | Error making API request.
  | 
  | URL: PUT http://vault.default.svc:8200/v1/auth/kubernetes/login
  | Code: 400. Errors:
  | 
  | * invalid role name "test-app"
   backoff=3m48.61s


The output from the kubectl logs command tells us alot about the vault-agent-injector and the reason for the failure




