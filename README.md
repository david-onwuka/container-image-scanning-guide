# Container Image Scanning Integration Guide

## Jira Task

*DEV-379 — Create Container Image Scanning Integration Guide*

### Task Requirement

Develop a step-by-step guide for integrating container image scanning tools into the CI/CD pipeline, covering:

- Tool selection
- Scanner configuration
- CI/CD integration
- Security best practices

This repository provides a working Trivy-based implementation using Docker and GitHub Actions, together with detailed operational documentation.

---

## 1. Objective

Container images can contain vulnerabilities in:

- Operating-system packages
- Application dependencies
- Language-specific packages
- Base-image layers
- Build-time dependencies
- Embedded files and configuration

The objective is to introduce automated container image vulnerability scanning into the CI/CD process so security findings are detected after the image is built and before it is promoted or deployed.

The implementation uses:

- Docker for container image creation
- Trivy for vulnerability scanning
- GitHub Actions for CI/CD integration
- SARIF for machine-readable security reporting
- GitHub Security for uploaded SARIF results

---

## 2. Repository Structure

The repository is organized as follows:

    container-image-scanning-guide/
    ├── .github/
    │   └── workflows/
    │       └── container-scan.yml
    ├── docker/
    │   ├── Dockerfile
    │   ├── app.py
    │   └── requirements.txt
    ├── docs/
    │   └── container-image-scanning-guide.md
    ├── sbom/
    ├── .gitignore
    ├── .trivyignore
    └── README.md

### File Responsibilities

| File | Purpose |
|---|---|
| .github/workflows/container-scan.yml | GitHub Actions CI/CD scanning workflow |
| docker/Dockerfile | Builds the demonstration container image |
| docker/app.py | Minimal Flask demonstration application |
| docker/requirements.txt | Python application dependency definition |
| docs/container-image-scanning-guide.md | Detailed integration and operational guide |
| .trivyignore | Controlled location for documented Trivy exceptions |
| sbom/ | Location reserved for SBOM output |
| README.md | Project overview and implementation summary |

---

## 3. Implementation Flow

The implemented security flow is:

    Source Code
        |
        v
    Build Container Image
        |
        v
    Trivy Security Scan
        |
        v
    Evaluate Findings
        |
        +---- Blocking HIGH/CRITICAL ----> Fail Pipeline
        |
        v
    Generate SARIF
        |
        v
    Upload Results to GitHub Security
        |
        v
    Later Publish / Deploy Stage

The scanner is therefore positioned between image creation and later image promotion or deployment.

---

## 4. Tool Selection

The following tools were considered:

| Tool | Type | Relevant Capabilities |
|---|---|---|
| Trivy | Open source | Container vulnerabilities, language dependencies, secrets, IaC, SBOM |
| Grype | Open source | Container and filesystem vulnerability scanning |
| Clair | Open source | Container vulnerability analysis and registry-oriented workflows |
| Snyk Container | Commercial/free tier | Container and dependency security |
| Anchore Enterprise | Commercial | Enterprise vulnerability management and policy enforcement |

### Selected Tool: Trivy

Trivy was selected for this implementation because it:

- Is open source.
- Provides a simple CLI.
- Scans container images directly.
- Detects OS-package vulnerabilities.
- Detects language-specific dependency vulnerabilities.
- Supports severity filtering.
- Supports CI/CD exit codes.
- Supports SARIF output.
- Supports SBOM generation.
- Provides an official GitHub Actions integration.
- Can run locally without requiring a separate scanning server.

This makes it suitable for demonstrating container image scanning without additional infrastructure.

---

## 5. Demonstration Application

The repository contains a minimal Flask application.

The application dependency is defined in:

    docker/requirements.txt

The application source is:

    docker/app.py

The container base image is:

    python:3.11-slim

The application listens on:

    port 5000

The image is configured to run the application as a dedicated non-root user named:

    appuser

Running the application as a non-root user reduces unnecessary privileges inside the container.

---

## 6. Dockerfile

The implemented Dockerfile is:

    FROM python:3.11-slim

    WORKDIR /app

    COPY requirements.txt .

    RUN pip install --no-cache-dir -r requirements.txt && pip uninstall -y setuptools wheel

    COPY app.py .

    RUN useradd --create-home --shell /usr/sbin/nologin appuser \
        && chown -R appuser:appuser /app

    USER appuser

    EXPOSE 5000

    CMD ["python", "app.py"]

The Dockerfile:

1. Uses a slim Python base image.
2. Installs the application dependencies.
3. Removes unnecessary build-time setuptools and wheel packages from the final runtime environment.
4. Creates a dedicated application user.
5. Changes ownership of the application directory.
6. Runs the application as the non-root user.
7. Exposes port 5000.

---

