+++
author = "Stelios Kousouris"
title = "mTLS in the Mesh with Redhat OpenShift Service Mesh (OSSM) - On or Off?"
date = "2022-05-10"
thumbnail = "/images/20220510/mTLS-in-the-Mesh-with-OSSM-OnorOff.jpg"
shareImage = "/images/20220510/mTLS-in-the-Mesh-with-OSSM-OnorOff.jpg"
featureImage = "/images/20220510/mTLS-in-the-Mesh-with-OSSM-OnorOff.jpg"
description = "The article goes over examples where mTLS configuration on Redhat OpenShift Service Mesh (OSSM) settings do not completely turn off mTLS, when it would be needed to be turned off or turned on and how one would determine whether traffic is encrypted"
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

## Introduction

One of the benefits of a service mesh is security by design, in fact it is one of the main use cases which drives adoption of the mesh in cloud based environments where there are disparate types of microservice workloads. 

Although this is very powerful as one can imagine there may be cases that a workload cannot participate in mTLS handshakes using the default mesh certificates. This is due to a workload having to handle its own TLS termination or because simply it should not receive TLS traffic (performance or policy reasons). However, what are the possible security settings and when do they result in an **on** or **off** mTLS traffic setup? Like in the case of a recent customer of mine this can be a little confusing.

