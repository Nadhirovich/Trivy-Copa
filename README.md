# Trivy-Copa mini project!

# GitHub Actions Workflow: Patch, Validate, and Use Local Image

This GitHub Actions workflow automates the process of scanning, patching, validating, and saving a container image for future use. Below is an explanation of each job and its functionality.

---

## **Workflow Overview**

The workflow consists of **three sequential jobs**:
1. **Scan Vulnerabilities**: Pull the unpatched image, install Trivy, generate a vulnerability report, and save the report as an artifact.
2. **Patch the Image**: Use Copacetic to patch the pulled image based on the vulnerability report and save the patched image as an artifact.
3. **Test and Validate the Patched Image**: Load the patched image locally on the runner, run a web service to validate functionality, and upload test results and logs as artifacts for future debugging.

---

## **Workflow Details**

### **Job 1: Scan Vulnerabilities**
- **Purpose**: Pull the unpatched image, install Trivy, scan for vulnerabilities, and generate a report.
- **Steps**:
  1. **Pull the unpatched image**:
     ```bash
     docker pull nginx:1.21.6
     ```
  2. **Install Trivy**:
     Trivy is installed on the runner using `apt` to ensure the vulnerability scanning tool is available. The installation process includes:
     ```bash
     sudo apt-get update && sudo apt-get install -y wget
     wget https://github.com/aquasecurity/trivy/releases/download/v0.61.1/trivy_0.61.1_Linux-64bit.deb
     sudo dpkg -i trivy_0.61.1_Linux-64bit.deb
     ```
  3. **Run a vulnerability scan** and extract the "Total" number of vulnerabilities for visibility in the logs:
     ```bash
     trivy image --vul-type os --ignore-unfixed nginx:1.21.6 | grep "Total"
     ```
  4. **Generate a full JSON vulnerability report** using Trivy.
  5. **Save the vulnerability report** as an artifact named `vulnerability-report`.

---

### **Job 2: Patch the Image**
- **Purpose**: Patch the vulnerabilities identified in the report using Copacetic.
- **Steps**:
  1. Download the `vulnerability-report` artifact from Job 1.
  2. Use **Copacetic** to patch the image (`nginx:1.21.6`) based on the vulnerability report:
     ```yaml
     uses: project-copacetic/copa-action@v1.0.0
     with:
       image: nginx:1.21.6
       image-report: 'report.json'
       patched-tag: nginx:patched
     ```
  3. Save the patched image as a `.tar` file using:
     ```bash
     docker save nginx:patched -o patched-image.tar
     ```
  4. Upload the patched image as a workflow artifact named `patched-image`.

---

### **Job 3: Test and Validate the Patched Image**
- **Purpose**: Load the patched image locally, validate its functionality, and save logs for debugging or further validation.
- **Steps**:
  1. Download the `patched-image` artifact from Job 2.
  2. Load the patched image into Docker on the **runner's local environment** using:
     ```bash
     docker load < patched-image.tar
     ```
     *Note*: "Locally" refers to the GitHub Actions runner where the workflow is executed.
  3. Start a web service container using the patched image:
     ```bash
     docker run -d --name patched-nginx -p 8080:80 nginx:patched
     ```
     This exposes the container on port **8080** of the runner.
  4. Run automated tests to ensure the patched image is functional:
     ```bash
     curl -o /dev/null -s -w "%{http_code}\n" http://localhost:8080 | tee test-results.txt
     ```
     Validates the HTTP response; a **200 status code** indicates success.
  5. Capture container logs for debugging:
     ```bash
     docker logs patched-nginx > container-logs.txt
     ```
  6. Upload test results (`test-results.txt`) and container logs (`container-logs.txt`) as artifacts named `test-results`.
  7. Clean up by stopping and removing the container:
     ```bash
     docker stop patched-nginx
     docker rm patched-nginx
     ```
  8. Upload the patched image as a final artifact named `patched-image-final`.

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