## 7. Building the Container Image

From the repository root, build the image with:

    docker build -t container-scanning-demo:1.0 -f docker/Dockerfile docker/

The Dockerfile is located under docker/, so the -f option explicitly specifies its location.

The build context is also the docker/ directory.

The resulting image is:

    container-scanning-demo:1.0

---

## 8. Running the Container

The image can be tested locally with:

    docker run --rm -p 5000:5000 container-scanning-demo:1.0

The application can then be accessed at:

    http://localhost:5000

The image does not need to be pushed to a container registry for the local scanning demonstration.

---

## 9. Trivy Installation and Version

Trivy was installed locally using the official Aqua Security Debian repository.

The Trivy version used for this implementation is:

    0.74.0

Verify the installation with:

    trivy --version

The CI/CD workflow explicitly uses the same version:

    version: '0.74.0'

Pinning the scanner version improves reproducibility between local development and CI/CD.

---

## 10. Baseline Image Scan

A general Trivy image scan can be performed with:

    trivy image container-scanning-demo:1.0

This performs vulnerability and secret scanning.

For the CI/CD security gate, the scan is restricted to HIGH and CRITICAL vulnerabilities:

    trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 container-scanning-demo:1.0

---

## 11. Security Gate Configuration

The configured policy is:

    Severity: HIGH, CRITICAL
    Ignore unfixed: true
    Exit code: 1

### HIGH and CRITICAL

HIGH and CRITICAL findings with an available fix are treated as blocking findings.

### Unfixed Vulnerabilities

Vulnerabilities for which no fix is currently available are reported but do not block the configured gate.

### Exit Code

Trivy returns exit code 1 when findings meet the configured blocking criteria.

Trivy returns exit code 0 when no findings meet those criteria.

This allows the CI/CD pipeline to determine automatically whether the security gate passes or fails.

---

## 12. Why ignore-unfixed Is Used

Some vulnerabilities may not have an available upstream fix.

Blocking the pipeline on every unfixed vulnerability can prevent remediation-independent progress.

This implementation therefore uses:

    ignore-unfixed: true

The purpose is not to hide unfixed vulnerabilities. They should still be monitored and reviewed during subsequent scans.

The setting should be reviewed against the organization's security policy before production adoption.

---

## 13. GitHub Actions Integration

The CI/CD workflow is located at:

    .github/workflows/container-scan.yml

The workflow runs for:

- Pushes to main
- Pull requests targeting main

The CI process:

1. Checks out the repository.
2. Builds the Docker image.
3. Scans the image with Trivy.
4. Applies the HIGH/CRITICAL security gate.
5. Generates a SARIF report.
6. Uploads the SARIF report to GitHub Security.

---

## 14. Implemented CI/CD Workflow

The current workflow configuration is:

    name: Container Image Security Scan

    on:
      push:
        branches:
          - main
      pull_request:
        branches:
          - main

    permissions:
      contents: read
      security-events: write

    jobs:
      container-scan:
        name: Build and Scan Container
        runs-on: ubuntu-24.04

        steps:
          - name: Checkout repository
            uses: actions/checkout@v4

          - name: Build Docker image
            run: |
              docker build \
                -t container-scanning-demo:${{ github.sha }} \
                -f docker/Dockerfile \
                docker/

          - name: Scan container image with Trivy
            uses: aquasecurity/trivy-action@v0.36.0
            with:
              image-ref: container-scanning-demo:${{ github.sha }}
              version: '0.74.0'
              severity: HIGH,CRITICAL
              ignore-unfixed: true
              exit-code: '1'
              format: sarif
              output: trivy-results.sarif

          - name: Upload Trivy results to GitHub Security
            if: always()
            uses: github/codeql-action/upload-sarif@v4
            with:
              sarif_file: trivy-results.sarif

---

## 15. Image Tagging in CI

The CI workflow tags the image using:

    ${{ github.sha }}

This associates the scanned image with the exact Git commit that generated it.

This is preferable to using a mutable tag such as latest for security validation because the scan is tied to a specific source revision.

---

## 16. GitHub Actions Permissions

The workflow explicitly declares:

    permissions:
      contents: read
      security-events: write

contents: read allows the workflow to check out repository contents.

security-events: write allows the workflow to upload security results to GitHub Security.

Explicit permissions reduce unnecessary workflow privileges.

---

## 17. SARIF Reporting

The Trivy action generates:

    trivy-results.sarif

using:

    format: sarif
    output: trivy-results.sarif

SARIF is a standardized machine-readable security reporting format.

The generated report is uploaded with:

    github/codeql-action/upload-sarif@v4

The upload step uses:

    if: always()

This allows the reporting step to run when the vulnerability scan causes the security gate to fail.

The implementation therefore separates:

- Security enforcement: pipeline pass/fail.
- Security reporting: recording scan findings.

---

## 18. Vulnerability Remediation Demonstrated

The initial image scan identified HIGH vulnerabilities associated with Python packaging components, including:

    jaraco.context
    wheel

Instead of adding those vulnerabilities to .trivyignore, the final runtime image was changed to remove unnecessary build-time setuptools and wheel packages:

    RUN pip install --no-cache-dir -r requirements.txt && pip uninstall -y setuptools wheel

The image was then rebuilt and rescanned.

The final image passed the configured HIGH/CRITICAL security gate.

This demonstrates the remediation workflow:

    Detect
      |
      v
    Investigate
      |
      v
    Remediate
      |
      v
    Rebuild
      |
      v
    Rescan
      |
      v
    Pass Security Gate

---

## 19. Final Local Validation

The final image was scanned with:

    trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 container-scanning-demo:1.0

Final result:

    CRITICAL: 0
    HIGH: 0

The scan returned exit code:

    0

The Debian target reported zero HIGH/CRITICAL vulnerabilities.

The Python package scan also reported zero HIGH/CRITICAL vulnerabilities.

This confirms that the final local image passes the same severity and exit-code policy used by the CI/CD workflow.

---

## 20. Vulnerability Triage Procedure

When Trivy reports a vulnerability:

1. Identify the affected package.
2. Identify the CVE or vulnerability identifier.
3. Check the installed package version.
4. Determine whether a fixed version is available.
5. Determine whether the affected component exists in the final runtime image.
6. Determine whether the dependency is direct or transitive.
7. Upgrade or remove the affected dependency where possible.
8. Rebuild the image.
9. Rescan the image.
10. Use a documented exception only when remediation is not currently practical.

The preferred approach is to remediate the underlying dependency or image issue rather than suppressing the scanner result.

---

## 21. Vulnerability Exceptions

The repository contains:

    .trivyignore

This file is available for controlled and documented exceptions.

Exceptions should not be added merely to make a pipeline pass.

An exception should document:

- Vulnerability identifier.
- Reason for the exception.
- Affected component.
- Owner.
- Review date or expiration.
- Remediation plan where applicable.

Exceptions should be reviewed periodically and removed when they are no longer required.

---

## 22. SBOM Generation

Trivy can generate a Software Bill of Materials.

Example:

    trivy image --format cyclonedx --output sbom/container-scanning-demo-1.0.json container-scanning-demo:1.0

An SBOM provides an inventory of software components contained in the image.

SBOMs can support:

- Vulnerability management
- Software inventory
- Incident response
- Compliance
- Dependency tracking

SBOM generation is documented as part of the integration guide. Production retention and publication should follow organizational requirements.

---

## 23. Container Security Best Practices

### Scan Before Promotion

Scan the image after it is built and before it is pushed to a registry or promoted toward deployment.

### Use Severity Thresholds

Define explicit severity thresholds rather than treating every scanner finding as a deployment blocker.

### Do Not Blindly Ignore Findings

Use .trivyignore only for justified and documented exceptions.

### Monitor Unfixed Vulnerabilities

Unfixed findings should remain visible and should be reviewed during future scans.

### Keep Base Images Updated

Regularly rebuild images to incorporate security updates to the base image.

### Use Minimal Base Images

Minimal base images reduce unnecessary packages and can reduce the vulnerability surface.

### Run Containers as Non-Root

The demonstration container runs using:

    appuser

rather than root.

### Pin Scanner Versions

The CI workflow explicitly specifies:

    version: '0.74.0'

This provides predictable scanner behavior between pipeline runs.

### Generate SBOMs

Generate and retain SBOMs where required.

### Rescan Regularly

New vulnerabilities can be disclosed after an image was initially built.

Scheduled rescanning should therefore complement build-time scanning.

### Store Machine-Readable Results

SARIF enables security results to be consumed by GitHub Security and compatible security-management systems.

---

## 24. CI/CD Rollout Strategy

For an existing production pipeline, container scanning can be introduced in stages.

### Stage 1 — Visibility

Run scans and collect findings without immediately blocking releases.

### Stage 2 — Define Policy

Establish:

- Severity thresholds
- Treatment of unfixed vulnerabilities
- Exception process
- Remediation timeframes
- Ownership

### Stage 3 — Enforce

Enable pipeline failure for actionable findings that violate the agreed security policy.

### Stage 4 — Continuous Monitoring

Add scheduled rescanning to detect newly disclosed vulnerabilities even when application source code has not changed.

---

## 25. Operational Responsibilities

### Developers

- Remediate vulnerable application dependencies.
- Rebuild affected images.
- Review findings related to application components.

### DevOps / Platform

