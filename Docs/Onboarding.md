# Onboarding a New Service: CI/CD Integration Walkthrough

Onboarding a new service requires a series of steps by the **infra** and **dev** teams to integrate the application into the existing CI/CD system.  
This is a detailed walkthrough of the entire process, from a fresh cluster to a working pipeline.

---
### 📂 Prerequisites
Before beginning, ensure the following tools are installed on your machine.

- **Docker**: Used to run containers and build images. Select the installation instructions according to your operating system: [Docker Desktop Installation](https://www.docker.com/products/docker-desktop/).
- **tkn**: The Tekton CLI for managing Tekton resources. Refer the installation instructions here: [tkn CLI Installation](https://tekton.dev/docs/cli/).
- **kubectl**: The command-line tool for interacting with your Kubernetes cluster.
- **Kubernetes Cluster**: Kubernetes cluster is required.
For various setup options, refer the [official documentation](https://kubernetes.io/docs/tasks/tools/)

Note: This implementation is based on a **Kind** cluster - a tool for running local Kubernetes clusters

## 1. 🛠 Initial Cluster Setup

The first steps are performed by the **infra team** to prepare the Kubernetes cluster for the new service.

### 📂 Create Namespaces
Create dedicated namespaces for the [dev](../Tests/namespace-dev.yaml) and [qa](../Tests/namespace-qa.yaml) environments:
```bash
kubectl create namespace dev
kubectl create namespace qa
```

### 🔑 Create Docker Registry Secrets

Create secrets in both [dev](../Tests/docker-registry-secret-dev.yaml) and [qa](../Tests/docker-registry-secret-qa.yaml) namespaces to allow Tekton to push and pull images from the registry:
```bash
# In both dev and qa namespaces
kubectl create secret docker-registry docker-registry-secret \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-docker-token> \
  --namespace=dev

kubectl create secret docker-registry docker-registry-secret \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-docker-token> \
  --namespace=qa
```


### 👤 Create Service Accounts

A dedicated service account, tekton-sa, is needed for the Tekton pipelines in both dev and qa namespaces:
```bash
kubectl create serviceaccount tekton-sa --namespace=dev
kubectl create serviceaccount tekton-sa --namespace=qa
```
Attach the Docker registry secret to the tekton-sa service account in both the namespaces for image pull and push
[tekton-sa-dev](../Tests/tekton-sa-dev.yaml)
[tekton-sa-qa](../Tests/tekton-sa-qa.yaml)
### 🔒 Create Roles and Bindings

Define the necessary permissions for tekton-sa:

[ClusterRole](../Tests/tekton-deployer-clusterrole.yaml) and [ClusterRoleBinding](../Tests/tekton-deployer-clusterrolebinding.yaml): Needed for cluster-wide permissions (e.g., creating/listing PipelineRuns, triggers,etc).

Bind the ClusterRole to the ServiceAccount in both namespaces

Role and RoleBinding: For namespace-specific permissions (e.g., creating and patching deployments).
Example Role and RoleBinding for managing deployments:
[tekton-triggers-role-dev](../Tests/tekton-triggers-role-dev.yaml)
[tekton-triggers-rolebinding-dev](../Tests/tekton-triggers-rolebinding-dev.yaml)
[tekton-triggers-role-qa](../Tests/tekton-triggers-role-qa.yaml)
[tekton-triggers-rolebinding-qa](../Tests/tekton-triggers-rolebinding-qa.yaml)

### 🌐 Install Ingress Controller

To expose the application, [install an Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) (e.g., Nginx Ingress Controller):
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

- [git-clone-task-dev](../Tests/git-clone-task-dev.yaml): Clones the application repository.

- [buildah-task-dev](../Tests/buildah-task-dev.yaml): Builds the Docker image and pushes it to the registry.

- [deploy-k8s-task-dev](../Tests/deploy-k8s-task-dev.yaml): Applies the new image to the Kubernetes deployment.

- [trigger-qa-pipeline-task-dev](../Tests/trigger-qa-pipeline-task-dev.yaml): Sends an HTTP POST request to the QA EventListener to trigger the next deployment.

### 🛠 Create Pipeline

Chain the tasks together into a pipeline:
[hello-world-pipeline-dev](../Tests/hello-world-pipeline-dev.yaml)

### 🔗 Create Triggering Components

Set up the EventListener, TriggerBinding, and TriggerTemplate to handle the GitHub webhook:

- [triggerbinding-dev](../Tests/triggerbinding-dev.yaml): Extracts parameters (e.g., branch name, commit hash).

- [triggertemplate-dev](../Tests/triggertemplate-dev.yaml): Defines the PipelineRun, passing parameters from the binding.

- [eventListener-dev](../Tests/eventlistener-dev.yaml): Listens for events, filters push events to the main branch, and validates the secret.

3. ### 🧪 Tekton Pipeline and Trigger Setup (QA)

The QA environment mirrors DEV for consistency.

🧩 Create Same Pipelines and Tasks

Create identical manifests for QA:

- [git-clone-task-dev](../Tests/git-clone-task-qa.yaml)

- [buildah-task-qa](../Tests/buildah-task-qa.yaml)

- [deploy-k8s-task-qa](../Tests/deploy-k8s-task-qa.yaml)

📡 Create QA EventListener

Configure separate [EventListener](../Tests/eventlistener-qa.yaml), [TriggerBinding](../Tests/triggerbinding-qa.yaml), and [TriggerTemplate](triggertemplate-qa.yaml) for QA:

This EventListener is triggered by the [trigger-qa-pipeline-task](../Tests/trigger-qa-pipeline-task-dev.yaml) in the DEV pipeline.

4. ### 🌍 GitHub and Webhook Integration

This final step links the Git repository to the Kubernetes cluster.

🔑 Create GitHub Webhook Secret

Generate a random token and create the [github-webhook-secret](../Tests/github-webhook-secret-dev.yaml):
```bash
# Generate a random string
openssl rand -hex 20

# Create the secret in Kubernetes
kubectl create secret generic github-webhook-secret \
  --from-literal=secretToken=<your-generated-token> \
  --namespace=dev

```
### 🌐 Expose EventListener

Use ngrok to create a public URL for the EventListener service:
```bash
kubectl port-forward svc/<eventlistener-svc-dev> 8080:8080 -n dev
ngrok http 8080
```
### 🔗 Create GitHub Webhook

In your GitHub repository settings, configure a new webhook:

- Payload URL: Public URL provided by ngrok.

- Secret Token: Paste the generated secret token.

- Events: Select only Pushes to trigger the pipeline.
