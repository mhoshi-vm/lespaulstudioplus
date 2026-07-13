---
title: "Learning Horizontal Pod Autoscaler with Wavefront Part-1"
date: 2020-08-21T21:30:12+09:00
categories: [Tanzu Observability]
tags: ["Wavefront","Tanzu Observability", "Horizontal Pod Autoscaler", "featured"]
thumbnail: "hpa.png"
featured: true
---

This is the first installment of the Learning Horizontal Pod Autoscaler with Wavefront series.<!--more-->
# Series

Part 1 : Overview **← you are here**  
Part 2 : [Scaling from Wavefront data](../wf-demanabu-hpa-02)  
Part 3 : [Implementing a serverless look-alike](../wf-demanabu-hpa-03)

# What on earth is Wavefront?

Wavefront is a SaaS-based cloud and application monitoring platform.
It has since been acquired by VMware and is called Tanzu Observability.
This series frequently uses the old name Wavefront, but they are the same thing.

# What on earth is the Horizontal Pod Autoscaler?

The Horizontal Pod Autoscaler (HPA) is a Kubernetes feature that monitors the load of deployed PODs and scales them out.

https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

# What's wrong with using HPA normally?

A common misconception about this feature: it is not usable OOTB (Out of the Box) right after installing k8s.

Normally, to use HPA you need some separate tool collecting metrics.
The famous one is metrics-server:

https://github.com/kubernetes-sigs/metrics-server

However, as documented, it only supports CPU and Memory.
Another common problem is the "was the CPU really that high at the time?" problem. At the end of the day the Ops team is watching the monitoring dashboards, and if the CPU/Memory there doesn't match the AutoScale events, it can escalate into needless communication problems.

# What about Prometheus?

Using Prometheus as the HPA source has been covered in overseas blogs recently.
For example:

https://towardsdatascience.com/kubernetes-hpa-with-custom-metrics-from-prometheus-9ffc201991e

But beware: this promotes Prometheus into a component that affects production workloads. If you're not experienced operating Prometheus, or don't have an HA configuration, it's a setup that's hard to recommend.

# So what do we do?

That's where Wavefront comes in.
With the recent Tanzu momentum, Wavefront has been strengthening its kubernetes integration and officially supports an HPA adapter:

https://github.com/wavefrontHQ/wavefront-kubernetes-adapter

It's in English, but this video covers and explains it:

[![IMAGE ALT TEXT HERE](https://i3.ytwatch?v=nZnbdNHFNyU)  

https://www.youtube.com/watch?v=nZnbdNHFNyU


The benefits of using this:

* HPA works off the same metrics the Ops dashboards show
* Freely define any metrics beyond CPU and Memory
* SaaS-grade availability (99.95% Availability) and performance guarantees

# So what is this series?

Having long known HPA existed while it remained one of kubernetes' exotic party tricks, hesitating to touch it — surely that's not just me.

In this series, we configure HPA with Wavefront, a SaaS-supported system, to deepen our understanding of how HPA actually works.

Next: "[Scaling from Wavefront data](../wf-demanabu-hpa-02)".
