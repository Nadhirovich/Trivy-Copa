# Trivy-Copa


# GitHub Actions Workflow: Patch and Use Local Image

This GitHub Actions workflow automates the process of scanning, patching, and running a container image locally, while also saving the patched image for future use. Below is an explanation of each job and its functionality.

---

## **Workflow Overview**

The workflow consists of **three sequential jobs**:
1. **Scan Vulnerabilities**: Pull the unpatched image, generate a vulnerability report using Trivy, and save the report as an artifact.
2. **Patch the Image**: Use Copacetic to patch the pulled image based on the vulnerability report and save the patched image as an artifact.
3. **Run Web Service**: Load the patched image locally on the runner and use it to start a web service. Finally, upload the patched image as a downloadable artifact for future use.

---

## **Workflow Details**

### **Job 1: Scan Vulnerabilities**
- **Purpose**: Pull the unpatched image, scan it for vulnerabilities using Trivy, and generate a report.
- **Steps**:
  - Pull the **`nginx:1.21.6`** image.
  - Run a vulnerability scan using the command:
    ```bash
    trivy image --vul-type os --ignore-unfixed nginx:1.21.6 | grep "Total"
    ```
    This extracts the "Total" number of vulnerabilities found for easy visibility in the workflow logs.
  - Generate a full **JSON vulnerability report** with Trivy.
  - Save the report as a workflow artifact named `vulnerability-report` for use in the next job.

---

### **Job 2: Patch the Image**
- **Purpose**: Patch the vulnerabilities identified in the report using Copacetic.
- **Steps**:
  - Download the `vulnerability-report` artifact from Job 1.
  - Use **Copacetic** to patch the image (`nginx:1.21.6`) based on the vulnerability report:
    ```yaml
    uses: project-copacetic/copa-action@v1.0.0
    with:
      image: nginx:1.21.6
      image-report: 'report.json'
      patched-tag: nginx:patched
    ```
  - Save the patched image locally as a `.tar` file using:
    ```bash
    docker save nginx:patched -o patched-image.tar
    ```
  - Upload the patched image as a workflow artifact named `patched-image` for use in the next job.

---

### **Job 3: Run Web Service**
- **Purpose**: Load the patched image locally and use it to run a web service.
- **Steps**:
  - Download the `patched-image` artifact from Job 2.
  - Load the patched image into Docker on the **runner's local environment** using:
    ```bash
    docker load < patched-image.tar
    ```
    *Note*: "Locally" refers to the GitHub Actions runner where the workflow is being executed.
  - Start a web service container using the patched image:
    ```bash
    docker run -d --name patched-nginx -p 8080:80 nginx:patched
    ```
    This exposes the container on port **8080** of the runner.
  - Upload the patched image as a workflow artifact named `patched-image-final` for future use.

---

## **Artifacts Created**
1. `vulnerability-report`: Contains the Trivy-generated JSON vulnerability report.
2. `patched-image`: Contains the patched image saved as a `.tar` file after processing with Copacetic.
3. `patched-image-final`: Contains the final patched image saved for future downloading and use.

---

## **Usage Notes**
- To access the artifacts, navigate to the **"Actions" tab** of your GitHub repository, select the workflow run, and download artifacts from the **"Artifacts" section**.
- This workflow is ideal for automating container image patching and testing within GitHub-hosted runners or self-hosted environments.

