layout: post
title: "Ingress NGINX Is Being Retired — What Kubernetes Users Should Do Next"
date: 2025-11-18
author: "Chandana Nawarathna"
categories: [kubernetes, cloud-native, networking]
tags: [Kubernetes, IngressNGINX, GatewayAPI, DevOps, SRE, Security]
excerpt: "Ingress NGINX will be retired in March 2026. Start planning migration to Gateway API or another maintained ingress controller now to avoid future security and maintenance risks."
Ingress NGINX Is Being Retired — What Kubernetes Users Should Do Next

On November 12, 2025, Kubernetes SIG Network and the Security Response Committee announced that Ingress NGINX will be retired, with best-effort maintenance continuing until March 2026. After that date there will be no releases, no bug fixes, and no security patches. Existing deployments will continue to function, and installation artifacts will remain available — but running an unmaintained ingress controller in production carries real security and compliance risks.

<div style="text-align: center;">
<img src="/assets/images/k8s.jpg" alt="Ingress NGINX Is Being Retired — What Kubernetes Users Should Do Next - chandanadev.com"/>
</div>

Why this matters

Ingress NGINX was one of the earliest and most widely adopted ingress controllers for Kubernetes. Its flexibility and feature breadth made it popular across clouds and self-hosted clusters. Over time, however, the very flexibility that made it powerful (for example, arbitrary nginx snippets via annotations) introduced security and maintainability challenges. Combined with chronic understaffing of maintainers, these issues led SIG Network and the Security Response Committee to retire the project to protect users.

What changes after March 2026

GitHub repositories will be made read-only.

There will be no new releases or security patches.

Existing Helm charts and container images will remain available for reference.

Existing Ingress NGINX deployments will continue to run, but will not receive fixes for newly discovered vulnerabilities.

Recommended next steps

Assess your clusters now.
Check whether Ingress NGINX is present with:

kubectl get pods --all-namespaces --selector app.kubernetes.io/name=ingress-nginx


If this returns pods, you need a migration plan.

Plan migration to Gateway API (recommended).
Gateway API is a modern, extensible standard designed to replace many Ingress use cases with clearer semantics and stronger extension points.

Evaluate maintained ingress controllers.
If Gateway API isn’t a fit immediately, choose a maintained ingress controller from your cloud vendor or an active open-source project.

Test and rollout gradually.
Start with staging clusters, validate traffic, TLS, and rate-limiting rules before promoting to production.

Eliminate unsafe config patterns.
During migration, remove or reduce reliance on arbitrary nginx snippets and other patterns that introduce security risk.

A note of thanks

Ingress NGINX powered billions of requests and helped shape Kubernetes networking. Thanks to the maintainers and contributors for years of dedicated work — their efforts made modern Kubernetes adoption possible for many teams.

Final thoughts

This retirement is a call to action: treat it as both a risk to be mitigated and an opportunity to modernize your cluster networking. Teams that begin migration now will reduce technical debt, improve security posture, and benefit from newer, more maintainable networking APIs.
