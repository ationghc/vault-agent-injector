In this series we'll learn about Kubernetes concepts, by debugging Hashicorps Vault Agent Injector Webhook server. 

Most of vault-agent-injector dependencies are missing. As a result we'll encounter typical errors that clients would face, when deploying vault-agent-injector in their K8s infrastructure. 

**Session prerequisites:**
*  K8s Minikube Cluster
*  Vault installed and unsealed
*  K8s Vault Auth Method enabled

**What is vault-agent-injector?**

* Vault-Agent-Injector is a Webhook server
* Alters a pods spec to include 2 Vault agent containers
* The containers job is to retrieve secrets from a vault server
* Store the secrets in a pods shared volume with main app container
* Main app container accesses the secret in the shared volume

**Kubernetes Objectives:**
- Distinguish between kubernetes container types
  - Main App Container
  - initContainer
  - Sidecar
- Create a Kubernetes deployment object
- Check deployment state
- Check container and pod status
- Query Kubernetes API for pod container names and annotations
- Update a deployment object annotations using kubectl patch and kubectl replace commands
- Check container logs 

**Step 1.**

**Clone the nginx-deployment repo** 
  
CLI: Clones GitHub repo to your local directory:
```
git clone https://github.com/ationghc/vault-agent-injector.git
```
**Step 2.**

**Create an nginx-deployment with 1 replica using the nginx-deployment.yaml supplied in repo.**

CLI: Reads YAML file and POST it to the K8s API to create a deployment object:
```
kubectl apply -f nginx-deployment.yaml
```
**Step 3.**

**Check the status of the nginx-deployment:**

CLI: Retrieves deployment state:
```
kubectl get deploy nginx-deployment
```
**Output shows nginx-deployment is running with 1 replica and appears to be healthy .**

```
  NAME               READY   UP-TO-DATE   AVAILABLE   AGE
  nginx-deployment   1/1     1            1          70m
```
**Step 4.**

**The next step is to confirm if the containers were injected into the nginx application pod.**

CLI: List pods with label app=nginx
```
kubectl get pods -l app=nginx
```
**The injection didn't occur as expected, output shows the nginx-deployment pod has 1 of 1 container running.**
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7b9477ddd8-n9tc5   1/1     Running   0          83m
```
**Causes of a container injection to not occur in an application pod:**
* Connectivity issue between vault-agent-injector and K8s API
* Lack of vault-agent-injector annotations
* Vault-agent-injector not running
* Vault-agent-injector service is not reachable

**The annotation below triggers vault agent injector to inject vault agent containers into a pod:**
   - vault.hashicorp.com/agent-inject: 'true' 

**Step 5.**

**The next step is to check K8s deployment for the required annotation.**.

CLI: Queries API for POD or deployment annotations:
```
kubectl get deployment nginx-deployment -o json | jq -r .spec.template.metadata.annotations
```
**Output shows no annotations were defined.**
```
null
```
**ENDS HERE**
**Misc Information:**

**Other kubectl commands that list a deployments annotations**
```
kubectl annotate --list=true pod nginx-deployment-585f4cdbf-lqklw
kubectl get deployment nginx-deployment -o json | jq -r .spec.template.metadata.annotations
kubectl describe deployment nginx-deployment and check for annotations
kubectl get po nginx-deployment-7b9477ddd8-v7z5x -o yaml
```
**You can also update a deployment resource with the patch command:** 
* Adhoc updates on the fly 
* The kubectl patch command takes a partial Kubernetes resource spec as input and merges it into the object. 
* It's an update command for a Kubernetes resources.
* Last patch wins update model
* Different operations can simultaneously patch a resource at the same time, which will cause an update to be lost
* Patching is not a recommended approach, because it doesn't support atomic write operations

**Update a deployments annotations using kubectl patch subcommand:**
```
CLI: Update a single annotation:
kubectl patch deployment nginx-deployment -p '{"spec": {"template":{"metadata":{"annotations":{"vault.hashicorp.com/agent-inject":"true"}}}} }'

CLI: Update multiple annotations:
kubectl patch deployment nginx-deployment -p '{"spec": {"template":{"metadata":{"annotations":{"vault.hashicorp.com/tls-skip-verify":"true","vault.hashicorp.com/agent-inject":"true","vault.hashicorp.com/agent-inject-secret-password.txt":"test/secret/super_secret"}}}} }' 
```

