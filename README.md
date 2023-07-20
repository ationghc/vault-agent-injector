# vault-agent-injector
The Vault Agent Injector alters pod to include 2 Vault containers, that'll retreive secrets from vault and store the secrets in a shared volume, where the main application container can retreive the secret. 

This demonstration configures the vault agent from the outside in. Meaning, we're going to create the application pod in away that it is expected to fail. So that we can address the issues and have a better understanding of vault-agent dependencies. 

Check the status of the nginx-deployment:
- kubectl get deploy nginx-deployment
  
  NAME               READY   UP-TO-DATE   AVAILABLE   AGE
  nginx-deployment   0/3     3            0           70m

The output of the above command, shows that none of the nginx pods are running. 

Next step

List only the nginx application pods. Use the -l argument to specify only the pods that have the label apps=nginx
- kubectl get po -l app=nginx 


NAME                               READY   STATUS     RESTARTS   AGE
nginx-deployment-585f4cdbf-529qg   0/2     Init:0/1   0          3m34s
nginx-deployment-585f4cdbf-rrxdk   0/2     Init:0/1   0          3m34s
nginx-deployment-585f4cdbf-svwhg   0/2     Init:0/1   0          3m34s


Determine why the pods are stuck in an initializing state, by running kubectl describe on the nginx application pod and then check for the name and state of each container in the pod. In the output of the kubectl describe command, there are two sections that reference the Init Containers: and regular Containers: Below are the names of the containers defined in the pod spec.

- kubectl describe po nginx-deployment-585f4cdbf-529qg



1. Init Container: vault-agent-init
2. Main App Container: nginx
3. SideCar Container: vault-agent



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



