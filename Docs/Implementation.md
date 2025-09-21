# Automated Deployment Workflow for JavaScript Web Application

This workflow automates the deployment of a simple JavaScript web application to a **Kubernetes cluster** using **Tekton Pipelines**, designed with **security**, **traceability**, and **automation** as core principles.

- This setup is implemented on a local, single-node Kind cluster, serving as a proving ground for the CI/CD logic.
- The underlying pipeline logic is platform-agnostic, meaning the same process can be seamlessly adapted for any other Kubernetes cluster, be it in the cloud (e.g., EKS, GKE, AKS) or on-premise.
- By leveraging the same Tekton pipelines, this solution can easily integrate with more reliable and scalable platform-specific tools, such as cloud-native registries, managed ingress controllers, and robust monitoring solutions.
- Ultimately, this project serves as a comprehensive and reproducible blueprint for a robust and secure CI/CD workflow, suitable for a wide range of production environments.

---

## 📂 GitHub Implementation

The workflow starts in the **DevSecOps GitHub repository**, where **security** and **access controls** are strictly enforced:

### 🔒 Branch Protection
- The **main** branch is protected to prevent direct commits.  
- All changes must be made through **pull requests**, ensuring **code review** and **approval**.

### 👥 Codeowners
- The `.github/CODEOWNERS` file enforces the **principle of least privilege**:
  - The **infra-team** group owns the `k8s` directory (both `dev` and `qa` sub-folders). Only members of this group can review and approve pull requests modifying infrastructure manifests.
  - The **developers** group owns the `Apps` directory, managing application source code.  
  - This separation ensures developers **cannot directly change infrastructure**, and infrastructure changes are reviewed by the appropriate team.

### 🌐 Webhook & Security
- A **webhook** is configured on the **main** branch to trigger the CI/CD pipeline upon a merge.  
- The webhook sends a signal to the **EventListener** in the `Dev` namespace of the cluster.  
- Currently, the EventListener service is exposed **locally** and accessed externally using **ngrok**, which provides a public URL registered in GitHub’s webhook configuration.  
- A **shared secret token** authenticates requests, ensuring only valid payloads from GitHub trigger the pipeline.

---

## 🛠 DEV Namespace Implementation

Once a pull request is merged, the webhook triggers the automated process in the **DEV environment**:

### 📡 Event Listener & Tekton Trigger
- The webhook payload is received by a **Tekton EventListener** in the `DEV` namespace.  
- It checks if the event originated from the **main branch**, validates the **secret token**, and triggers a **Tekton PipelineRun** using the **TriggerBinding** and **TriggerTemplate**.

### 🚀 Pipeline Execution
- The DEV pipeline executes a series of tasks:
  1. **Clone the repository** for the latest code and manifests.
  2. **Build a Docker image** from the application code, tagging it with the commit hash for **traceability**. Push the image to a **private Docker registry**.
  3. **Apply Kubernetes manifests** from `k8s/dev`. Patch the deployment manifest with the new image tag, triggering a **rolling update** of the “Hello World” application pods.

### 🧱 Namespace Isolation
- All DEV deployment resources (pods, services, ingresses) are contained within the `dev` namespace.  
- This prevents resource contention and accidental cross-environment changes.

---

## 🛠 QA Namespace Implementation

After a successful DEV deployment, the workflow **progresses to QA**:

### 🔗 Automated Trigger
- The final task in the DEV pipeline sends an **HTTP signal** to a separate **QA EventListener**, indicating DEV deployment success.

### 🔗 QA Pipeline Execution
- The QA EventListener triggers a new **PipelineRun** for QA:
  - Clones the repository.
  - Builds or pulls the image.
  - Deploys using manifests from `k8s/qa`.

### 🔗 Consistency and Isolation
- **Same pipeline logic** is used for DEV and QA for consistency.  
- Separate namespaces (`dev` and `qa`) ensure isolation, preventing accidental cross-environment interference.

---

## 🌍 Ingress and ngrok

To access the deployed application externally:

### 🔗 Ingress
- Each namespace (`dev` and `qa`) has an **Ingress object** exposing the application’s Service.  
- The Ingress routes external traffic to the correct service based on the **hostname**.  
- For local testing, expose the Ingress service using port-forwarding

### 🔗 ngrok

- ngrok provides a public URL that tunnels traffic from the internet to the local cluster.
- This enables GitHub webhooks to send payloads securely to the Tekton EventListener during development and testing.
- For a production environment, a more reliable solution such as a cloud load balancer or a publicly accessible Ingress controller with a DNS record would be used.
