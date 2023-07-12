+++
author = "Stelios Kousouris"
title = "Red Hat OpenShift Service Mesh (OSSM) - Security options with mTLS for egress edge traffic"
date = "2022-08-11"
thumbnail = "/images/20220811/mTLS-egress-edge-security-options.jpg"
shareImage = "/images/20220811/mTLS-egress-edge-security-options.jpg"
featureImage = "/images/20220811/mTLS-egress-edge-security-options.jpg"
description = "The article lists the current options to setup `mTLS` with external (to the mesh) services"
tags = [
    "ossm",
    "ServiceMesh",
    "OpenShift",
    "mtls"
]
categories = [
    "ServiceMesh"
]
toc = true
years = "2022"
+++

The Service Mesh provides the option to boost application security with `mTLS` enabled traffic between _in mesh_ application components and external (_to the mesh_) services. In the context of Service Mesh `eggress` edge traffic there are several configuration options for this security feature and in this article we have clearly listed below the options (including options for _clear text_ traffic) along with a linked example to get hands-on expertise with each scenario.

{{<table "table table-striped table-bordered">}}
|#          |Scenario                                                                                                        |Notes     |
|:----------|:---------------------------------------------------------------------------------------------------------------|:----------|
| 1         | **Secure traffic to another external service (could be another Service Mesh not managed with OSSM Federation)**    |           |
| 1a        | [Encrypted External Traffic via sidecar container](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#option1aencrypted)  |           |
|           | [Variation 1: Un-Encrypted External Traffic via sidecar container (mTLS internal traffic PERMISSIVE)](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#option1aunencryptedpermissive)  |          |
|           | [Variation 2: Un-Encrypted External Traffic via sidecar container (mTLS internal traffic STRICT)](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#option1aunencryptedstrict)  |          |
| 1b        | [Encrypted External Traffic via egress gateway container](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#option1bencrypted)  |          |
|           | [Un-Encrypted External Traffic via egress gateway container](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#:~:text=Un%2DEncrypted%20External%20Traffic%20via%20egress%20gateway%20container)  |          |
| 1c        | [Encrypted External Traffic directly from the App container](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#option1aencryptedfromapp)  |          |
| 2         | **Secure Traffic to an external service on another federated Service Mesh (OSSM)**  |          |
| 2a        | [Encrypted federated Traffic directly from the Application](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#option2adirectenctrypted)  | **Impossible by Design** |
|           | [Encrypted federated Traffic directly from the Application](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#option2adirectunenctrypted)  | **Impossible by Design** (Unencrypted traffic is not possible in Federation by design |
| 2b        | [Encrypted federated Traffic via egress gateway](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#:~:text=2b-,Encrypted%20federated%20Traffic%20via%20egress%20gateway,-Ready)  |          |
|           | [Encrypted federated Traffic via egress gateway](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-3-SM-Service-To-External-MTLS-Handling#option2begressunenctrypted)  | **Impossible by Design** (Unencrypted traffic is not possible in Federation by design |
{{</table>}}

