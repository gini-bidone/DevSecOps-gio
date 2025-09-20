# 🧰 CI/CD Pipeline Troubleshooting Guide

Use the following procedures to diagnose and resolve common issues within the CI/CD pipeline.

## 🔎 General Troubleshooting Steps

### 1. ✅ Check GitHub Actions / Webhooks
- Navigate to your **repository settings → Webhooks**.  
- Check the **Recent Deliveries** section to confirm:
  - The webhook was **triggered**.  
  - The payload was successfully delivered (**HTTP status code 200**).  
- If there’s an error:
  - Review the **payload**.  
  - Verify the **public URL** provided by **ngrok**.

### 2. 📋 Inspect Tekton PipelineRun Logs
- Use the **tkn CLI tool** or **Tekton Dashboard** to view pipeline run statuses and logs.  
- List recent pipeline runs:
  ```bash
  tkn pipelinerun list
  ```
- View detailed logs for a specific run:
  ```bash
  tkn pipelinerun logs <pipelinerun-name>
    ```
- Check logs for each task (git-clone-task, buildah-task, etc.) to locate the failure point.

---
### ⚠️ Common Issues and Resolutions
🐳 buildah-task failed to push the image

Reason:

- The service account (tekton-sa) may lack permissions.

- The Docker registry secret could be incorrect or expired.

Resolution:

- Verify that tekton-sa has the necessary image-pull and push permissions.

- Check the docker-registry secret to ensure credentials are correct and valid.

- Confirm the image tag is valid and free of illegal characters.

---
### 🚢 deploy-k8s-task failed to deploy the application

Reason:

- Kubernetes manifest errors.

- Insufficient permissions.

- Issues with the Kubernetes cluster itself.

Resolution:

- Validate the syntax of the deployment manifest:
```bash
kubectl apply --dry-run=client -f k8s/dev/deployment.yaml
```
- Ensure tekton-sa has correct ClusterRole permissions (get, list, create, patch, and update deployments/resources).

- Check the cluster’s status and resource availability.
