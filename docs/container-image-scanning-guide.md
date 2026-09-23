# Container Image Scanning Integration Guide

## 1. Purpose

This guide describes how to integrate container image security scanning into a CI/CD pipeline using Trivy and GitHub Actions.

The implementation covers:

- Container image vulnerability scanning
- Security tool selection
- Scanner configuration
- CI/CD integration
- HIGH and CRITICAL vulnerability gating
- Handling vulnerabilities without available fixes
- SARIF security reporting
- GitHub Security integration
- SBOM generation
- Container hardening
- Vulnerability remediation
- Exception handling
- Rollout and operational best practices

The example implementation uses a small Flask application packaged as a Docker image.

---

# 2. Security Problem

Container images can contain vulnerabilities even when the application source code itself does not contain an obvious security issue.

Vulnerabilities may originate from:

- Operating-system packages
- Application dependencies
- Transitive dependencies
- Base-image layers
- Packages installed during the image build
- Secrets accidentally included in the image

A container image should therefore be scanned before it is published or deployed.

The CI/CD pipeline should turn the scan into an enforceable security control rather than simply printing vulnerability information to the build log.

The implementation in this repository follows this model:

    Source Code
        |
        v
    Build Container Image
        |
        v
    Scan Image
        |
        v
    Evaluate Security Policy
        |
        +-------------------------+
        |                         |
    Blocking findings        No blocking findings
        |                         |
        v                         v
    Fail pipeline             Continue pipeline
        |
        v
    Remediate
        |
        v
    Rebuild and re-scan

---

# 3. Tool Selection

## 3.1 Selection Criteria

A container image scanning tool should be evaluated against the following requirements:

- Container image vulnerability detection
- Operating-system package scanning
- Application dependency scanning
- CI/CD integration
- Machine-readable output
- SARIF support
- SBOM generation
- Support for automated security gates
- Maintainability
- Licensing and cost
- Network/offline requirements
- Existing organizational tooling

## 3.2 Tools Considered

| Tool | General capability | Typical use |
|---|---|---|
| Trivy | Container, dependency, SBOM and security scanning | Developer and CI/CD scanning |
| Grype | Container vulnerability scanning | Vulnerability scanning, often paired with Syft |
| Clair | Container vulnerability analysis | Registry-oriented environments |
| Snyk Container | Container and dependency security | Commercial developer/security platform |
| Anchore Enterprise | Enterprise container security and policy | Centralized enterprise security |

## 3.3 Selected Tool: Trivy

Trivy was selected for this implementation because it provides:

- Open-source availability
- CLI-based operation
- Container image vulnerability scanning
- Application dependency scanning
- CI/CD integration
- SARIF output
- SBOM generation
- GitHub Actions integration
- Support for severity-based security gates

The tool can be run locally by developers and automatically inside GitHub Actions.

---

# 4. Container Image Used for the Demonstration

The demonstration image is built from:

    python:3.11-slim

The application uses Flask.

The Dockerfile also creates a dedicated non-root user:

    appuser

This is important because the container does not require root privileges to run the application.

The Dockerfile therefore demonstrates both:

1. Container image scanning
2. Basic container hardening

The Dockerfile is located at:

    docker/Dockerfile

---

# 5. Build the Container Image

Build the image locally:

    docker build \
      -t container-scanning-demo:1.0 \
      -f docker/Dockerfile \
      docker/

Verify that the image exists:

    docker images | grep container-scanning-demo

Expected image name:

    container-scanning-demo

Expected tag:

    1.0

---

# 6. Run the Container

Run the application:

    docker run --rm -p 5000:5000 container-scanning-demo:1.0

The application listens on port 5000.

Test it locally:

    http://localhost:5000

The container can be stopped with:

    Ctrl+C

---

# 7. Install and Verify Trivy

Verify the Trivy installation:

    trivy --version

The scanner version should be controlled in CI/CD rather than relying on an unspecified version.

The local implementation used Trivy 0.74.0.

Keeping scanner versions controlled improves reproducibility and makes changes easier to audit.

---

# 8. Run a Baseline Scan

Before introducing a blocking policy, perform a complete scan:

    trivy image container-scanning-demo:1.0

The baseline scan provides visibility into vulnerabilities present in:

