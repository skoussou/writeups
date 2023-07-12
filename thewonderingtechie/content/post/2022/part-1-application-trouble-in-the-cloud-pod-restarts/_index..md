+++
author = "Stelios Kousouris"
title = "Part 1 - Application Troubles in the Cloud (Symptom: Kubernetes POD Failures or often Restarts)"
date = "2022-03-14"
description = "Part 1 of 3 troubleshoot application failures in the OpenShift container platform"
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

As a result in this and the following blog posts I will explore the 3 most common symptoms 

* [Part 1 - Application Troubles in the Cloud (Symptom: Kubernetes POD Failures/Restarts)](https://www.wonderingtechie.com/post/2022/part-1-application-trouble-in-the-cloud-pod-restarts/)
* [Part 2 - Application Troubles in the Cloud (Symptom: Performance Problems whilst part of a Service Mesh)](https://www.wonderingtechie.com/post/2022/part-2-application-trouble-in-the-cloud-servicemesh-app-performance/)
* [Part 3 - Application Troubles in the Cloud (Symptom: Application Functional Issues)](https://www.wonderingtechie.com/post/2022/part-3-application-trouble-in-the-cloud-application-functional-issues/)

that they were most often faced with. Based on the symptom we will suggest areas to focus as you try to verify the cause and finally will also provide focused troubleshooting and possible remediation actions to take. 

## Possible Causes Of POD Failures or Restarts

The first symptom the platform maintenance personnel came up against was either an Application POD complete failure or continuous restarts. In this case the possible causes (without particular order or priority) could be :

* [Cause 1: Cluster health related](#cause-1---cluster-health-related)
* [Cause 2: Cluster configuration related](#cause-2---cluster-configuration-related)
* [Cause 3: Application or Application environment related](#cause-3---application-or-application-environment-related)
* [Cause 4: Deployment configuration related](#cause-4---deployment-configuration-related)

Below, we present one by one the causes along with the possible areas to focus for each, the information which should be gathered and looked upon. The commands to perform the troubleshooting are summarized in the [openshift-service-mesh-application-troubleshooting](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc) repository.

### Cause 1 - Cluster health related 

Any application failure may not be an issue only of the application and it has to be reviewed in correlation with the platform health. If the OpenShift Cluster cannot offer the requested resources CPU/RAM/Storage by the POD then we need to focus on whether the *cluster state* is the expected ([checking at the Cluster Events](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cluster-events), [verify the Cluster Router state](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cluster-router), [inspecting the Registry](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#internal-registry)) and if it can offer the expected *cluster resources* ([CPU & RAM Nodes Inspection](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cpu-ram-nodes-inspection)). 

Furthermore, the health of the cluster and in extension the cause behind the failures could be related to *Worker Nodes Failing*, *Stopped* or even *Missing* ([check the Machine Configs & State](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#nodes-machine-configs-and-states)), like it happened in our project when the underlying infrastructure had moved some worker nodes to a different cluster. This could result in a reduction of cluster resources leading to resource contention but also could fail to offer labeled nodes for the deployments that require specific node type availability.

Finally, the state of the persistence layer is closely related to the health of the overall cluster as both cluster and application utilize it for their own storage purposes. Any *Persistence Layer Failures* will result in failures to offer adequate storage and so failed PODs ([check the PersistentVolume and PersistentVolumeClaim resources State](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#persistentvolume-and-persitentvolumeclaim-state)), In addition,  cluster storage class configurations misalignments ([check for the Persistence StorageClass availability](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#storageclass-availability-configuration)) could also result in failures for the application deployments.

### Cause 2 - Cluster configuration related

Another possible reason behind the application failure can be the definition of POD replicas or Deployment CPU/RAM resource request/limits beyond allowed limits. This can be caused due to `ResourceQuota` or `LimitRange` resources ([check if ResourceQuota & LimitRange applied on cluster/namespace](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#check-resource-quotas-limit-ranges)). If the deployment of the application is not inline with these limits or the combined namespace requests exceed it then it could be the cause of the POD failed start.

Additionally, another area of focus for the failures are possible Networking failures or misalignments in the cluster which could result in the *failure to pull required images from the registry* ([for this check at the Cluster Events](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cluster-events)) or *failure to connect to services on other worker nodes* ([check Node Calling/Called PODs Deployed and if CLUSTER n/w connectivity is available](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-deployment-location)). The result would be that the deployment of the POD would cease at this point.

`PersistentVolume` and/or `StorageClass` issues as mentioned above can be behind the POD failure to start. Inspect cluster configuration issues resulting in *`PeristentVolume` not bounding* due to persistence layer configuration problems ([by checking the PersistentVolume and PersistentVolumeClaim State](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#persistentvolume-and-persitentvolumeclaim-state)) as well as possible  *`PeristentVolume` config failures to locate the defined or default `StorageClass`*  (performing checks [in the Cluster Events](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cluster-events) and [the PersistentVolume and PersistentVolumeClaim State](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#persistentvolume-and-persitentvolumeclaim-state)). Finally, it could be that there is *no storage left* either on the PV or in the persistence layer (again [check the PersistentVolume and PersistentVolumeClaim State](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#persistentvolume-and-persitentvolumeclaim-state)).


### Cause 3 - Application or Application environment related

Often applications fail because dependent Services are unavailable and this can be caused due to the environment setup eg. *networking access to those services is not available* ([check Node Calling/Called PODs Deployed and if CLUSTER network connectivity is available](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-deployment-location) and [check if the POD is part of the Service Mesh](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-is-in-the-service-mesh)) or a *misplaced environmental configuration* has not allowed them to be instantiated correctly ([Check the deployments and service mesh configurations with service mesh observability](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#service-mesh-observability)).

In addition, the cause could be that the containerized application fails completely due to a problem with the application itself in which case we need to follow standard debugging techniques (first starting by familiarising with [How do I debug an application that fails to start up?](https://cookbook.openshift.org/logging-monitoring-and-debugging/how-do-i-debug-an-application-that-fails-to-start-up.html), then
[checking the Application Logs for application errors](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#application-logs), 
inspecting the [Cluster Events](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cluster-events) and 
[the Registry](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#internal-registry) for reported errors).

Labeling, Scheduling, Taints & Toleration misalignments are further reasons the environment of deployment can hinder POD deployments. Scheduling with Affinity/Anti-Affinity will require POD based affinity and anti-affinity rules to be in place (you can check [rules for placement of POD based on Affinity and Anti-Affinity](https://docs.openshift.com/container-platform/4.9/nodes/scheduling/nodes-scheduler-pod-affinity.html) as well as [Node affinity rules](https://docs.openshift.com/container-platform/4.9/nodes/scheduling/nodes-scheduler-node-affinity.html)). Furthermore, we noticed on certain occasions that the configurations fail to point to the correct `POD` or `Service` therefore focus needs to be given on validating their selectors (by peforming [selectors validation to ensure OCP configs are as expected](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#selectors-validation)) as well as Node Selectors (by peforming [checks on the existence/matching of POD to Node defined selectors](https://docs.openshift.com/container-platform/4.9/nodes/scheduling/nodes-scheduler-node-selectors.html)). Finally, check the [Taints and Tolerations](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-defined-tolerations) which may influence the POD placement.


### Cause 4 - Deployment configuration related

The final cause we will explore, which can be behind the failures or restarts, is the `Deployment` configuration. In these configurations failed `Readiness` and `Liveness` Health Checks due to replicas expected Not Equal to started PODs (identified by [checking if the # of Deployed PODs is correct](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#pod-replicas-desiredcreated)) or *failures due to timeouts* ([inspected at the Cluster Events](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cluster-events)) need to be investigated to determine the causes of the failures.

Sometimes the cause is just erroneous deployment configs such as for *failing init-containers* (seen [at the Cluster Events](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cluster-events) or by [checking the Application Logs](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#application-logs)), or *expected images not available in registry/targeted* repository (again seen [at the Cluster Events](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#cluster-events)). *Erroneous Environment Variable Configurations* should also be explored although this is not easily identifiable other than by checking for errors in the [application logs](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/TROUBLESHOOTING-ACTIONS.adoc#application-logs) and cross-check expected environment variables based on application documentation

The deployment can also suffer from the earlier mentioned `PersistentVolume` and `StorageClass` configuration issues as the selected in the deployment configuration `StorageClass` may not offer auto creation of a `PersistentVolume` or the `StorageClass` does not offer PV with required by `Deployment` mode (ReadWriteMany etc.)

## Conclusion

Application failures in an OpenShift cluster can be caused by many different reasons. In this post we have focused on the cluster’s health and configurations which when compared to the expected application configurations can establish the cause of these failures. 

The combined resources around exploring troubleshooting this symptom can also be found at [Application POD does not start or continuously restarts](https://github.com/skoussou/openshift-service-mesh-application-troubleshooting/blob/main/CLUSTER-HEALTH.adoc).

