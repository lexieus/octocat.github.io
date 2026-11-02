---
layout: post
title: "Automated Container Image Security Scanning with Trivy in GitHub Actions"
date: 2026-11-02 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [trivy, container-security, github-actions, ci-cd, devops]
description: "Run Trivy vulnerability scanning on container images within GitHub Actions to catch CVEs early in the CI pipeline, with practical example commands and configuration."
---

# Why image security scanning matters

Container images are the foundation of modern CI/CD pipelines. A single vulnerable base image can expose an entire deployment to known CVEs, supply‑chain attacks, or runtime exploits. Running a lightweight vulnerability scan as part of the pipeline catches these issues before the image reaches production, without requiring a separate security toolchain.

# Introducing Trivy

[Trivy](https://github.com/aquasecurity/trivy) is an open‑source container security scanner maintained by Aqua Security. It supports:

* Scanning remote and local images (Docker, OCI, Buildah, Kaniko)
* Detecting CVEs in OS packages and language‑specific dependencies
* Generating SBOMs (Software Bill of Materials)
* Integrating with CI systems via exit codes and JSON output

Trivy is purpose‑built for speed and simplicity, making it a natural fit for GitHub Actions.

# Scanning a local image

The simplest way to get started is to scan an image already present on the host:

```bash
# Scan the alpine:latest image from the local Docker daemon
trivy image alpine:latest
```

Typical output highlights any CVEs found in the distribution packages:

```
$ trivy image alpine:latest

2024-02-15T12:34:56Z INFO Loading template from local cache
2024-02-15T12:34:56Z INFO Detecting package manager: apk
2024-02-15T12:34:56Z INFO Detecting OS: alpine 3.19
2024-02-15T12:34:56Z WARN CVE-2023-42303: CVE found in glibc (fixed in 2.37-r0)
2024-02-15T12:34:56Z WARN CVE-2023-42302: CVE found in libstdc++ (fixed in 13.2.1-r0)
```

To limit the scan to only critical vulnerabilities and fail the job:

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 alpine:latest
```

# Scanning in GitHub Actions

The following workflow demonstrates a complete Trivy scan step. It works for images built in the same workflow or for pushing a newly built image to a registry.

```yaml
name: Container security scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up QEMU (for multi‑arch support, optional)
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.REGISTRY }}/myapp:${{ github.sha }}
          cache-from: type=registry,ref=${{ secrets.REGISTRY }}/myapp:cache
          cache-to: type=registry,ref=${{ secrets.REGISTRY }}/myapp:cache,mode=max

      - name: Run Trivy vulnerability scan
        aquasecurity/trivy-action@master
        with:
          image-ref: ${{ secrets.REGISTRY }}/myapp:${{ github.sha }}
          severity: HIGH,CRITICAL
          format: sarif
          exit-code: 1
          cache-ref: ghcr.io/aquasecurity/trivy-cache:latest
          ignore-unfixed: false
```

**Key parameters explained**

| Parameter | Purpose |
|---|---|
| `image-ref` | The full image name (registry + repo + tag) to scan. |
| `severity` | Only report vulnerabilities at the specified levels; defaults to all. |
| `format: sarif` | Outputs results in the SARIF format, which GitHub Security tab can display natively. |
| `exit-code: 1` | Makes the job fail if any vulnerability of the selected severities is found. |
| `cache-ref` | Reuses a cached vulnerability database across workflow runs, speeding up subsequent scans. |

# Interpreting the results

When the job finishes, open the **Security** tab of the pull request or run. Trivy lists each CVE with:

* **Pkg name** – vulnerable package
* **Installed version** – version present in the image
* **Fixed version** – version that contains the fix
* **Severity** – HIGH, CRITICAL, etc.
* **Link** – upstream advisory (e.g., NVD)

The SARIF output automatically groups findings by file and line, making it easy to triage.

# Customizing the scan

* **Only runtime packages** – `trivy image --type os alpine:latest`
* **Language dependency caches** – `trivy image --type pip node`. Useful when you want to skip OS‑level CVEs and focus on application‑level dependencies.
* **SBOM generation** – add `--generate-sbom --sbom-format spdx` to produce a SPDX SBOM that can be fed to other tooling.
* **Whitelisting** – create a `trivyignore` file to suppress false positives for non‑exploitable CVEs.

# Bonus: Integrating with GitHub status check

After the Trivy step, the workflow can optionally post a comment on the PR with a summary of findings:

```yaml
      - name: Comment on PR with scan results
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const trivyOutput = process.env.TRIVY_SARIF_OUTPUT || '';
            // Parse SARIF and post a concise comment (example omitted for brevity)
            core.setOutput('findings-count', '3'); // placeholder
```

In practice you would parse the SARIF file and construct a Markdown table, but the above shows the hook point.

# Key takeaways

* Add a Trivy scan early in your CI pipeline to catch CVEs before images are deployed.
* Use `severity` and `exit-code` to make the scan a gate that blocks merges when critical issues exist.
* Leverage the SARIF format for native GitHub Security tab integration.
* Cache the Trivy vulnerability database (`cache-ref`) to reduce scan time on subsequent runs.
* Combine Trivy with SBOM generation for a complete view of your software supply chain.

*This article is based on Trivy’s official documentation and common GitHub Actions patterns. Adjust severity levels, cache configuration, and registry credentials to match your environment.*