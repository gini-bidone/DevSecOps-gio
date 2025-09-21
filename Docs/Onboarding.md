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

---

## 1. 🛠 Initial Cluster Setup

The first steps are performed by the **infra team** to prepare the Kubernetes cluster for the new service.

### 📂 Create Namespaces
Create dedicated namespaces for the [dev](../Tests/namespace-dev.yaml) and [qa](../Tests/namespace-qa.yaml) environments:
```bash
kubectl create namespace dev
kubectl create namespace qa
```
### 📂 Create Persistent Volume Claims for both dev and qa namespaces:
A PVC is needed in each namespace to provide a shared workspace for the pipeline tasks. The git-clone-task will clone the repository into this volume, and subsequent tasks will use it to access the code.
- [dev-pvc](../pvc-dev.yaml)
- [qa-pvc](../pvc-qa.yaml)

Note: The PVC is then attached to the PipelineRun via the workspace field, allowing all tasks within the pipeline to share the same persistent storage.

### 🔑 Create Docker Registry Secrets

Create secrets in both the namespaces to allow Tekton to push and pull images from the registry:
- [docker-registry-secret-dev](../Tests/docker-registry-secret-dev.yaml)
- [docker-registry-secret-qa](../Tests/docker-registry-secret-qa.yaml)
  
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
Attach the Docker registry secret to the tekton-sa service account in both the namespaces for image pull and push:
- [tekton-sa-dev](../Tests/tekton-sa-dev.yaml)
- [tekton-sa-qa](../Tests/tekton-sa-qa.yaml)

### 🔒 Create Roles and Bindings

Define the necessary permissions for tekton-sa:

[ClusterRole](../Tests/tekton-deployer-clusterrole.yaml) and [ClusterRoleBinding](../Tests/tekton-deployer-clusterrolebinding.yaml): Needed for cluster-wide permissions (e.g., creating/listing PipelineRuns, triggers,etc).

Bind the ClusterRole to the ServiceAccount in both namespaces

Role and RoleBinding: For namespace-specific permissions (e.g., creating and patching deployments).
Example Role and RoleBinding for managing deployments:
- [tekton-triggers-role-dev](../Tests/tekton-triggers-role-dev.yaml)
- [tekton-triggers-rolebinding-dev](../Tests/tekton-triggers-rolebinding-dev.yaml)
- [tekton-triggers-role-qa](../Tests/tekton-triggers-role-qa.yaml)
- [tekton-triggers-rolebinding-qa](../Tests/tekton-triggers-rolebinding-qa.yaml)
---

## 2. 📦 Install Tekton Components

The infra team sets up the tekton components for the cluster.

### 🔗 Install Tekton Pipelines and Triggers:

```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
```
---
## 3. 🧪 Tekton Pipeline and Trigger Setup (Dev Namespace)
The dev pipeline is a high-level orchestration of tasks that ensures a new version of application is built, deployed to dev, and then promoted to QA in a predictable and hands-off manner.

### 🔗 Create Tasks
Each task is a modular step in the workflow, with dependencies that ensure they run in the correct sequence.

- [git-clone-task-dev](../Tests/git-clone-task-dev.yaml): Clones the application repository.

- [buildah-task-dev](../Tests/buildah-task-dev.yaml): Builds the Docker image and pushes it to the registry.

- [deploy-k8s-task-dev](../Tests/deploy-k8s-task-dev.yaml): Applies the new image to the Dev deployment

- [trigger-qa-pipeline-task-dev](../Tests/trigger-qa-pipeline-task-dev.yaml): Sends an HTTP POST request to the QA EventListener to trigger the next deployment.

### 🔗 Create Pipeline

Chain the tasks together into a pipeline. Chaining tasks into a Tekton pipeline creates a multi-step workflow for the application's CI/CD process

- [hello-world-pipeline-dev](../Tests/hello-world-pipeline-dev.yaml)

### 🔗 Create Triggering Components

Set up the EventListener, TriggerBinding, and TriggerTemplate to handle the GitHub webhook:

- [triggerbinding-dev](../Tests/triggerbinding-dev.yaml): Extracts parameters (e.g., branch name, commit hash).

- [triggertemplate-dev](../Tests/triggertemplate-dev.yaml): Defines the PipelineRun, passing parameters from the binding.

- [eventListener-dev](../Tests/eventlistener-dev.yaml): Listens for events, filters push events to the main branch, and validates the secret.

  ----

## 4. 🧪 Tekton Pipeline and Trigger Setup (QA Namespace)

The QA environment mirrors DEV for consistency.

### 🔗 Create Same Pipelines and Tasks

Create identical manifests for QA:

- [git-clone-task-dev](../Tests/git-clone-task-qa.yaml) - Clones the application repository.