- Operating-system packages
- Python packages
- Other components detected by Trivy

The baseline should be recorded before introducing enforcement so the team understands the existing security state.

---

# 9. Configure the Security Threshold

The demonstration CI/CD policy evaluates:

    HIGH
    CRITICAL

Run the focused scan locally:

    trivy image \
      --severity HIGH,CRITICAL \
      container-scanning-demo:1.0

This does not remove lower-severity vulnerabilities from the image.

It only determines which severity levels are evaluated by this particular security policy.

---

# 10. Configure the CI/CD Security Gate

The local equivalent of the CI/CD gate is:

    trivy image \
      --severity HIGH,CRITICAL \
      --ignore-unfixed \
      --exit-code 1 \
      container-scanning-demo:1.0

The important settings are:

    severity: HIGH,CRITICAL
    ignore-unfixed: true
    exit-code: 1

## 10.1 Severity

HIGH and CRITICAL findings are treated as blocking findings when a fix is available.

This threshold is an example policy and should be adjusted according to the organization's risk-management requirements.

## 10.2 Ignore Unfixed

The gate uses:

    --ignore-unfixed

This prevents vulnerabilities without an available fix from blocking the pipeline.

This does not mean the vulnerabilities are ignored operationally.

Unfixed vulnerabilities should remain visible, be tracked, and be re-evaluated when upstream fixes become available.

## 10.3 Exit Code

The gate uses:

    --exit-code 1

When Trivy detects blocking findings, it returns exit code 1.

The CI/CD system interprets the non-zero exit code as a failed job.

This is what converts the scanner from a reporting tool into an enforcement mechanism.

---

# 11. GitHub Actions Integration

The CI/CD workflow is:

    .github/workflows/container-scan.yml

The workflow is triggered by:

- Pushes to main
- Pull requests targeting main

The pipeline performs these steps:

1. Checkout the repository
2. Build the container image
3. Scan the image with Trivy
4. Evaluate HIGH and CRITICAL findings
5. Fail the scan when blocking findings are detected
6. Generate a SARIF report
7. Upload the SARIF report to GitHub Security

The workflow uses the commit SHA as the image tag:

    container-scanning-demo:${{ github.sha }}

Using the commit SHA gives each CI build an immutable image identifier tied to the source revision that produced it.

---

# 12. CI/CD Workflow

The implementation follows:

    Pull Request / Push
            |
            v
       Checkout Code
            |
            v
       Build Image
            |
            v
       Trivy Scan
            |
            v
    HIGH / CRITICAL Gate
            |
       +----+----+
       |         |
     Fail       Pass
       |         |
       v         v
    Remediate  Continue
       |
       v
    Rebuild
       |
       v
     Re-scan

The important security principle is that the image being scanned is the image produced by the CI build.

---

# 13. SARIF Reporting

The Trivy workflow generates:

    trivy-results.sarif

SARIF is a standardized format for security analysis results.

The workflow uploads the SARIF file to GitHub Security.

The workflow grants:

    contents: read
    security-events: write

The security-events permission is required so the workflow can upload security analysis results.

The upload step uses an always-run condition so that the SARIF report can still be uploaded when the Trivy security gate fails.

This separates:

- Security enforcement
- Security reporting

A failed pipeline can therefore still produce a persistent security record.

---

# 14. SBOM Generation

A Software Bill of Materials provides an inventory of components detected in the image.

Generate a CycloneDX SBOM with:

    trivy image \
      --format cyclonedx \
      --output sbom/container-scanning-demo.cdx.json \
      container-scanning-demo:1.0

The SBOM can be used for:

- Software inventory
- Vulnerability management
- Dependency tracking
- Incident response
- Compliance
- Supply-chain analysis

In a production pipeline, SBOM files should normally be stored as build artifacts or submitted to an approved software-supply-chain platform.

---

# 15. Initial Scan Results

The demonstration image was scanned locally.

The HIGH/CRITICAL-focused scan returned:

    CRITICAL: 0
    HIGH:     46

The findings were distributed across Debian operating-system packages and Python dependencies.

The 46 HIGH findings should not automatically be interpreted as 46 unique CVEs. A single vulnerability can affect multiple packages or targets.

Examples of affected Python dependencies included:

- jaraco.context
- wheel

