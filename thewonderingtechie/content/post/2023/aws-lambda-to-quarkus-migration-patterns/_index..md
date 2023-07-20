+++
author = "Stelios Kousouris"
title = "AWS Lambda to Quarkus Migration Patterns"
date = "2023-07-12"
thumbnail = "/images/20230712/migrating-lambda-handlerRequest-to-quarkus.png"
shareImage = "/images/20230712/migrating-lambda-handlerRequest-to-quarkus.png"
featureImage = "/images/20230712/migrating-lambda-handlerRequest-to-quarkus.png"
description = "The article provides guidance on migrating AWS Java Lambda code into a Quarkus based project dependent on the AWS service pattern."
tags = [
    "quarkus",
    "containers",
    "microservices",
    "OpenShift",
    "cloudnative"
]
categories = [
    "CloudNative"
]
toc = true
years = "2023"
+++

## Introduction

<div style="text-align: justify"> 

Recently, we were tasked with the migration of a set of AWS Java Lambda applications to containers on Red Hat's Kubernetes Platform of [OpenShift on the Azure cloud](https://www.redhat.com/en/technologies/cloud-computing/openshift/azure). [AWS Lambdas](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) are compute services that let you run code without provisioning or managing servers and they can do so on-demand, on the other hand containers offer richer functionality with the ability to run long-running, on-demand, in-memory multithreaded services whilst they demand your attention to the configuration of the underlying container framework.

Red Hat's [Quarkus Java runtime](https://quarkus.io/container-first/) was selected to form the basis for the migration of the lambdas to containers as it is a Java framework which would ease the migration of the business logic of the Java lambda and in addition it is designed exclusively for cloud native development, with reduced [memory consumption and startup time](https://quarkus.io/guides/performance-measure), support for container functionality (e.g. [health checks](https://quarkus.io/guides/smallrye-health), [metrics](https://redhat-developer-demos.github.io/quarkus-tutorial/quarkus-tutorial/metrics.html), [configuration](https://quarkus.io/guides/config-reference), [secrets](https://quarkus.io/guides/kubernetes-config) & [serverless functions](https://developers.redhat.com/articles/2022/03/28/build-your-first-java-serverless-function-using-quarkus-quick-start)) and finally it delivers enhanced developer experience with dynamic reload and local developer services using test containers.

In this article we walk through the four types of AWS lambda service patterns involved in the use case and how one can migrate the AWS lambda `handlerRequest()` code into a Quarkus method and project. 

The  four types of AWS lambda service patterns identified in the use case included:
* long-running REST API triggered lambdas,
* long-running scheduled lambdas,
* event message based triggered lambdas and
* one-off job based lambdas.

Furthermore, in a follow-up article we will look at the set of cookbook activities to migrate and configure external integrations, configure the Quarkus project for them and set up container building and testing.
</div>

## AWS Lambda to Quarkus application code migration activities

<div style="text-align: justify"> 