- [buildah-task-qa](../Tests/buildah-task-qa.yaml) -  Builds the Docker image and pushes it to the registry.

- [deploy-k8s-task-qa](../Tests/deploy-k8s-task-qa.yaml) - Applies the new image to the QA deployment

- [hello-world-pipeline-qa](../Tests/hello-world-pipeline-qa.yaml) - Chain the tasks together and invoke them in order

### 🔗 Create Triggering Components

Configure separate [EventListener](../Tests/eventlistener-qa.yaml), [TriggerBinding](../Tests/triggerbinding-qa.yaml), and [TriggerTemplate](triggertemplate-qa.yaml) for QA:

This EventListener is triggered by the [trigger-qa-pipeline-task](../Tests/trigger-qa-pipeline-task-dev.yaml) in the DEV pipeline.

---

## 5. 🌍 GitHub and Webhook Integration

This next step links the Git repository to the Kubernetes cluster.

### 🔑 Create GitHub Webhook Secret

Generate a random token and create the [github-webhook-secret](../Tests/github-webhook-secret-dev.yaml):
```bash
# Generate a random string
openssl rand -hex 20

# Create the secret in Kubernetes
kubectl create secret generic github-webhook-secret \
  --from-literal=secretToken=<your-generated-token> \
  --namespace=dev

```
### 🔗 Create GitHub Webhook

In the GitHub repository settings, configure a new webhook:

- **Payload URL**: Public URL of Dev event-listener service or its ingress URL that GitHub sends webhook data to when a repository event occurs.

- **Secret Token**: Paste the generated secret token.

- **Events**: Select only Pushes to trigger the pipeline.
---
## 6. 🧪 Additional Cluster Setup for Local Environment
   
For the workflow to function in a local environment, such as a Kind cluster, some extra steps are required to expose services and enable external access. While these manual steps are necessary for local testing, they can be replaced by more reliable platform-native tools in enterprise or cloud environments.

### 🌐 Expose EventListener

Use ngrok to create a public URL for the EventListener service:
```bash
kubectl port-forward svc/<eventlistener-svc-dev> 8080:8080 -n dev
ngrok http 8080
```
This URL can be used as the Payload URL while setting up github webhook.

Note: For a production environment, a more reliable solution such as a cloud load balancer or a publicly accessible Ingress controller with a DNS record would be used.

### 🌐 Install Ingress Controller

To expose the application, [install an Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) (e.g., Nginx Ingress Controller):
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/kind/deploy.yaml
```
Check if the ingress-controller pods are up and running:
```bash
kubectl get pods -n ingress-nginx
```
Note: The installation command may vary based on the cluster type.

### 🔗 Create the application service

The Service is a fundamental Kubernetes resource that provides a stable, internal IP address and DNS name for your application. It acts as an abstraction layer, routing traffic to a set of pods without needing to know their individual, dynamic IP addresses.
- [service-dev](../service-dev.yaml)
- [service-qa](../service-qa.yaml)
  
### 🔗 Create the Ingress Manifest
An Ingress resource is crucial for exposing the application to external traffic in a Kubernetes cluster. It works with an Ingress Controller (like NGINX) to route external requests to the correct service within the cluster. Once the Ingress and its associated Service are in place, they act as a stable entry point for the application.
- [ingress-dev](../ingress-dev.yaml)
- [ingress-qa](../ingress-qa.yaml)

Note: The Service and Ingress are part of the initial setup and do not need to be created or updated with every new application deployment. They provide a stable, unchanging endpoint, allowing the CI/CD pipeline to focus solely on updating the application's Deployment manifest with the new image.

 ### 🌐 Viewing the Application
The final step is to use the Ingress URL to verify that the application is running correctly.

For local testing:
- Port-forward the Nginx Ingress Controller to make it accessible on the local machine through a local port
```bash
kubectl port-forward svc/ingress-nginx-controller 80:80 -n ingress-nginx
```
- Find the Ingress URL: The URL is the host name defined in the Ingress manifest of each namespaces, such as dev.custom-url.com or qa.custom-url.com.
- To access the application from a local machine, the /etc/hosts file must be updated to resolve the custom Ingress hostnames to the localhost address (127.0.0.1).
```bash
# Example /etc/hosts entries
127.0.0.1    dev.custom-url.com
127.0.0.1    qa.custom-url.com
```
- Now, open a web browser and navigate to both hostnames:
    - Go to `http://dev.custom-url.com` to see the application in the dev environment.
    - Go to `http://qa.custom-url.com` to see the application in the qa environment.

The Nginx Ingress Controller inspects the hostname in the browser's request and routes it to the correct application service based on the rules defined in the Ingress manifests.

Note: For enterprise environments: In production or enterprise setups, DNS records are typically configured to point the Ingress hostnames to the public IP address of the Ingress controller or a load balancer, providing a more reliable way for users to access the application.