Multiple Debian packages inherited from the base image also produced HIGH-severity findings.

The correct engineering response is to identify the affected component, determine whether a fix exists, update the relevant dependency or base image, rebuild the image, and re-scan.

The purpose of this project is to demonstrate the security integration and gating mechanism. It is not necessary to suppress all findings simply to obtain a green pipeline.

---

# 16. Vulnerability Triage

When a vulnerability is reported:

    Finding
       |
       v
    Identify affected package
       |
       v
    Review severity and vulnerability details
       |
       v
    Check whether a fixed version exists
       |
       +-------------------------+
       |                         |
    Fix available            No fix available
       |                         |
       v                         v
    Update package          Track and monitor
    or base image           the vulnerability
       |                         |
       +------------+------------+
                    |
                    v
                Rebuild
                    |
                    v
                 Re-scan
                    |
                    v
             Verify remediation

The preferred remediation process is:

1. Identify the vulnerable package.
2. Determine whether it comes from the application dependency layer or base image.
3. Check the vulnerability details.
4. Identify a fixed version if available.
5. Update the dependency or base image.
6. Rebuild the container.
7. Re-run Trivy.
8. Confirm that the vulnerability is resolved.

---

# 17. Vulnerabilities Without Available Fixes

Not every vulnerability has an available fix.

When no fix exists:

1. Record the vulnerability.
2. Assess its relevance to the deployed environment.
3. Determine whether compensating controls exist.
4. Monitor the upstream project.
5. Re-scan periodically.
6. Reassess when a fixed version becomes available.

The CI gate uses ignore-unfixed so that an unfixed vulnerability does not automatically block delivery.

This should not be interpreted as a permanent exemption.

---

# 18. Vulnerability Exceptions

The .trivyignore file should be used only for approved exceptions.

An exception should have:

- Vulnerability identifier
- Reason for suppression
- Owner
- Review date
- Remediation or reassessment plan

Example format:

    CVE-YYYY-NNNNN

    # Reason:
    # No upstream fix currently available.
    # Reassess during scheduled security review.

Exceptions should be reviewed regularly.

Do not add vulnerabilities to .trivyignore merely because they cause a pipeline failure.

---

# 19. Container Security Best Practices

## 19.1 Use Minimal Base Images

Use an appropriately maintained minimal base image where practical.

A smaller image generally contains fewer packages and therefore fewer potential attack surfaces.

The demonstration uses:

    python:3.11-slim

The base image must still be scanned and updated regularly.

## 19.2 Run as a Non-Root User

The Dockerfile creates:

    appuser

The application runs under this account rather than root.

This follows the principle of least privilege.

## 19.3 Scan Before Publishing

The recommended sequence is:

    Build
      |
      v
    Scan
      |
      v
    Security Gate
      |
      v
    Publish
      |
      v
    Deploy

An image should not be promoted to a release artifact before required security controls have passed.

## 19.4 Pin Dependencies

Application dependencies and CI/CD actions should use controlled versions.

This improves reproducibility and reduces unexpected changes.

## 19.5 Keep Scanner Data Current

Trivy relies on vulnerability database information.

The CI environment should be able to obtain current vulnerability data.

Scheduled scanning is important because new vulnerabilities can be disclosed after an image was originally built.

## 19.6 Scan Regularly

Recommended scan triggers include:

- Pull requests
- Main branch builds
- Release builds
- Scheduled scans
- Base-image updates

## 19.7 Do Not Suppress Findings to Make CI Green

A passing pipeline should represent a meaningful security decision.

Suppressions should be based on documented risk assessment rather than convenience.

---

# 20. Tool Configuration Considerations

The scanner configuration should be treated as part of the security policy.

Important configuration decisions include:

### Severity threshold

Example:

    HIGH,CRITICAL

### Fix availability

Example:

    ignore-unfixed: true

### Failure behavior

Example:

    exit-code: 1

### Output format

Example:

    SARIF

### Reporting destination

Example:

    GitHub Security

These settings should be reviewed whenever the organization's security policy changes.

---

# 21. CI/CD Action Versioning

CI/CD actions should use controlled versions.

The demonstration workflow uses versioned GitHub Actions rather than unversioned action references.

For higher-assurance production environments, actions can be pinned to immutable commit SHAs and reviewed through the organization's dependency-management process.

