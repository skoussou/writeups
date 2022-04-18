+++
author = "Stelios Kousouris"
title = "Part 2 - Application Troubles in the Cloud (Symptom: Performance Problems whilst part of a Service Mesh)"
date = "2022-03-21"
description = "Part 2 of 3 troubleshoot application failures in the OpenShift container platform"
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
years = "2022"
+++

## Introduction

I have recently supported a customer to deploy numerous applications over multiple OpenShift clusters running on top of hundreds of worker nodes, using external storage infrastructure for persistence and [Red Hat Service Mesh (OSSM)](https://docs.openshift.com/container-platform/latest/service_mesh/v2x/ossm-architecture.html) for encryption, authentication and traffic management. 

The method to deliver in this environment did not differ too much from delivering the applications in any other environment, with automation involved throughout, however when an application issue was reported for many of the platform maintenance team members it was not possible to hone in quickly on the area they should focus on. It was not immediately obvious what information would assist in the problem identification and resolution or where from and how to collect them.

In [Part 1 - Application Troubles in the Cloud (Symptom: Kubernetes POD Failures/Restarts)](https://www.wonderingtechie.com/post/2022/part-1-application-trouble-in-the-cloud-pod-restarts/) blog post we looked at the cluster health state in order to resolve POD startup failures, in this second blog I will focus on *Performance Problems* that may be caused by a Service Mesh setup, configuration and health. 

* [Part 1 - Application Troubles in the Cloud (Symptom: Kubernetes POD Failures/Restarts)](https://www.wonderingtechie.com/post/2022/part-1-application-trouble-in-the-cloud-pod-restarts/)
* [Part 2 - Application Troubles in the Cloud (Symptom: Performance Problems whilst part of a Service Mesh)](https://www.wonderingtechie.com/post/2022/part-2-application-trouble-in-the-cloud-servicemesh-app-performance/)
* [Part 3 - Application Troubles in the Cloud (Symptom: Application Functional Issues)](https://www.wonderingtechie.com/post/2022/part-3-application-trouble-in-the-cloud-application-functional-issues/)


## Causes of Application Performance Problems in Service Mesh

The second symptom the platform maintenance personnel came up against were various kind of performance failures related to connecting to or from within the Service Mesh. The result was that responses were being delayed and at times eventually failed. Troubleshooting such issues we will focus on two possible causes:

* [Cause 1: Failed Connections](#cause-1---failed-connections)
* [Cause 2: Slow Responses](#cause-2---slow-responses)

Below we present one by one the causes along with the possible areas to focus for each, as well as the information which should be gathered and looked upon.

### Cause 1 - Failed Connections 

In our use case all traffic was secured by exercising the [Mutual TLS](https://github.com/maistra/api/blob/maistra-2.1/docs/crd/maistra.io_ServiceMeshControlPlane_SecurityConfig_v2.adoc) (`mTLS`) authentication between caller and services. Service Mesh based services exposed externally receive their own certificates to present to the caller (with a `passthrough` OCP `Route` declaring a DNS resolvable service `hostname` and a `Gateway` resource defining the `secret` hosting the certificate for that hostname). This brings us to the first set of possible failures related to `mTLS` configurations. Troubleshooting should focus on Client/Service Mesh certificate configurations, for that [check the defined Service Mesh certificates](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-certificates) and [verify Service Mesh external (in/out) network configurations](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-external-inout-network-configurations), as there can be misalignments on presenting or accepting the correct certificates or pointing to the correct service. Similarly, outgoing communications could also suffer from misconfigurations of egress certificates ([check the applied certificates](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-certificates) and [egress resource configurations](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-external-inout-network-configurations)). In addition to *authentication* some *authorisation* failures could be caused by the Service Mesh security settings, hence exploring those configuration settings applied in [`ServiceMeshControlPlane`](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-certificates) resource is deemed necessary.

Furthermore, special focus must be given on possible network issues, resulting from the service mesh configurations, which can cause certain locality deployment expectations of the applications to be broken ([check based on the involved PODs the topology of the nodes calls originate from/are received at to verify n/w connectivity is viable](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-deployment-location)). In addition, the cause could be that the calling service is not in a service mesh but called service is ([check the PODs are in the same Service Mesh](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-is-in-the-service-mesh) and [if they are in a different Service Mesh check the certificates used](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-certificates)) or that the calling service is part of a different service mesh ([check on which Service Mesh the service is a member of](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-is-in-the-service-mesh)). Further checks should be performed that the [OCP Route exposing the service exists](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-certificates) or that the [service mesh configs render it accessible](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-external-inout-network-configurations) and finally it should be validated that the Service Mesh envoy sidecar has the correct/up-to-date configurations ([first check the Service Mesh Operator State](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#anchor-1), and if the cause still unknown [check the Service Mesh Configuration](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-deep-dive-troubleshooting-actions-envoyistio-proxy)).

### Cause 2 - Slow Responses 

Application can suffer performance degradation because `POD` or `Deployment` resource limits have been reached. Firstly, ensure that CPU/RAM/Network/Storage POD Limits have not been exceeded ([inspect POD Resources Usage](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-resources-usage)) and that `Deployment` state (`replicas`) is the expected ([by checking if the # of Deployed PODs is correct](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-replicas-desiredcreated)) . If there are issues then check to ensure that any set [Resource Limits are not exceeded](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#check-resource-quotas-limit-ranges). 

If it is not clear from reviewing the resources on what could be causing the applications to fail or to start performing erratically, look at the [cluster events](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cluster-events) for the possible causes behind [POD restarts](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-restarts). The [application](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#application-logs) and [service mesh logs](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#check-set-logging-levels-of-service-mesh-components) can also reveal if the *logging level* is behind the performance degradation or if some other functional cause is behind backed up requests towards a failing/slow service. The latter can be further investigated utilizing the Service Mesh observability stack ([validate configuration/errors with KIALI](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-observability) to ensure for instance that retries occur but persisting constantly.

## Conclusion

Application failures can be caused by misconfigurations in many levels from the network to the client, the service mesh or the application itself. The result of these misconfigurations could potentially result in slow response rate, reduced throughput, timeouts etc. All these will degrade the performance of the overall solution and in this post WE focused on what can be the possible causes of these performance related issues when the application is part of the mesh or communicates to a service in a mesh. 

The combined resources around exploring troubleshooting around this symptom can also be found at [Application (in Service Mesh) performance problems](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/APPLICATION-PERFORMANCE.adoc).



