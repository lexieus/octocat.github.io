layout: post
title: "Building a Production-Ready iOS CI/CD Pipeline with GitHub Actions, .NET MAUI & Fastlane"
date: 2026-08-23
author: Chandana Nawarathna
categories: [DevOps, CI/CD, Mobile DevOps]
tags: [GitHub Actions, .NET 9, .NET MAUI, Fastlane, iOS, TestFlight, Xcode, App Store Connect, Automation, Code Signing, Self-Hosted Runner]
---


# Building a Production-Ready iOS CI/CD Pipeline with GitHub Actions, .NET MAUI & Fastlane

Modern mobile applications need more than a successful build. A reliable release process should provide **repeatability, security, automated signing, controlled deployments, and traceability**.

For an iOS application built with **.NET MAUI**, GitHub Actions combined with a self-hosted macOS runner and Fastlane can provide a powerful CI/CD platform.

This article walks through the key components of a production-oriented iOS pipeline and the lessons that can be applied to other mobile DevOps environments.

## 1. The CI/CD Architecture

A clean pipeline should separate the release process into logical stages.

A typical flow is:

**Source Code → Environment Preparation → Dependency Restore → Signing → Build → Artifact → Approval → TestFlight**

The workflow discussed here separates the environment setup, iOS build, and TestFlight upload into different jobs. The build job depends on environment preparation, while the TestFlight job depends on a successful iOS build. 
This separation makes troubleshooting and maintenance much easier.

## 2. Why Use a Self-Hosted macOS Runner?

iOS development requires Apple's build ecosystem, including Xcode and Apple's signing infrastructure.

A self-hosted macOS runner provides greater control over:

- Xcode versions
- Installed SDKs
- .NET SDK versions
- Ruby and Fastlane
- Keychains
- Build caches
- Network connectivity
- Internal package repositories

The pipeline explicitly switches the runner to **Xcode 26.5** before continuing with the build process.

This is especially useful when an organization needs a predictable and controlled build environment.

## 3. Keep the .NET Environment Predictable

One of the common problems with self-hosted runners is environment drift.

Different workflow runs can accidentally use different versions of .NET, global.json files, cached SDKs, or architecture-specific binaries.

The pipeline removes existing .NET installations and then installs a specific SDK version:

**.NET SDK 9.0.305**

It also verifies the installed architecture before continuing. 
This is an important DevOps principle:

> A build server should be predictable, not dependent on whatever happens to be installed on it.

## 4. Dependency Management Matters

The application uses .NET MAUI workloads and restores the required dependencies before the actual iOS build.

The pipeline installs MAUI workloads including Android, iOS and WebAssembly tooling and restores the project dependencies.

Internal NuGet repositories are also configured for organization-specific packages.

This approach allows the pipeline to work with both public and private dependencies while keeping the package configuration inside the automated build process.

## 5. iOS Code Signing — The Critical Part

Building an iOS application is only part of the problem.

For distribution, the application must be correctly signed and associated with an appropriate provisioning profile.

The pipeline creates a temporary keychain specifically for the build:

- Create keychain
- Set it as the default
- Unlock it
- Configure its timeout
- Add it to the keychain search path



This is much safer than relying blindly on whatever happens to exist in the runner's default login keychain.

The pipeline then uses Fastlane Match to retrieve the signing assets.

## 6. Verify Signing Assets Before Building

One of the most important lessons from iOS CI/CD troubleshooting is:

**Don't wait for `dotnet publish` to discover that signing is broken.**

The workflow explicitly checks:

- Code-signing identities
- Installed provisioning profiles
- Provisioning profile names

before starting the build.

This makes failures easier to diagnose.

Instead of receiving a generic build failure, the pipeline can tell us whether the required certificate or provisioning profile is actually available.

## 7. Build for a Physical iOS Device

The pipeline uses:

```bash
dotnet publish -f net9.0-ios -c Release -r ios-arm64
```

and provides the signing identity, provisioning profile and Apple development team information during the build.

The important point is that signing assets must already be available **before** `dotnet publish` executes because the publishing process invokes the Apple build/signing toolchain.

## 8. Separate Build and Deployment

A good CI/CD pipeline should not automatically deploy every successful build to production distribution services.

The workflow creates an artifact after the iOS build.

The next job downloads that artifact and performs a separate TestFlight deployment.

This provides a clean separation:

**Build → Artifact → Approval → Deployment**

That separation is valuable because the same artifact that was tested can be promoted without rebuilding the application.

## 9. Add an Approval Gate

The workflow includes a manual approval input before the TestFlight upload.

The deployment proceeds only when the approval value is:

```text
yes
```

Otherwise, the deployment stops.

This is a simple but effective release-control mechanism.

For organizations with formal release processes, this can be extended with environment approvals, change-management systems, or other governance controls.

## 10. Use App Store Connect API Authentication

The pipeline uses an App Store Connect API key rather than relying on interactive Apple credentials.

The API private key is temporarily decoded into `AuthKey.p8` and used during the TestFlight deployment.

The API key is then explicitly removed after deployment.

This follows an important security principle:

**Secrets should have the shortest practical lifetime.**

## 11. Clean Up Persistent Self-Hosted Runners

Self-hosted runners are different from ephemeral GitHub-hosted runners.

Files can remain between workflow executions.

The cleanup stage therefore removes:

- Temporary keychains
- Provisioning profiles
- .NET installations
- Build outputs
- Temporary configuration files



This prevents one build from unintentionally affecting another.

It also reduces the risk of sensitive signing material remaining on the server.

## 12. Lessons for DevOps Engineers

There are several broader lessons from this pipeline.

### Infrastructure consistency

Pin important tool versions such as Xcode and .NET rather than relying on whatever happens to be installed.

### Security by design

Use secrets management, temporary keychains and short-lived authentication files.

### Fail early

Validate certificates and provisioning profiles before starting an expensive application build.

### Separate responsibilities

Keep environment preparation, building and deployment as independent pipeline stages.

### Artifact-based deployment

Build once and deploy the generated artifact rather than rebuilding for every environment.

### Self-hosted runner hygiene

Always assume that a self-hosted runner contains leftovers from previous executions.

## Conclusion

A reliable iOS CI/CD pipeline is not simply a command that runs `dotnet publish`.

It is an engineered system that combines:

**GitHub Actions + macOS + Xcode + .NET MAUI + Fastlane + Apple Signing + App Store Connect**

The real value comes from making the entire release process **repeatable, secure, observable and predictable**.

When these principles are applied correctly, developers can focus on application development while DevOps automation handles the complex path from source code to TestFlight.

**Build once. Validate early. Sign securely. Deploy with confidence.**
