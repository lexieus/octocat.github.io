---
layout: post
title: "Ingress NGINX Is Being Retired — What Kubernetes Users Should Do Next"
date: 2025-11-18
author: "Chandana Nawarathna"
categories: [kubernetes, cloud-native, networking]
tags: [Kubernetes, IngressNGINX, GatewayAPI, DevOps, SRE, Security]
excerpt: "Ingress NGINX will be retired in March 2026. Start planning migration to Gateway API or another maintained ingress controller now to avoid future security and maintenance risks."
image: /assets/images/k8s.jpg
---

On November 12, 2025, Kubernetes SIG Network and the Security Response Committee announced that Ingress NGINX will be retired, with best-effort maintenance continuing until March 2026. After that date there will be no releases, no bug fixes, and no security patches. Existing deployments will continue to function, and installation artifacts will remain available — but running an unmaintained ingress controller in production carries real security and compliance risks.

<div style="text-align: center;">
<img src="/assets/images/k8s.jpg" alt="Ingress NGINX Is Being Retired — What Kubernetes Users Should Do Next - chandanadev.com"/>
</div>

## Why This Matters

Ingress NGINX was one of the earliest and most widely adopted ingress controllers for Kubernetes. Its flexibility and feature breadth made it popular across clouds and self-hosted clusters. Over time, however, the very flexibility that made it powerful (for example, arbitrary nginx snippets via annotations) introduced security and maintainability challenges. Combined with chronic understaffing of maintainers, these issues led SIG Network and the Security Response Committee to retire the project to protect users.

## What Changes After March 2026

- **GitHub repositories will be made read-only** — no new pull requests or issues
- **No new releases or security patches** — vulnerabilities will remain unpatched
- **Existing artifacts remain available** — Helm charts and container images stay accessible for reference
- **Running deployments continue working** — but without fixes for newly discovered vulnerabilities

## Recommended Next Steps

### 1. Assess Your Clusters Now

Check whether Ingress NGINX is present with:

```bash
kubectl get pods --all-namespaces --selector app.kubernetes.io/name=ingress-nginx
```

If this returns pods, you need a migration plan.

### 2. Plan Migration to Gateway API (Recommended)

Gateway API is a modern, extensible standard designed to replace many Ingress use cases with clearer semantics and stronger extension points. It offers:

- **Better resource model** — separation of concerns between infrastructure and application teams
- **More expressive routing** — header-based routing, weighted traffic splitting, and more
- **Standardized extensions** — vendor-specific features through a common interface
- **Enhanced security** — built-in security boundaries and role-based access control

### 3. Evaluate Maintained Ingress Controllers

If Gateway API isn't a fit immediately, choose a maintained ingress controller from:

- **Cloud vendor options** — AWS Load Balancer Controller, GKE Ingress, Azure Application Gateway
- **Active open-source projects** — Traefik, HAProxy Ingress, Contour, Emissary-Ingress
- **Commercial solutions** — NGINX Plus, Kong Gateway, Ambassador Edge Stack

### 4. Test and Rollout Gradually

- Start with **staging clusters** to validate functionality
- Verify **traffic routing**, TLS termination, and rate-limiting rules
- Test **failure scenarios** and rollback procedures
- Promote to production **incrementally** by namespace or application

### 5. Eliminate Unsafe Config Patterns

During migration, remove or reduce reliance on:

- Arbitrary nginx snippets in annotations
- Custom configuration that bypasses controller validation
- Undocumented or deprecated annotation patterns

## Migration Timeline Recommendation

| Phase | Timeline | Action |
|-------|----------|--------|
| **Assessment** | Now - December 2025 | Inventory all clusters, document current configurations |
| **Planning** | December 2025 - January 2026 | Choose target solution, create migration runbooks |
| **Testing** | January - February 2026 | Validate in non-production environments |
| **Production Migration** | February - March 2026 | Rollout to production before support ends |

## A Note of Thanks

Ingress NGINX powered billions of requests and helped shape Kubernetes networking. Thanks to the maintainers and contributors for years of dedicated work — their efforts made modern Kubernetes adoption possible for many teams.

## Final Thoughts

This retirement is a call to action: treat it as both a risk to be mitigated and an opportunity to modernize your cluster networking. Teams that begin migration now will reduce technical debt, improve security posture, and benefit from newer, more maintainable networking APIs.

The Kubernetes ecosystem is evolving, and this change reflects the community's commitment to security and sustainability. Start your migration planning today to ensure uninterrupted, secure service delivery well beyond March 2026.

---

**Additional Resources:**

- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [Kubernetes Ingress Controllers Comparison](https://kubevela.io/docs/reference/addons/ingress-controllers)
- [Ingress NGINX Migration Guide](https://kubernetes.github.io/ingress-nginx/)

*Have questions about migrating from Ingress NGINX? Reach out in the comments or connect with me on social media.*

