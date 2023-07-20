# vault-agent-injector
The Vault Agent Injector alters pod to include 2 Vault containers, that'll retreive secrets from vault and store the secrets in a shared volume, where the main application container can retreive the secret. 

This demonstration configures the vault agent from the outside in. Basically, we're going to create the application pod in away that it is expected to fail. So that we can address the issues and have a better understanding of vault-agent dependencies. 


List only the nginx application pods and check then confirm status of the nginx-deployment pods. Use the -l argument to specify only the pods that have the apps=nginx label
kubectl get po -l app=nginx 


NAME                               READY   STATUS     RESTARTS   AGE
nginx-deployment-585f4cdbf-529qg   0/2     Init:0/1   0          3m34s
nginx-deployment-585f4cdbf-rrxdk   0/2     Init:0/1   0          3m34s
nginx-deployment-585f4cdbf-svwhg   0/2     Init:0/1   0          3m34s


Determine why the pods are stuck in an initializing state, by running kubectl describe on the nginx application pod and then check for the name and state of each container in the pod. There are two sections that reference the Init Containers: and regular Containers:

- Init Container: vault-agent-init
- Main App Container: nginx
- SideCar Container: vault-agent

kubectl describe po nginx-deployment-585f4cdbf-529qg
