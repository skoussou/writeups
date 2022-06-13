+++
author = "Stelios Kousouris"
title = "Red Hat Openshift Service Mesh - Federation Automated Setup"
date = "2022-06-13"
thumbnail = "/images/20220613/automating-ossm-federation-setup.jpg"
shareImage = "/images/20220613/automating-ossm-federation-setup.jpg"
featureImage = "/images/20220613/automating-ossm-federation-setup.jpg"
description = "The article shows delivers an automated setup of federation and federated services on Red Hat Openshift Service Mesh (OSSM)"
tags = [
    "ossm",
    "ServiceMesh",
    "OpenShift"
]
categories = [
    "ServiceMesh"
]
toc = true
years = "2022"
+++

## Service Mesh Federation

Sometimes requirements (high-availability, multi-region solutions) dictate the split a solution deployments over multiple clusters. This brings the question how should we discover and call these services from within Openshift Container Platform deployed workloads without explicit definition of the deployment location as far as the calling service is concerned. Red Hat Openshift Service Mesh (OSSM) based on Istio offers Federation of services to allow to export/import remote service as local.

Service Mesh federation functionality and configurations are described in detail in the [documentation](https://docs.openshift.com/container-platform/4.9/service_mesh/v2x/ossm-federation.html). The following video presents an automated federation setup which you can find in this [Federation Demo Automation github repository](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-Platform-1-Federation) should you wish to try it out.

{{< youtube id="USrTSixYd80" autoplay="true" title="Automated Red Hat Openshift Service Mesh setup" >}}