In this article we will explore how encryption with Mutual TLS (mTLS) is configured **on** or **off** in [Red Hat OpenShift Service Mesh (OSSM)](https://www.redhat.com/en/technologies/cloud-computing/openshift/what-is-openshift-service-mesh) and how you can verify if the traffic is encrypted or not.


## Setting up the Service Mesh controlplane

In order to setup the Red Hat Service Mesh you would need either a

* Local [CodeReady Containers (CRC)](https://access.redhat.com/documentation/en-us/red_hat_codeready_containers/1.34/html/getting_started_guide) cluster or
* an [OpenShift Clusters (4.6+)](https://www.redhat.com/en/technologies/cloud-computing/openshift/try-it) 
and
* `oc` [binary](https://downloads-openshift-console.apps.cluster-wwt8j.wwt8j.sandbox1899.opentlc.com/) installed in the local path for the version of OCP

The next step is to deploy in the cluster the necessary [operators the service OSSM requires](https://docs.openshift.com/container-platform/4.9/service_mesh/v2x/installing-ossm.html) for the deployment and configuration of the mesh. We have prepared such a script [`add-operators-subscriptions-sm.sh`](https://github.com/skoussou/servicemesh-playground/blob/main/scripts/add-operators-subscriptions-sm.sh) and all you require is to login to your cluster and execute (in a linux based system) this script or alternatively copy the contents of it and execute them with the `oc` binary.

Finally, the OSSM operator will require a `ServiceMeshControlPlane` (SMCP) resource to define the characteristics of the Service Mesh (both for the *controlplane* and *dataplane*). We will create it with the following commands.

{{< highlight go "linenos=inline,hl_lines=1 2 9-11 35 38,linenostart=0" >}}
oc new-project istio-system (1)
echo "apiVersion: maistra.io/v2 (2)
kind: ServiceMeshControlPlane
metadata:
  name: basic
  namespace: istio-system
spec:
  security:
    dataPlane:
      automtls: false
      mtls: false
  addons:
    grafana:
      enabled: true
    jaeger:
      install:
        storage:
          type: Memory
    kiali:
      enabled: true
    prometheus:
      enabled: true
  policy:
    type: Istiod
  profiles:
    - default
  telemetry:
    type: Istiod
  tracing:
    sampling: 10000
    type: Jaeger
  version: v2.1"| oc apply -n istio-system -f -

oc get smcp -n istio-system
NAME    READY   STATUS           PROFILES      VERSION   AGE (3)
basic   7/10    PausingInstall   ["default"]             18s

oc wait --for condition=Ready -n istio-system smcp/basic --timeout 300s (4)
servicemeshcontrolplane.maistra.io/basic condition met
{{< / highlight >}}  

1. Creates the OCP namespace (project) where the controlplane components will be deployed in.
2. The `SMCP` which defines that dataplane **mTLS** (mutual TLS encryption between mesh included workloads) is **false**. It is key to understand what the effect of this is and we will explain it below.
3. The OSSM operator begins to configure the controlplane and when all components are ready it is ready (STATUS: `ComponentsReady`) to be used
4. The OSSM operator begins to configure the controlplane and when conditions are met it is ready to be used

After applying the above we will have a Service Mesh *controlplane* which manages the observability stack as well as policies and configurations applied to the *dataplane* and we now need to define the *dataplane*.


## Setting up the dataplane

Next apply the commands in the following link to create a namespace and deploy the bookinfo [application](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-0-Deploy-In-ServiceMesh#bookinfo). Once that is in place you should have all PODs in the bookinfo namespace started.

{{< highlight go "linenostart=0" >}}
oc get pods -n bookinfo
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-6cd699df8c-nnbtx       2/2     Running   0          14s
productpage-v1-5ddcb4b84f-6jlf8   2/2     Running   0          9s
ratings-v1-bdbcc68bc-7vjtj        2/2     Running   0          13s
reviews-v1-754ddd7b6f-d4xxj       2/2     Running   0          12s
reviews-v2-675679877f-b78qm       2/2     Running   0          12s
reviews-v3-79d7549c7-xdsjj        2/2     Running   0          11s
{{< / highlight >}}

Add to each of the `Deployment` under `bookinfo` namespace the following annotation in order to register statistics by `istio-proxy` on TLS handshakes:

{{< highlight go "linenostart=0" >}}
oc patch deployment productpage-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/statsInclusionPrefixes": "tls_inspector,listener,cluster"}}}}}' -n  bookinfo
oc patch deployment details-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/statsInclusionPrefixes": "tls_inspector,listener,cluster"}}}}}' -n  bookinfo
oc patch deployment ratings-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/statsInclusionPrefixes": "tls_inspector,listener,cluster"}}}}}' -n  bookinfo
oc patch deployment reviews-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/statsInclusionPrefixes": "tls_inspector,listener,cluster"}}}}}' -n  bookinfo
oc patch deployment reviews-v2 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/statsInclusionPrefixes": "tls_inspector,listener,cluster"}}}}}' -n  bookinfo
oc patch deployment reviews-v3 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/statsInclusionPrefixes": "tls_inspector,listener,cluster"}}}}}' -n  bookinfo
{{< / highlight >}}

As soon as the re-deployment of the PODs completes the *dataplane* is ready and the following command should verify this.

{{< highlight go "linenos=inline,hl_lines=2, linenostart=0" >}}
curl -s "http://$(oc get route istio-ingressgateway -o jsonpath='{.spec.host}' -n istio-system)/productpage" | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
{{< / highlight >}}


## Disabling mTLS for all applications 

As mentioned above we need to have a clear understanding of the `SMCP` mTLS settings and their effect. The result of the above applied settings for `mtls: false` is that OSSM creates in the *controlplane* namespace a `PeerAuthentication` resource with mTLS mode set to `PERMISSIVE`. The effect of this configuration is that if 2 workloads participating in communication in the mesh can perform mTLS handshake then the mesh will enforce it. 

{{< highlight go "linenostart=0" >}}
oc get peerauthentication -n istio-system
NAME          MODE         AGE
default       PERMISSIVE   5d18h
{{< / highlight >}}

This is then inherited by all *dataplane* namespaces and this came as a surprise to one of our customers who expected that `mtls: false` meant no mTLS handshakes. 

You can verify this whilst calling the URL

{{< highlight go "linenostart=0" >}}
watch curl -s "http://$(oc get route istio-ingressgateway -o jsonpath='{.spec.host}' -n istio-system)/productpage" | grep -o "<title>.*</title>"
{{< / highlight >}}

TLS handshakes will take place between the components and statistics will be registered in the counters (see the numbers on the stats increasing executing the [test-ssl-handshakes.sh](https://github.com/skoussou/servicemesh-playground/blob/main/Scenario-MTLS-4-Turn-Off-MTLS/test-ssl-handshakes.sh) script per POD).

{{< highlight go "linenostart=0" >}}
./test-ssl-handshakes.sh productpage-v1-556db7cbb5-x5n55	<-- HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh details-v1-68cbd47bc5-xwf2x		<-- HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh reviews-v1-75755d569f-z6jwf		<-- HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh reviews-v2-86c76b84c5-xzq56		<-- HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh reviews-v3-56cbff6b99-cfwj4		<-- HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh ratings-v1-5f867c4bb7-7fdv8		<-- HANDSHAKES TAKE PLACE
{{< / highlight >}}

You can further verify this by accessing the KIALI UI (url: `oc get route kiali -o jsonpath='{.spec.host}' -n istio-system`) *App Graph* and in the display drop down select *Security*. You should note the padlock icon appears in all the arrows between the `bookinfo` components.

So how can we then completely disable mTLS in the communications for the traffic in the *dataplane*? The answer is by applying `DestinationRule` which forces clients of a service (host) to not apply mTLS in the communication towards/from that host. This is what the following commands will do after you apply them.

{{< highlight go "linenostart=0" >}}
echo "apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage.bookinfo.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE" |oc apply -n bookinfo -f -

echo "apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: productpage-2
spec:
  host: productpage
  trafficPolicy:
    tls:
      mode: DISABLE" |oc apply -n bookinfo -f -

echo "apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: details
spec:
  host: details.bookinfo.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE" |oc apply -n bookinfo -f -

echo "apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: details-2
spec:
  host: details
  trafficPolicy:
    tls:
      mode: DISABLE" |oc apply -n bookinfo -f -

echo "apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings.bookinfo.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE" |oc apply -n bookinfo -f -

echo "apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ratings-2
spec:
  host: ratings
  trafficPolicy:
    tls:
      mode: DISABLE" |oc apply -n bookinfo -f -

echo "apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews.bookinfo.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE" |oc apply -n bookinfo -f -

echo "apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-2
spec:
  host: reviews
  trafficPolicy:
    tls:
      mode: DISABLE" |oc apply -n bookinfo -f -
{{< / highlight >}}

Testing against the `productpage` application (curl above) we should see no TLS handshakes taking place now.

{{< highlight go "linenostart=0" >}}
./test-ssl-handshakes.sh productpage-v1-556db7cbb5-x5n55	<-- NO HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh details-v1-68cbd47bc5-xwf2x		<-- NO HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh reviews-v1-75755d569f-z6jwf		<-- NO HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh reviews-v2-86c76b84c5-xzq56		<-- NO HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh reviews-v3-56cbff6b99-cfwj4		<-- NO HANDSHAKES TAKE PLACE
./test-ssl-handshakes.sh ratings-v1-5f867c4bb7-7fdv8		<-- NO HANDSHAKES TAKE PLACE
{{< / highlight >}}

KIALI shows a similar behavior (notice no padlock on any of the connections and on the right handside *unknown Principals* on the from/to).

![Service Mesh KIALI UI - No mTLS Security](/images/20220510/no-security-applied.png)

## Enforce mTLS for all applications

In the case of a set of applications that require to be excluded from mTLS the above may make sense. However, when the data is sensitive and policy needs to be very strict around security by encryption the Mesh admin can force all components to communicate via `mTLS` by defining in the `SMCP` Resource the following mTLS settings.

{{< highlight go "linenostart=0" >}}
 security:
    dataPlane:
      automtls: true
      mtls: true
{{< / highlight >}}

You can also follow this by accessing the `SMCP` Resource YAML under namespace `istio-system` and change the existing settings. Alternatively, re-apply the resource at [Setting up the Service Mesh controlplane](#setting-up-the-service-mesh-controlplane) changing the settings for `mtls: true` and `automtls: true`. The outcome is that the `PeerAuthentication` resource will now have mTLS set to `STRICT` mode. 

{{< highlight go "linenostart=0" >}}
oc get peerauthentication -n istio-system
NAME                            MODE         AGE
default                         STRICT       5d18h
{{< / highlight >}}

The effect of this configuration is now any request to the `productpage` will fail.

{{< highlight go "linenos=inline,hl_lines=7, linenostart=0" >}}
$ curl -v "http://$(oc get route istio-ingressgateway -o jsonpath='{.spec.host}' -n istio-system)/productpage" | grep -o "<title>.*</title>"
> Host: istio-ingressgateway-istio-system.apps.cluster-e8e9.e8e9.sandbox866.opentlc.com
> User-Agent: curl/7.71.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 503 Service Unavailable
< content-length: 95
< content-type: text/plain
< date: Wed, 23 Mar 2022 11:10:25 GMT
< server: istio-envoy
< set-cookie: 44371fc75fdb694d574e56e33b166cc7=619f273b9d2709119dd0b6b5b31cdc01; path=/; HttpOnly
{{< / highlight >}}

## mTLS disabled for specific application workloads

Our customer had specific workloads (elastic search, kafka streams etc.) which needed to handle their own TLS termination. In this case we wanted to maintain the `STRICT` policy of mTLS on all traffic except those workloads. To achieve this in the current setup apply the following to disable mTLS ONLY for the `details` service:

{{< highlight go "linenostart=0" >}}
oc delete dr productpage -n bookinfo
oc delete dr productpage-2 -n bookinfo
oc delete dr reviews -n bookinfo
oc delete dr reviews-2 -n bookinfo
oc delete dr ratings -n bookinfo
oc delete dr ratings-2 -n bookinfo
oc delete peerauthentication default-disable -n bookinfo
echo "apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: details-mtls-disable
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: details
  mtls:
    mode: DISABLE" |oc apply -n bookinfo -f -
{{< / highlight >}}

Testing should show the following in KIALI UI where a padlock appears in all connections except the `details` and *Principal* shows content of the certs used on the from/to now) whilst you can also check the [istio-proxy handshake stats](https://github.com/skoussou/servicemesh-playground/tree/main/Scenario-MTLS-4-Turn-Off-MTLS#anchor-1) once more to verify TLS handshakes do take place for all workloads except `details`.

![Service Mesh KIALI UI - No mTLS for details service](/images/20220510/all-but-details-with-mtls.png)

Final thoughts around the behavior of mTLS **on** or **off** in the OSSM
* If `SMCP` Resource config is set to `PERMISSIVE` mTLS the above additional `PeerAuthentication` for details is not required.
* If `SMCP` Resource config is set to `STRICT` mTLS the above additional `PeerAuthentication` for details is required and if removed the result will be as follows

![Service Mesh KIALI UI -  Error for STRICT mTLS when no PeerAuthentication DISABLE is defined](/images/20220510/error-without-peerauthentication-disable.png)


## Conclusion

The Red Hat OpenShift Service Mesh (OSSM) will always apply mTLS encryption to the traffic and the options are if it should be `STRICT` or `PERMISSIVE`. In the `STRICT` case, all traffic participating workloads will be required to have the ability to adhere to this policy or be explicitly excluded through a `PeerAuthentication` disable policy for that service. In the `PERMISSIVE` scenario workloads will participate in mTLS traffic if both parties can and this can be by-passed via a `DestinationRule` for informing all clients of the host of a service to not initiate mTLS connection.

You can try all above examples (and more) in the [servicemesh-playground](https://github.com/skoussou/servicemesh-playground) repository and provide feedback.




