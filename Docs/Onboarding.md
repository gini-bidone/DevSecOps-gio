# Onboarding a New Service: CI/CD Integration Walkthrough

Onboarding a new service requires a series of steps by the **infra** and **dev** teams to integrate the application into the existing CI/CD system.  
This is a detailed walkthrough of the entire process, from a fresh cluster to a working pipeline.

---

## 1. 🛠 Initial Cluster Setup

The first steps are performed by the **infra team** to prepare the Kubernetes cluster for the new service.

### 📂 Create Namespaces
Create dedicated namespaces for the `dev` and `qa` environments:
```bash
kubectl create namespace dev
kubectl create namespace qa
```

### 🔑 Create Docker Registry Secrets

Create secrets in both namespaces to allow Tekton to push and pull images from the registry:
```bash
# In both dev and qa namespaces
kubectl create secret docker-registry docker-registry-secret \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --namespace=dev

kubectl create secret docker-registry docker-registry-secret \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --namespace=qa
```


### 👤 Create Service Accounts

A dedicated service account, tekton-sa, is needed for the Tekton pipelines:
```bash
kubectl create serviceaccount tekton-sa --namespace=dev
kubectl create serviceaccount tekton-sa --namespace=qa
```

### 🔒 Create Roles and Bindings

Define the necessary permissions for tekton-sa:

ClusterRole and ClusterRoleBinding: Needed for cluster-wide permissions (e.g., creating/listing PipelineRuns, triggers).

Role and RoleBinding: For namespace-specific permissions (e.g., creating and patching deployments).

```bash
# Example ClusterRole that allows creating and listing PipelineRuns
# (can be reused for both namespaces)
tekton-deployer-clusterrole.yaml
```

Bind the ClusterRole to the ServiceAccount in both namespaces:
```bash
kubectl create clusterrolebinding tekton-sa-binding \
  --clusterrole=tekton-cluster-role \
  --serviceaccount=dev:tekton-sa

kubectl create clusterrolebinding tekton-sa-binding \
  --clusterrole=tekton-cluster-role \
  --serviceaccount=qa:tekton-sa
```
Example Role and RoleBinding for managing deployments:
```yaml
# trigger-binding-dev.yaml
```
```bash
kubectl create rolebinding tekton-sa-deployer-binding \
  --role=tekton-deployer-role \
  --serviceaccount=dev:tekton-sa

kubectl create rolebinding tekton-sa-deployer-binding \
  --role=tekton-deployer-role \
  --serviceaccount=qa:tekton-sa

```
Finally, attach the Docker registry secret to the tekton-sa service account:
```yaml
tekton-sa.yaml
```

### 🌐 Install Ingress Controller

To expose the application, install an Ingress controller (e.g., Nginx Ingress Controller):
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/kind/deploy.yaml
```


Note: The installation command may vary based on your cluster type.

### 📦 Install Tekton Components
Install Tekton Pipelines and Triggers:

```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
```

2. ### 🚀 Tekton Pipeline and Trigger Setup (DEV)

The infra team sets up the pipeline for the DEV environment.

### 🧩 Create Tasks

- git-clone-task.yaml: Clones the application repository.

- buildah-task.yaml: Builds the Docker image and pushes it to the registry.

- deploy-k8s-task.yaml: Applies the new image to the Kubernetes deployment.

- trigger-qa-pipeline-task.yaml: Sends an HTTP POST request to the QA EventListener to trigger the next deployment.

### 🛠 Create Pipeline

Chain the tasks together into a pipeline:
```yaml
# hello-world-pipeline.yaml
```
### 🔗 Create Triggering Components

Set up the EventListener, TriggerBinding, and TriggerTemplate to handle the GitHub webhook:

- triggerBinding.yaml: Extracts parameters (e.g., branch name, commit hash).

- triggerTemplate.yaml: Defines the PipelineRun, passing parameters from the binding.

- eventListener.yaml: Listens for events, filters push events to the main branch, and validates the secret.

3. ### 🧪 Tekton Pipeline and Trigger Setup (QA)

The QA environment mirrors DEV for consistency.

🧩 Create Same Pipelines and Tasks

Create identical manifests for QA:

- git-clone-task-qa.yaml

- buildah-build-qa.yaml

- deploy-k8s-qa.yaml

📡 Create QA EventListener

Configure separate EventListener, TriggerBinding, and TriggerTemplate for QA:

- eventlistener-qa.yaml

- triggerbinding-qa.yaml

- triggertemplate-qa.yaml

This EventListener is triggered by the trigger-qa-pipeline-task in the DEV pipeline.

4. ### 🌍 GitHub and Webhook Integration

This final step links the Git repository to the Kubernetes cluster.

🔑 Create GitHub Webhook Secret

Generate a random token and create the secret:
```bash
# Generate a random string
openssl rand -base64 32

# Create the secret in Kubernetes
kubectl create secret generic github-webhook-secret \
  --from-literal=secretToken=<your-generated-token> \
  --namespace=dev

```
### 🌐 Expose EventListener

Use ngrok to create a public URL for the EventListener service:
```bash
kubectl port-forward svc/el-my-dev-listener 8080:8080 -n dev
ngrok http 8080
```
### 🔗 Create GitHub Webhook

In your GitHub repository settings, configure a new webhook:

- Payload URL: Public URL provided by ngrok.

- Secret Token: Paste the generated secret token.

- Events: Select only Pushes to trigger the pipeline.