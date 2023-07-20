+++
author = "Stelios Kousouris"
title = "CI Implementations with Tekton Resolvers and Pipelines As Code"
date = "2023-07-19"
thumbnail = "/images/20230720/ci-tekton-resolvers-pipelines-as-code.png"
shareImage = "/images/20230720/ci-tekton-resolvers-pipelines-as-code.png"
featureImage = "/images/20230720/ci-tekton-resolvers-pipelines-as-code.png"
description = "The article showcases how to utilize resolvers in tekton pipelines and migrating them to pipelines as code."
tags = [
    "cicd",
    "tekton",
    "OpenShift",
    "devopstools"
]
categories = [
    "cicd"
]
toc = true
years = "2023"
+++


## Overview

<div style="text-align: justify"> 

For sometime now we have been, together with DevOps empowered application teams and DevOps engineers, using the [OpenShift Pipelines](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/understanding-openshift-pipelines.html), OpenShift's productised version of [tekton.dev](https://tekton.dev/), as the Continuous Integration (CI) technology of choice when onboarding new applications or migrating existing ones into Red Hat's kubernetes.

Recently, the article on [Migration from ClusterTasks to Tekton Resolvers in OpenShift Pipelines](https://cloud.redhat.com/blog/migration-from-clustertasks-to-tekton-resolvers-in-openshift-pipelines) by colleagues [Vincent Demeester](https://www.linkedin.com/in/vincentdemeester/) and [Koustav Saha](https://www.linkedin.com/in/koustav-saha-7bb20580/) and the GA of [Pipelines as code](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/op-release-notes.html#compatibility-support-matrix_op-release-notes) prompted me to perform a refresher on the resources used as a baseline when starting a completely new implementation/migration or introducing the topic of tekton to a team. 

In this article we will go through the manner in which we performed a classic CICD release cycle with tekton and gitops as well as how we can overhaul the existing material to give practical substance to all possible manners of utilsing tasks in pipelines beyond installing them as `ClusterTask` or `Task` as well as moving from pipeline webhooks for CI (and CD) to pipeline as code.
</div>

## The CICD and GitOps lifecycle

<div style="text-align: justify"> 

In the accompanying [tekton-gitops-workshop](https://github.com/skoussou/tekton-gitops-workshop) github repository the setup populates:
* the OpenShift Gitops and Openshift Pipeline operators in an OpenShift cluster
* the Gitea Git Repository provider with 2 repositories
  * [application-source](https://github.com/skoussou/tekton-gitops-workshop/tree/main/application-source) repository hosts the application source 
  * [application-deploy](https://github.com/skoussou/tekton-gitops-workshop/tree/main/application-deploy) hosts an ArgoCD `Application` that in turn bootstraps via an `ApplicationSet` the environment based applications for `dev`, `test`, `prod`, delivers and monitors the kubernetes configurations for the application in each environment.
* 2 OpenShift pipelines
  * [ci-pipeline](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/pipeline-ci.yaml): runs the application testing, packaging and containerisation process
  * [cd-pipeline](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/pipeline-cd.yaml): with a PR from the `ci-pipeline` to the `application-deploy` git repo the CD process initiates the process of delivering the new application deployments into OpenShift 
* Various [OpenShift Pipelines objects](https://github.com/skoussou/tekton-gitops-workshop/tree/main/application-cicd/resources) (`Task`, `ClusterTask`, `Pipeline`, `EventListerner`, `TriggerBinding`, `TriggerTemplate`) are installed in order to facilitate the pipeline implementation and triggering.

The resulting setup offers the ability through Git repository updates (as shown in the following diagram) to run through a lifecycle of Continuous Integration and Continuous Delivery with the tekton pipelines whilst the Continuous Deployment is handled by the ArgoCD component and again promoted via git Pull Requests.

![CICD Lifecycle](/images/20230720/CICD-Cycle.png)
</div>

## Migrating from `ClusterTask` and `Task` to `Tekton Resolvers`

<div style="text-align: justify"> 

Between the [article](https://cloud.redhat.com/blog/migration-from-clustertasks-to-tekton-resolvers-in-openshift-pipelines) mentioned earlier, the OpenShift [Remote Pipelines Tasks Resolvers documentation](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/remote-pipelines-tasks-resolvers.html) and [tekton's documentation](https://tekton.dev/docs/pipelines/cluster-resolver/#requirements) there is very good information on how to configure the resolvers, what are the resolver types and purpose of each. We shall concentrate on how we made the changes to the original pipelines to take advantage of the resolver functionality.

It is important to note at this point that the 2 pipelines contain the following type of tasks:

* `ClusterTask` resources
  * `git-clone` (clones a repo from the provided url)
  * `s2i-java` (clones a Git repository and builds and pushes a container image using S2I and a Java builder image).

* Custom `Task` resources
  * [next-env](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/task-next-environment.yaml) (works out the next environment to promote the updated image to)
  * [sync-argo](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/task-sync-argo.yaml) (executes an argocd sync to update the targeted environment resources)
  * [create-pr-to-deploy](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/task-create-pr-to-deploy.yaml) (creates a PR towards the `application-deploy` repository with an updated deployment to the latest produced image)
  * [generate-version](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/task-generate-version.yaml)(generates application version based on timestamp and pom version.)
  * [mock](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/task-mock.yaml)(a task that mocks pipeline activities)
  * [tag-image](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/task-tag-image.yaml)(Tags the image to the generated version)
</div>

### Migrating to `Cluster` Resolvers

<div style="text-align: justify"> 

A cluster resolver is a Tekton Pipelines feature that allows you to reference tasks and pipelines from other namespaces, therefore it can act as a cluster wide known location for DevOps engineers to maintain tasks on behalf of pipeline creators who can in-turn reference them from that namespace. 

In the [cluster-resolver](https://github.com/skoussou/tekton-gitops-workshop/blob/cluster-resolver) branch the setup has been updated with the 2 aforementioned `ClusterTask` resources `git-clone` and `s2i-java`
* implemented as `Task` resources and
* installed in `sk-workshop-ci-components` namespace

The [ci-pipeline](https://github.com/skoussou/tekton-gitops-workshop/blob/cluster-resolver/application-cicd/resources/pipeline-ci.yaml) has been updated to use now the `resolver: cluster` to reference the task from the CI components namespace.
```YAML
  # ------------ CLONE APP SOURCE ------------ #
    - name: git-app-clone
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: git-clone-custom
          - name: namespace
            value: $(params.CI_NS_PREFIX)-workshop-ci-components
   ...
  # ------------ BUILD IMAGE ------------ #
    - name: build-image
      runAfter:
      - nexus-upload
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: s2i-java-custom
          - name: namespace
            value: $(params.CI_NS_PREFIX)-workshop-ci-components   
```
</div>

### Migrating to `Git` Resolvers

<div style="text-align: justify"> 

A [git resolver](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/remote-pipelines-tasks-resolvers.html#resolver-git_remote-pipelines-tasks-resolvers) is a Tekton Pipelines feature that allows you to reference tasks and pipelines from Git repositories., therefore it can act as a common team or organization source control location for DevOps engineers to maintain tasks on behalf of pipeline creators who can in-turn reference them from that repository. 

In the [git-resolver](https://github.com/skoussou/tekton-gitops-workshop/blob/git-resolver) branch the setup has been updated with the 2 aforementioned `ClusterTask` resources `git-clone` and `s2i-java`
* implemented as `Task` resources, 
* now available from a purposed setup [tekton-catalog](https://github.com/skoussou/tekton-catalog) repository with a versioned entry for [0.1/git-clone.yaml](https://github.com/skoussou/tekton-catalog/blob/main/ci-tasks/resources/git-clone-custom/0.1/git-clone.yaml) and [0.1/s2i-java.yaml](https://github.com/skoussou/tekton-catalog/blob/main/ci-tasks/resources/s2i-java-custom/0.1/s2i-java.yaml)

The [ci-pipeline](https://github.com/skoussou/tekton-gitops-workshop/blob/git-resolver/application-cicd/resources/pipeline-ci.yaml) has been updated to use now the `resolver: git` to reference the task from [tekton-catalog](https://github.com/skoussou/tekton-catalog/tree/main) repository.
```YAML
    # ------------ CLONE APP SOURCE ------------ #
    - name: git-app-clone
      taskRef:
        resolver: git
        params:
          - name: url
            value: 'https://github.com/tektoncd/catalog.git'
          - name: revision
            value: main
          - name: pathInRepo
            value: task/git-clone/0.9/git-clone.yaml
   ...
    # ------------ BUILD IMAGE ------------ #
    - name: build-image
      runAfter:
        - nexus-upload
      taskRef:
        resolver: git
        params:
          - name: url
            value: https://github.com/skoussou/tekton-catalog.git
          - name: revision
            value: main
          - name: pathInRepo
            value: ci-tasks/resources/s2i-java-custom/0.1/s2i-java.yaml
```
</div>


### Migrating to `Bundle` Resolvers

<div style="text-align: justify"> 

A [bundle resolver](https://tekton.dev/docs/pipelines/bundle-resolver/) is a feature in Tekton Pipelines that allows you to reference Tekton resources from a Tekton bundle image, therefore it can act as a common set of prepared images by devops engineers to maintain tasks on behalf of pipeline creators who can in-turn reference them from the bundles. 

In the [bundle-resolver](https://github.com/skoussou/tekton-gitops-workshop/tree/bundle-resolver) branch the setup has been updated with the aforementioned `ClusterTask` resource `s2i-java`
* implemented as a `Task` resource, 
* now available from a purposed prepared image bundle stored in [quay.io/skoussou/s2i-java](https://quay.io/repository/skoussou/s2i-java) repository.

Firstly, the bundle was built from the [0.1/s2i-java.yaml](https://github.com/skoussou/tekton-catalog/blob/main/ci-tasks/resources/s2i-java-custom/0.1/s2i-java.yaml) and versioned to 0.1
```BASH
tkn bundle push quay.io/skoussou/s2i-java:0.1 -f ci-tasks/resources/s2i-java-custom/0.1/s2i-java.yaml

*Warning*: This is an experimental command, it's usage and behavior can change in the next release(s)
Creating Tekton Bundle:
	- Added Task: s2i-java-custom to image

Pushed Tekton Bundle to quay.io/skoussou/s2i-java@sha256:88fb756feeab8e79cf43ba4845cb4b5f2bc27c38f528bba95708c23b34a75908
```

The [ci-pipeline](https://raw.githubusercontent.com/skoussou/tekton-gitops-workshop/bundle-resolver/application-cicd/resources/pipeline-ci.yaml) has been updated to use now the `resolver: bundles` to reference the image bundle from [quay.io](https://quay.io/repository/skoussou/s2i-java) repository.
```YAML
# ------------ BUILD IMAGE ------------ #
  - name: build-image
    runAfter:
      - nexus-upload
    taskRef:
      resolver: bundles
      params:
        - name: bundle
          value: quay.io/skoussou/s2i-java:0.1
        - name: name
          value: s2i-java-custom
        - name: kind
          value: task
```
</div>

## Migrating to Pipelines As Code

<div style="text-align: justify"> 

With the usage of `resolvers` we have moved away from managing pipeline tasks in a distributed and uncontrolled manner. However, up until this point the remainder of the pipeline resources (including `webhooks`, `eventlisteners`, `triggers`, `templates` and `pipelines`), required in order to automatically trigger the  `ci-pipeline` in OpenShift,  are still managed separately from the `application-source` repository, not to mention that they are required to be implemented and maintained adding an extra overhead.

Pipelines-as-code (Generally Available on [OpenShift Pipelines](https://cloud.redhat.com/blog/openshift-pipelines-1.9-released)) which allows to ship the CI/CD pipelines within the same git repository as the application, making it easier to keep both of them in sync in terms of release updates, is the feature we are now going to utilise to migrate the `ci-pipeline` to. 

An important note here is that `pipeline-as-code` functionality relies on integration with supported Git Repository Providers and as the current setup utilises `gitea`, which is not a supported [Pipeline as Code GIT Repository hosting provider](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/using-pipelines-as-code.html#using-pipelines-as-code-with-a-git-repository-hosting-service-provider), the application source code is placed in a separate github [application-source](https://github.com/skoussou/application-source) repository. 

### Creation of a `PipelineRun` Resource

In order to achieve the migration
* the `pipeline-as-code` branch no longer contains the resources [trigger-quarkus-app-push.yaml](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/trigger-quarkus-app-push.yaml) and [pipeline-ci.yaml](https://github.com/skoussou/tekton-gitops-workshop/blob/main/application-cicd/resources/pipeline-ci.yaml), and
* instead they are replaced with a `PipelineRun` resource under the [.tekton](https://github.com/skoussou/application-source/blob/main/.tekton/pipelinerun.yaml) directory.

The `PipelineRun` contains the following important configurations in order to materialise the pipeline and all the necessary functionality including tasks, triggers etc.
1. The definitions when the pipeline will be initiated inside OpenShift include:
  * on a `push` or `pull request` [[1]](https://github.com/skoussou/tekton-gitops-workshop/blob/7a169fab5ead38ec4d6a8da4f40a005d59ecdea3/application-source/.tekton/pipelinerun.yaml#L9)
  * on the `main` branch (of the source code repository) [[2]](https://github.com/skoussou/tekton-gitops-workshop/blob/7a169fab5ead38ec4d6a8da4f40a005d59ecdea3/application-source/.tekton/pipelinerun.yaml#L12)
  * when the `pom.xml` has also been updated [[3]](https://github.com/skoussou/tekton-gitops-workshop/blob/7a169fab5ead38ec4d6a8da4f40a005d59ecdea3/application-source/.tekton/pipelinerun.yaml#L37-L38)
  ```YAML
    apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      name: quarkus-app
      annotations:
        # The event we are targeting as seen from the webhook payload
        # this can be an array too, i.e: [pull_request, push]
        pipelinesascode.tekton.dev/on-event: "[pull_request, push]"

        # The branch or tag we are targeting (ie: main, refs/tags/*)
        pipelinesascode.tekton.dev/on-target-branch: "[main]"

        # Executes only for specific paths
        pipelinesascode.tekton.dev/on-cel-expression: |
          event == "push" && "pom.xml".pathChanged()   
  ```
2. References [[4]](https://github.com/skoussou/application-source/blob/main/.tekton/pipelinerun.yaml#L17-L31) to remote tasks via the [Pipeline as Code resolver](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/using-pipelines-as-code.html#using-pipelines-as-code-resolver_using-pipelines-as-code) in order to retrieve the tasks either from git repository or the hub. 
```YAML
  # git-clone custom task
  pipelinesascode.tekton.dev/task: "https://github.com/skoussou/tekton-catalog/blob/main/ci-tasks/resources/git-clone-custom/0.1/git-clone.yaml"

  # Use maven task from the hub to test our Java project
  pipelinesascode.tekton.dev/task-1: "maven"
   
  # additional custom tasks
  pipelinesascode.tekton.dev/task-2: "https://github.com/skoussou/tekton-catalog/blob/main/ci-tasks/resources/task-generate-version/0.1/task-generate-version.yaml"
  pipelinesascode.tekton.dev/task-3: "https://github.com/skoussou/tekton-catalog/blob/main/ci-tasks/resources/mock/0.1/task-mock.yaml"    
  pipelinesascode.tekton.dev/task-4: "https://github.com/skoussou/tekton-catalog/blob/main/ci-tasks/resources/s2i-java-custom/0.1/s2i-java.yaml"    
  pipelinesascode.tekton.dev/task-5: "https://github.com/skoussou/tekton-catalog/blob/main/ci-tasks/resources/task-tag-image/0.1/task-tag-image.yaml"    
  pipelinesascode.tekton.dev/task-6: "https://github.com/skoussou/tekton-catalog/blob/main/ci-tasks/resources/task-create-pr-to-deploy/0.1/task-create-pr-to-deploy.yaml"       
```

3. The `parameters` which determine: 
  * dynamic [expandable](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/using-pipelines-as-code.html#creating-pipeline-run-using-pipelines-as-code_using-pipelines-as-code) parameters which can be pulled from the commit 
  * or hardcoded [[5]](https://github.com/skoussou/tekton-gitops-workshop/blob/7a169fab5ead38ec4d6a8da4f40a005d59ecdea3/application-source/.tekton/pipelinerun.yaml#L41-L70) ones.

4. The `pipelineSpec` which determines the pipeline either as:
  * a [reference](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/using-pipelines-as-code.html#using-remote-pipeline-annotations-with-pipelines-as-code_using-pipelines-as-code) via the annotation ` pipelinesascode.tekton.dev/pipeline: "<https://git.provider/raw/pipeline.yaml>"`
  * or (as delivered here) a list of tasks [[6]](https://github.com/skoussou/tekton-gitops-workshop/blob/7a169fab5ead38ec4d6a8da4f40a005d59ecdea3/application-source/.tekton/pipelinerun.yaml#L71-L280)

### Pipelines as Code integration with a `GitHub App`

GitHub Apps act as a point of integration with Red Hat OpenShift Pipelines and in order to achieve this a single GitHub App for all cluster users is configured. This can be performed both via `tkn app` CLI or [manually](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/using-pipelines-as-code.html#configuring-github-app-for-pac), here the former has been selected.

In order to configure Pipelines as Code to access the newly created GitHub App the following is executed whilst logged into OpenShift and within the `openshift-pipelines` namespace. As a result a `Secret` [resource](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/using-pipelines-as-code.html#configuring-pac-for-github-app) `pipelines-as-code-secret` is created in `openshift-pipelines` enabling the integration. 

> Note: the Github App name must be unique.

```BASH
$ oc project openshift-pipelines
$ tkn pac bootstrap
  => Checking if Pipelines as Code is installed.
  ✓ Pipelines as Code is already installed.
  ? Enter the name of your GitHub application:  test-ppln-as-code
  👀 I have detected an OpenShift Route on: https://pipelines-as-code-controller-openshift-pipelines.apps.<CLUSTER_NAME>.<DOMAIN_NAME>
  ? Do you want me to use it? Yes
  🌍 Starting a web browser on http://localhost:8080, click on the button to create your GitHub APP
  🔑 Secret pipelines-as-code-secret has been created in the openshift-pipelines namespace
  🚀 You can now add your newly created application on your repository by going to this URL:
          
  https://github.com/apps/test-ppln-as-code
          
  💡 Don't forget to run the "tkn pac create repo" to create a new Repository CRD on your cluster.
```

The new `Github App` is associated with the source repository. 

![Github App](/images/20230720/pipeline-as-code-github-app.png)


Finally the following command [adds a webhook and a Repository CR](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/using-pipelines-as-code.html#using-pipelines-as-code-with-github-webhook_using-pipelines-as-code) automatically for the pipeline as code to process events from [application-source](https://github.com/skoussou/application-source) repository and run the pipeline in `sk-workshop-components` namespace based on the [Repository CR](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/using-pipelines-as-code.html#using-repository-crd-with-pipelines-as-code_using-pipelines-as-code).

```BASH
$ tkn pac create repository
  ? Enter the Git repository url (default: https://github.com/skoussou/tekton-gitops-workshop):  https://github.com/skoussou/application-source
  ? Please enter the namespace where the pipeline should run (default: openshift-pipelines): sk-workshop-components
  ✓ Repository skoussou-application-source has been created in sk-workshop-components namespace
  ℹ Directory .tekton has been created.
  ✓ A basic template has been created in .tekton/pipelinerun.yaml, feel free to customize it.
  ℹ You can test your pipeline by pushing generated template to your git repository
```

Now the pipeline as code reacts to any `push` event and initiates a new `PipelineRun` (for more pac information read [here](https://cloud.redhat.com/blog/create-developer-joy-with-new-pipelines-as-code-feature-on-openshift)).

![Pipeline As Code Flow](/images/20230720/pac.png)


The results of the `PipelineRun` are not only reported in the OpenShift console but additionally shared back on the github repo. 

![Pipeline As Code Results](/images/20230720/pac-results.png)



## Conclusion

The efforts have showcased how as a team we can better organize both pipeline components such as tasks and pipelines in order to promote re-usability, versioning and maintenance. There are still some questions open on the use of pipeline as code in multi component applications however the ease of setting up the flow in comparison to the complexity of maintaining all tekton CRs to achieve the same effect is undoubtedly impressive.

</div>