In order to ensure all lambda code migrations to Quarkus containers follow a standard set of activities they have been documented and are available via an [interactive diagram](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers#interactive-detailed-migration-flow) of the migration path. The following diagram is a static view of the activities path defined for a single AWS Lambda migration and it contains the two main paths the migration involved:
* prepare for deployment 
* migrate the business code (the focus of this article)
</div>

![AWS Lambda to Quarkus Application Code Migration Activities](/images/20230712/Lamda-to-Quarkus-Code-Migration-Activities.png)


### Quarkus quickstart application project selection

<div style="text-align: justify"> 

The initial action in the migration path involves selecting the appropriate quickstart project and generating a new project based on this skeleton. The cookbook repository [aws-lambdas-to-azure-quarkus-containers](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers#interactive-detailed-migration-flow) contains one quickstart application project for each of the four AWS Lambda service patterns and the selection is based on the Lambda charactetistics. The quickstart projects available are:
</div>

{{<table "table table-striped table-bordered">}}
|Quickstart     | App Characteristics     |
|---------------|-------------------------|
| [Rest API (`Deployment` Based OCP app) Quarkus Quickstart](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers/tree/main/quickstart-lambda-to-quarkus-rest)   | A REST API application resulting in a CosmosDB Connection and entry creation   |
| [Job Main (`CronJob` Based OCP app)  Quarkus Quickstart](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers/tree/main/quickstart-lambda-to-quarkus-cronjob)   | A Job based application, it demonstrates setting up the job and a CosmosDB connection  |
| [Scheduled Job (`Deployment` Based OCP app) Quarkus Quickstart](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers/tree/main/quickstart-lambda-to-quarkus-scheduled)   | A scheduled based application runs at pre-defined intervals. Demonstrates, connecting and authenticating with Kafka, publishing and subscribing to messages   |
| [Message Initiated microservice (`Deployment` Based OCP app) Quarkus Quickstart](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers/tree/main/quickstart-lambda-to-quarkus-kafka)  | An event or message driven application, demonstrates connecting and authenticating with Kafka, publishing and subscribing to messages  |
{{</table>}}



### Lambda `handlerRequest()` code migration to Quarkus method

<div style="text-align: justify"> 

The `handerRequest` is the Java method which contains the lambda business code and is invoked in each of the lambda implemented service patterns.  Here we will look at how this code can be migrated to a Quarkus based method that delivers the same pattern.
</div>

#### Code Migrations of a Job based Lambda

<div style="text-align: justify"> 

For a lambda which starts as the result of a single run job the code migration begins by calling or placing the `handlerRequest()` lambda code in a method simply annotated with `@ApplicationScoped`. The Quarkus application does not need to configure anything additional as long as the Kubernetes deployment for the POD created for this application is of `CronJob` type.
</div>

{{< highlight go "linenos=inline,hl_lines=12-13,linenostart=0" >}}
```
@ApplicationScoped
public class JobMain {
  @Override
  public int run(String... args) {
    Log.info("Running JobMain .... ");
      this.httpClient = HttpClient.newBuilder()
          .connectTimeout(Duration.ofSeconds(10))
          .version(Version.HTTP_1_1)
          .build();
      this.mapper = new ObjectMapper();
      //Lambda business method below
      this.handleRequest();     
      return 0;
  }
```  
{{< / highlight >}} 

An example for this service exists in [JobMain.java#run](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers/tree/main/quickstart-lambda-to-quarkus-cronjob#code-migrations).


#### Code Migrations of a REST API Triggered based Lambda

<div style="text-align: justify"> 

For a lambda which starts as the result of a REST API call the code migration begins by calling or placing the `handlerRequest()` lambda code in a method annotated with `@Path("/hello")`. Furthermore, depending on the URI  path the code should be invoked at the annotation can be appropriately adjusted.
</div>

{{< highlight go "linenos=inline,hl_lines=3 17-18 27-28,linenostart=0" >}}
```
  @ApplicationScoped
  @Path("/hello")
  public class HelloCosmosResource {
  
      @Inject
      MeterRegistry registry;
         
      @Inject
      protected CosmosConnection connection;
         
      @GET
      @Produces(MediaType.TEXT_PLAIN)
      @Path("{country}")
      public String hello(String country) {
  
          // Code call to Lambda handler method
          this.handleRequest(); 
      }
  
      @POST
      @Produces(MediaType.TEXT_PLAIN)
      @Consumes(MediaType.APPLICATION_JSON)
      @Path("/create")
      public String helloPost(HelloCountry hc) {
  
          // Code call to Lambda handler method
          this.handleRequest(); 
      }
  }
```
{{< / highlight >}}   

<div style="text-align: justify"> 

In [HelloCosmosResource.java#hello](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers/blob/main/quickstart-lambda-to-quarkus-rest/src/main/java/com/redhat/cloudnative/hellocosmos/HelloCosmosResource.java) you can find the above example whilst the Quarkus project requires the following extensions in order to offer the REST functionality.
</div>

```XML
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-resteasy-reactive</artifactId>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-resteasy-reactive-jackson</artifactId>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-smallrye-openapi</artifactId>
</dependency>
```


#### Code Migrations of a Scheduled Triggered based Lambda

<div style="text-align: justify"> 

For a lambda which starts as the result of a scheduled timer based event the code migration begins by calling or placing the `handlerRequest()` lambda code in a method with `@Scheduled()` annotation adjusting accordingly the intervals the scheduling will occur. The following example exists in [PollOrchestrator.java#dmdScheduler](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers/blob/main/quickstart-lambda-to-quarkus-scheduled/src/main/java/com/redhat/cloudnative/simmanagement/datahandler/pollers/function/PollOrchestrator.java).
</div>

{{< highlight go "linenos=inline,hl_lines=15 18-19,linenostart=0" >}}
```
@ApplicationScoped
public class PollOrchestrator {

  // The Lambda handle method containing class
  @Inject
  LambdaFunction dmdPoller;

  @Inject
  LastFetchedEventRepository lastFetchedEventRepository;

  public PollOrchestrator() {
  }

  @Scheduled(every = "10s")
  public void dmdScheduler() {
    try {

      // Call the Lambda handlerRequest method
      dmdPoller.handleEvent(UUID.randomUUID().toString());

      registry.counter("country_counter", Tags.of("name", "dmdScheduled")).increment();

    }  catch (Exception e) {
      logger.error("Unknown exception for DMD Scheduler: {}", e.getMessage(), e);
    }
  }
}
```
{{< / highlight >}}

In order for Quarkus to provide the scheduling functionality the `quarkus-scheduler` extension is added to the project.

```XML
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-scheduler</artifactId>
</dependency>
```


#### Code Migrations of an Event or Message Triggered based Lambda

<div style="text-align: justify"> 

For a lambda which starts as the result of an event the code migration begin by calling or placing the `handlerRequest()` lambda code in a method annotated with `@Incoming("quickstart-kafka-in")` which determines the Kafka Topic Quarkus will be integrating and listening to for new events. 
</div>

{{< highlight go "linenos=inline,hl_lines=2 13-14,linenostart=0" >}}
```
  @Incoming("quickstart-kafka-in")
  public CompletionStage<Void> consumeQuickstartKafkaIn(Message<Event> msg) {

      try{
          Log.info("consumeQuickstartKafkaIn : Event received from Kafka");

          registry.counter("events", Tags.of("event_type", msg.getPayload().getType())).increment();

          Event event = msg.getPayload();
          Log.info("Payload : "+event);

          // Code call to Lambda handlerRequest method
          this.handleRequest(); 
      }
      catch (Exception e) {
          Log.error(e.getMessage());    
      }
      finally {
          return msg.ack();
      }
    }
```
{{< / highlight >}} 

<div style="text-align: justify"> 

An example implementation exists in [EventResource.java#consumeQuickstartKafkaIn](https://github.com/skoussou/aws-lambdas-to-azure-quarkus-containers/blob/main/quickstart-lambda-to-quarkus-kafka/src/main/java/com/redhat/cloudnative/kafka/EventResource.java#L79-L97) method whilst the `quarkus-smallrye-reactive-messaging-kafka` extension is added to the Quarkus project to provide the integration functionality to Kafka.
</div>

```XML
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-smallrye-reactive-messaging-kafka</artifactId>
</dependency>
```

## Summary

<div style="text-align: justify"> 

In this article we have seen how one can start migrating Java lambda application code into container based Quarkus projects. In the [A cookbook on migrating AWS Lambda applications to Quarkus]() we will delve more on the activities to migrate Lambda external integrations configuring the Quarkus project for them whilst also setting them up for building and testing of the containerized application.

</div>


