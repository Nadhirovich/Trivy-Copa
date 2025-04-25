# Trivy-Copa mini project!

# GitHub Actions Workflow: Patch, Validate, and Use Local Image

This GitHub Actions workflow automates the process of scanning, patching, validating, and saving a container image for future use. Below is an explanation of each job and its functionality.

---

## **Workflow Overview**

The workflow consists of **three sequential jobs**:
1. **Scan Vulnerabilities**: Pull the unpatched image, generate a vulnerability report using Trivy, and save the report as an artifact.
2. **Patch the Image**: Use Copacetic to patch the pulled image based on the vulnerability report and save the patched image as an artifact.
3. **Test and Validate the Patched Image**: Load the patched image locally on the runner, run a web service to validate functionality, and upload test results and logs as artifacts for future debugging.

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
    This extracts the "Total" number of vulnerabilities found for visibility in the workflow logs.
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

### **Job 3: Test and Validate the Patched Image**
- **Purpose**: Load the patched image locally, validate its functionality, and save logs for debugging or further validation.
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
  - Run automated tests to ensure the patched image is functional:
    ```bash
    curl -o /dev/null -s -w "%{http_code}\n" http://localhost:8080 | tee test-results.txt
    ```
    Validates the HTTP response; a **200 status code** indicates success.
  - Capture container logs for debugging:
    ```bash
    docker logs patched-nginx > container-logs.txt
    ```
  - Upload test results (`test-results.txt`) and container logs (`container-logs.txt`) as artifacts named `test-results`.
  - Clean up by stopping and removing the container:
    ```bash
    docker stop patched-nginx
    docker rm patched-nginx
    ```
  - Upload the patched image as a final artifact named `patched-image-final`.

---

## **Artifacts Created**
1. `vulnerability-report`: Contains the Trivy-generated JSON vulnerability report.
2. `patched-image`: Contains the patched image saved as a `.tar` file after processing with Copacetic.
3. `test-results`: Contains test results and container logs from validation tests.
4. `patched-image-final`: Contains the final patched image saved for future downloading and use.

---

## **Usage Notes**
- To access the artifacts, navigate to the **"Actions" tab** of your GitHub repository, select the workflow run, and download artifacts from the **"Artifacts" section**.
- This workflow is ideal for automating container image patching, validation, and testing within GitHub-hosted runners or self-hosted environments.

---