Scanner versions should also be deliberately controlled.

This reduces the risk of unexpected changes in CI behavior.

---

# 22. Rollout Strategy

A security gate should be introduced in stages where an existing organization has no container scanning process.

## Stage 1 — Visibility

Run scans without blocking delivery.

Objectives:

- Establish a vulnerability baseline
- Identify recurring vulnerability sources
- Assign remediation ownership
- Understand the impact on existing images

## Stage 2 — Controlled Enforcement

Introduce a defined severity threshold such as HIGH and CRITICAL.

Establish an exception process before enforcement becomes mandatory.

## Stage 3 — Continuous Monitoring

Add scheduled scans to detect newly disclosed vulnerabilities even when application source code has not changed.

## Stage 4 — Expanded Security Coverage

Depending on organizational requirements, add:

- SBOM publication
- Secret scanning
- Dockerfile/configuration scanning
- Registry scanning
- Dependency scanning
- Centralized vulnerability management
- Policy enforcement
- Security dashboards

---

# 23. Troubleshooting

## Trivy is not installed

Verify:

    trivy --version

If the command is unavailable, install Trivy using the approved installation method for the operating system.

## Docker image does not exist

Verify:

    docker images

Then scan the exact image name and tag:

    trivy image container-scanning-demo:1.0

## Trivy returns exit code 1

This can be an expected result.

Review the output for HIGH or CRITICAL findings that meet the configured gate.

A failed security gate should trigger vulnerability triage.

## SARIF upload fails

Verify that the workflow has:

    permissions:
      contents: read
      security-events: write

Also verify that the Trivy step generated:

    trivy-results.sarif

## A vulnerability has no fixed version

Do not automatically suppress it.

Track the vulnerability, assess the risk, and monitor for an upstream fix.

---

# 24. Operational Responsibilities

A production implementation should define ownership.

Recommended responsibilities include:

### Developers

- Review findings affecting application dependencies
- Update vulnerable packages
- Respond to failed security gates

### DevOps / Platform Engineering

- Maintain the CI/CD integration
- Maintain scanner configuration
- Maintain build infrastructure
- Monitor pipeline failures

### Security Team

- Define security thresholds
- Review exceptions
- Establish vulnerability-management policy
- Monitor security trends

Ownership should be adapted to the organization's team structure.

---

# 25. Implementation Checklist

## Tool Selection

- [x] Evaluate container scanning tools
- [x] Select Trivy for the demonstration
- [x] Define selection criteria

## Container

- [x] Create demonstration Dockerfile
- [x] Build container image
- [x] Run container locally
- [x] Configure non-root execution

## Scanning

- [x] Install Trivy
- [x] Perform baseline scan
- [x] Configure HIGH/CRITICAL filtering
- [x] Configure unfixed vulnerability handling
- [x] Configure non-zero exit code for blocking findings

## CI/CD

- [x] Create GitHub Actions workflow
- [x] Build image in CI
- [x] Scan CI-built image
- [x] Configure security gate
- [x] Generate SARIF output
- [x] Upload SARIF to GitHub Security

## SBOM

- [x] Define SBOM generation procedure
- [ ] Generate SBOM as a CI artifact

## Documentation

- [x] Document tool selection
- [x] Document configuration
- [x] Document CI/CD integration
- [x] Document vulnerability handling
- [x] Document exceptions
- [x] Document best practices
- [x] Document rollout strategy
- [x] Document troubleshooting

---

# 26. Final CI/CD Integration

The resulting security flow is:

    Developer
        |
        v
    Git push / Pull Request
        |
        v
    GitHub Actions
        |
        v
    Build Container Image
        |
        v
    Trivy Scan
        |
        v
    Evaluate HIGH / CRITICAL
        |
        +------------------------+
        |                        |
    Blocking findings       No blocking findings
        |                        |
        v                        v
    Fail pipeline            Pass gate
        |                        |
        v                        v
    Remediate                Continue pipeline
        |                        |
        v                        v
    Rebuild                  Publish / Deploy
        |
        v
     Re-scan

This approach places container security directly into the software delivery lifecycle.

The scanner provides the vulnerability information, the configured policy determines what blocks delivery, and the CI/CD system enforces the result.

---

# 27. References

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
