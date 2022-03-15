+++
author = "Stelios Kousouris"
title = "Part 2 - Application Troubles in the Cloud (Symptom: Performance Problems whilst part of a Service Mesh)"
date = "2022-03-15"
description = "Part 2 or 3 troubleshoot application failures in the OpenShift container platform"
tags = [
    "Troubleshooting",
    "CloudApps",
    "OpenShift"
]
categories = [
    "OpenShift"
]
series = "cloud-apps-troubleshoot"
toc = true
draft = true
years = "2022"
+++

## Introduction

I have recently supported a customer to deploy numerous applications over multiple Openshift clusters running on top of hundreds of worker nodes, using external storage infrastructure for persistence and Service Mesh (OSSM) for authentication and traffic management. 

The method to deliver in this environment did not differ too much from delivering the applications in any other environment, with automation involved throughout, however when an application issue was reported for many of the platform maintenance team members it was not possible to hone in quickly on the area they should focus on. As a result it was not immediately obvious what information, where from and how to collect information that would assist in the problem resolution.

In [Part 1 - Application Troubles in the Cloud (Symptom: Kubernetes POD Failures/Restarts)](www.wonderingtechie.com/post/part-1-application-trouble-in-the-cloud-pod-restarts/) blog post I looked at the cluster health and state in order to resolve POD startup failures, in this second blog I will focus on the Performance Issues that may be caused by Service Mesh Setup, Configuration and Health. 

* [Part 1 - Application Troubles in the Cloud (Symptom: Kubernetes POD Failures/Restarts)](www.wonderingtechie.com/post/part-1-application-trouble-in-the-cloud-pod-restarts/)
* [Part 2 - Application Troubles in the Cloud (Symptom: Performance Problems whilst part of a Service Mesh)](www.wonderingtechie.com/post/part-2-application-trouble-in-the-cloud-servicemesh-app-performance.md)
* Part 3 - Application Troubles in the Cloud (Symptom: Application Functional Issues)


