
### Automated CI/CD Pipeline for Kubernetes
This project implements an automated CI/CD pipeline for a simple web application, adhering to key DevOps principles like security, traceability, and hands-off deployment. The solution uses Tekton Pipelines on a Kubernetes cluster to automate the build, test, and deployment process from a Git merge.

The primary goal is to ensure that once a developer merges code to the main branch, the application is automatically built and deployed to a DEV environment, with a subsequent automated promotion to a QA environment, all without manual intervention.

### 🚀 High-Level Workflow
The solution's workflow is triggered by a pull request merged to the main branch of the DevSecOps repository. This single event kicks off a chain of automated tasks.

- Code Commit: A developer merges new code to the main branch.
- Webhook Trigger: A GitHub webhook sends a signal to a Tekton EventListener in the DEV cluster.
- CI/CD Pipeline (DEV): The DEV pipeline is triggered, which:
   - Clones the repository.
   - Builds a new Docker image from the application code.
   - Deploys the application to the DEV Kubernetes namespace.
- Automated Promotion: Upon a successful DEV deployment, a final task in the DEV pipeline sends an HTTP request to the QA EventListener.
- CI/CD Pipeline (QA): The QA pipeline is triggered, which performs the same deployment steps, promoting the validated application to the QA environment.

### 📂 Repository Structure
The project is organized to clearly separate application code from infrastructure configuration, with access control enforced by CODEOWNERS and branch protection.
```text
.
├── Apps/
│   ├── src/                 # Application source code (e.g., JavaScript)
│   └── Dockerfile           # Instructions for building the container image
├── k8s/
│   ├── dev/                 # Kubernetes manifests for the DEV environment
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── qa/                  # Kubernetes manifests for the QA environment
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
├── .github/
│   └── CODEOWNERS           # Defines ownership and permissions for folders
├── Tests/                       # All working infrastructure-as-code YAML files
│   ├── cluster-roles.yaml
│   ├── service-accounts.yaml
│   ├── dev-deployment.yaml
│   ├── dev-service.yaml
│   ├── dev-ingress.yaml
│   ├── qa-deployment.yaml
│   ├── qa-service.yaml
│   ├── qa-ingress.yaml
│   ├── pipeline-tasks.yaml
│   ├── tekton-pipelines.yaml
│   └── tekton-triggers.yaml
|   |--- .......
└── docs/                    # Project documentation
    ├── workflow.md
    ├── onboarding.md
    ├── troubleshooting.md
    └── future-improvements.md
```
Working infrastructure-as-code files are placed under the **Tests** folder
### 📝 Project Documentation
The [docs](../docs) folder contains detailed documentation to explain the solution's core components and processes.

- [implementation.md](../Docs/implementation.md): A detailed explanation of the entire CI/CD process, from code commit to application deployment, including how namespace isolation and security measures are handled.

- [Onboarding.md](../Docs/onboarding.md): Steps for New Services: A step-by-step implementation guide of the entire setup, outlining the responsibilities of both development and infrastructure teams.

- [troubleshooting.md](../Docs/troubleshooting.md) A guide to diagnosing and resolving common issues that may arise within the setup and pipeline, with practical tips for debugging failed runs.

- [future-improvements.md](../Docs/future-improvements.ms): A discussion of potential enhancements, such as adopting a GitOps workflow with ArgoCD, and implementing a more robust testing and monitoring strategy.
