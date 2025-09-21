## 🗺️ Architecture Diagram

This diagram outlines a secure, automated CI/CD pipeline that uses Tekton to deploy applications to a Kubernetes cluster, from Git push to final application delivery.
The entire process is initiated by a single event: a code push to the main branch of a Git repository. This event triggers a chain of automated tasks that build, test, and deploy the application without any manual intervention.  The workflow is designed to be fully automated, ensuring a consistent and reliable process for every new code change.

<img width="2829" height="1570" alt="image" src="https://github.com/user-attachments/assets/d73ebe58-bd07-457e-9828-5745b22061e2" />

- Trigger: A pre-configured GitHub Webhook sends an event payload to a Tekton EventListener inside the Kubernetes cluster.

- Dev Pipeline: The EventListener triggers a Tekton PipelineRun in the Dev namespace. This pipeline's tasks perform the following steps:

   - Clone the repository's source code.

   - Build a new container image from the code and push it to a Container Registry.

  - Deploy the application by updating the Kubernetes Deployment with the new image tag.

- Automated Promotion: Upon successful completion of the Dev deployment, the pipeline's final task sends a secure HTTP request to a separate EventListener in the QA namespace.

- QA Pipeline: This new event triggers a similar PipelineRun in the QA namespace. This pipeline pulls the same validated image from the registry and deploys it to the QA environment.

- Final Access: Users can access the deployed application through a Kubernetes Ingress, which routes traffic to the correct application service in either the Dev or QA namespace.
