---
layout: post
title: "Automating Docker image vulnerability scanning with Trivy in CI/CD"
date: 2026-10-11 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [docker, trivy, container-security, ci-cd, kubernetes, devops]
description: "Scan Docker images for vulnerabilities at build time using Trivy integrated into a GitHub Actions pipeline. Prevent vulnerable images from reaching registries and clusters."
---

## Why image scanning matters

Docker images can contain dozens of layers from base operating systems and third-party packages. Each layer may carry known CVEs, outdated libraries, or unnecessary root privileges. Scanning images before they enter a registry or cluster is a foundational practice for container security.

This article demonstrates a practical, repeatable workflow: integrate Trivy into a GitHub Actions pipeline to scan images on every push and block deployment of vulnerable findings.

## Prerequisites

- A Docker Hub (or other registry) account
- A GitHub repository with push access
- Trivy installed locally (optional, for the examples below)

```bash
# Install Trivy locally (macOS example)
brew install aquasecurity/trivy/trivy

# Or via binary download from https://github.com/aquasecurity/trivy/releases
```

## Scanning a local image

Trivy can inspect an image by tag or digest. The following command scans the `nginx:latest` image from Docker Hub and returns a structured JSON report.

```bash
trivy image --format json --output report.json nginx:latest
```

The `report.json` contains a list of vulnerabilities, each with severity, title, fixed version, and remediation guidance. Key fields include:

- `Type`: `os` (operating system package) or `library` (language-specific dependency)
- `Severity`: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`
- `PkgName` and `InstalledVersion`: what's installed and where to find a fixed version

## Integrating Trivy into GitHub Actions

Create `.github/workflows/container-scanning.yml` in your repository:

```yaml
name: Container Scanning

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Trivy
        uses: aquasecurity/trivy-action@0.12.0
        with:
          image-ref: "my-registry.example.com/my-app:${{ github.sha }}"
          format: "json"
          output: "trivy-results.json"
          severity: "HIGH,CRITICAL"

      - name: Upload Trivy scan report
        uses: actions/upload-artifact@v4
        with:
          name: trivy-report
          path: trivy-results.json

      - name: Fail on critical vulnerabilities
        run: |
          CRITICAL_COUNT=$(jq '.Results[0].Vulnerabilities | map(select(.Severity == "CRITICAL")) | length' trivy-results.json)
          if [ "$CRITICAL_COUNT" -gt 0 ]; then
            echo "❌ Critical vulnerabilities found: $CRITICAL_COUNT"
            exit 1
          fi
          echo "✅ No critical vulnerabilities detected"
```

### How it works

1. **`aquasecurity/trivy-action`** pulls the image from your registry (or builds and scans in one step).
2. The scan returns JSON results filtered to `HIGH` and `CRITICAL` severities.
3. `jq` parses the output; if any `CRITICAL` vulnerabilities exist, the job fails and the image is not deployed.
4. The report artifact is available for later review in the GitHub Actions UI.

## Enforcing scan results in deployment

A common pattern is to gate ArgoCD or Helm releases on the scan passing. For example, an ArgoCD application can have a `syncOptions` hook that checks a `trivy` report stored in a ConfigMap before allowing a sync.

Alternatively, use the Trivy Operator to emit `Vulnerability` custom resources into your cluster, and constrain `PodSecurityPolicy` or admission controllers to reject pods built from vulnerable images.

## Remediation workflow

When Trivy reports a vulnerability:

1. Note the `PkgName`, `InstalledVersion`, and `FixedVersion` from the report.
2. Update the application's `Dockerfile` or base image tag.
3. Rebuild and push the image; the GitHub Action will re-scan automatically.
4. Once the report shows zero `CRITICAL` findings, unblock the deployment pipeline.

## Additional Trivy capabilities

- **Filesystem scanning**: `trivy fs --type os .` scans the host or a build directory.
- **SBOM generation**: `trivy image --generate SBOM --format cyclonedx -o sbom.json` creates a Software Bill of Materials.
- **Cache support**: Trivy caches layers locally; add `--cache-backend redis` for distributed caching in CI.

## Conclusion

Integrating Trivy into your CI/CD pipeline is a low-effort, high-impact step toward securing containerized workloads. By failing builds on critical vulnerabilities and surfacing actionable remediation data, you prevent vulnerable images from reaching production clusters while keeping the feedback loop fast for developers.