- Maintain the CI/CD scanning integration.
- Maintain scanner configuration.
- Maintain the base-image strategy.
- Monitor security-related pipeline failures.

### Security

- Define vulnerability policy.
- Review exceptions.
- Monitor vulnerability trends.
- Establish remediation requirements.

---

## 26. Troubleshooting

### Dockerfile Not Found

If Docker reports:

    open Dockerfile: no such file or directory

Use:

    docker build -t container-scanning-demo:1.0 -f docker/Dockerfile docker/

### Trivy Not Installed

Verify:

    trivy --version

### Slow Scans

Trivy may perform vulnerability and secret scanning.

For vulnerability-only scanning:

    trivy image --scanners vuln container-scanning-demo:1.0

### Pipeline Failure

A Trivy exit code of 1 indicates that the configured security-gate criteria were met.

Review the findings before considering an exception.

### SARIF Upload Problems

Verify that the workflow contains:

    security-events: write

and that the scan generates:

    trivy-results.sarif

---

## 27. Implementation Checklist

### Tool Selection

- [x] Evaluate container scanning tools.
- [x] Select Trivy.
- [x] Document selection criteria.

### Scanner Configuration

- [x] Install Trivy.
- [x] Verify Trivy version.
- [x] Configure HIGH and CRITICAL severity filtering.
- [x] Configure unfixed vulnerability handling.
- [x] Configure CI/CD exit-code behavior.
- [x] Pin Trivy version in CI/CD.

### Container

- [x] Create demonstration Dockerfile.
- [x] Build demonstration image.
- [x] Run the image locally.
- [x] Configure the container to run as a non-root user.
- [x] Remove unnecessary build-time packages from the final runtime image.

### CI/CD

- [x] Create GitHub Actions workflow.
- [x] Build the image in CI.
- [x] Scan the CI-built image.
- [x] Tag the image using the Git commit SHA.
- [x] Configure the security gate.
- [x] Generate SARIF results.
- [x] Upload SARIF to GitHub Security.

### Documentation

- [x] Document tool selection.
- [x] Document scanner configuration.
- [x] Document CI/CD integration.
- [x] Document vulnerability triage.
- [x] Document exception handling.
- [x] Document SBOM generation.
- [x] Document security best practices.
- [x] Document troubleshooting.
- [x] Document rollout strategy.

### Validation

- [x] Run local Trivy scan.
- [x] Confirm HIGH/CRITICAL scan returns zero findings for the final image.
- [x] Confirm the security gate returns exit code 0 for the final image.

---

## 28. Jira Requirement Coverage

DEV-379 requires a step-by-step guide covering tool selection, configuration, CI/CD integration, and best practices.

### Tool Selection

Covered by the comparison of Trivy, Grype, Clair, Snyk Container, and Anchore Enterprise, followed by selection of Trivy.

### Configuration

Covered by:

- Trivy version
- Severity threshold
- ignore-unfixed
- exit-code
- SARIF output
- GitHub Actions configuration
- Explicit workflow permissions

### CI/CD Integration

Implemented using GitHub Actions.

The workflow:

1. Checks out the repository.
2. Builds the container image.
3. Scans the image with Trivy.
4. Applies the vulnerability gate.
5. Generates SARIF.
6. Uploads the results to GitHub Security.

### Best Practices

Covered by:

- Non-root containers
- Minimal base images
- Base-image updates
- Dependency remediation
- Severity thresholds
- Unfixed vulnerability handling
- Controlled exceptions
- SBOM generation
- SARIF reporting
- Scanner version pinning
- Scheduled rescanning
- Vulnerability triage

---

## 29. Final Implementation Summary

The completed repository provides a working example of container image scanning integrated into CI/CD.

The final implementation consists of:

    Docker
       |
       v
    Container Image
       |
       v
    Trivy 0.74.0
       |
       v
    HIGH / CRITICAL Security Gate
       |
       +---- Finding ----> Pipeline Failure
       |
       v
    SARIF Report
       |
       v
    GitHub Security

The final local image passed the configured HIGH/CRITICAL gate with:

    HIGH: 0
    CRITICAL: 0

The implementation therefore provides the required:

1. Tool selection.
2. Scanner configuration.
3. CI/CD integration.
4. Security gating.
5. SARIF reporting.
6. Vulnerability remediation guidance.
7. Exception handling guidance.
8. SBOM generation guidance.
9. Container security best practices.
10. Operational and troubleshooting guidance.

---

## 30. References

Official Trivy documentation:

https://trivy.dev/

Official Trivy GitHub repository:

https://github.com/aquasecurity/trivy

Official Trivy GitHub Actions integration:

https://github.com/aquasecurity/trivy-action

GitHub SARIF upload action:

https://github.com/github/codeql-action

GitHub Actions documentation:

https://docs.github.com/actions
