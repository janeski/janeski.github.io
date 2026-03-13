---
title: "Light and Fast: Building Cloud-Native Microservices with Azure Functions and Dapr Extension"
date: 2024-07-29 00:00:00
description: "How the Dapr Extension for Azure Functions simplifies microservices development by eliminating boilerplate for Service Bus and CosmosDB integrations."
categories: [cloud, microservices]
tags: [azure-functions, dapr, microservices, cloud-native, azure, azure-container-apps, service-bus, cosmosdb, dotnet, csharp, distributed-systems, serverless, devops]
canonical: https://www.linkedin.com/pulse/light-fast-cloud-native-microservices-azure-functions-janeski-8upwf/
---

In today's microservices-driven world, managing communication between services can become complex and cumbersome. Azure Functions, Microsoft's serverless computing service, offers a powerful way to run event-driven code without worrying about the underlying infrastructure. The serverless concept is not new, and I have already shared a couple of posts regarding its benefits. The inspiration for this post comes from a recent feature released by the Azure Functions team: the **Dapr Extension for Azure Functions**.

For those unfamiliar with Dapr, I recommend watching my conference talks, in which I discuss its capabilities and benefits in detail, especially from a developer perspective.

When combined with Dapr (Distributed Application Runtime), Azure Functions can leverage a set of building blocks that simplify distributed application development. This post will explore using Dapr bindings with Azure Functions to streamline microservices development. The **Dapr Extension for Azure Functions** combines the best practices for developing and operating microservices.

## Microservices with Dapr

In my Dapr example, I developed three microservices:

1. **Ingestion Service** — responsible for receiving, storing, and forwarding requests to the next microservice
2. **Transformation Service** — responsible for reading requests from a queue, applying business logic transformation, and forwarding requests to the next microservice
3. **Extraction Service** — responsible for extracting the final knowledge from requests. It receives requests from a queue and extracts knowledge using AI services.

The microservices are Docker images running in containers on top of Azure Container Apps.

The microservices adopt the Dapr sidecar pattern, and whenever they need to talk to a queue or state management service (CosmosDB, in my case), they call services from the Dapr sidecar container.

## Microservices with Dapr and the Dapr Extension for Azure Functions

To benefit from the serverless technology, I restructured my example with the **Dapr Extension for Azure Functions**. I no longer need three standalone applications (1 API for the Ingestion Service and 2 Console Apps for queue processors). This is replaced with one Azure Functions Application with three functions — one for each microservice. In general, it means I reduced some of the boilerplate code of the solution. The final solution is only responsible for business logic.

In this post, I illustrate the restructuring of the Ingestion Service.

## Transform the Ingestion Service

The Ingestion Service is no longer an API project — only one Azure Function with:

- **HTTP Trigger** invocation
- **Dapr Publish** output → Service Bus Queue
- **Dapr State** output → CosmosDB

One challenge I found was using multiple outputs, but there is an example in the Dapr documentation. The final result is a single-file solution that maintains the same functionality.

## Conclusion

The example above shows how an API project can be transformed into a one-file solution while maintaining the same functionality and gaining all the benefits of Azure Functions. The Dapr benefits simplify the development process by reducing boilerplate source code, while the Dapr Extension for Azure Functions takes this reduction to the next level. It is the best example of why we should consider developing more cloud-native applications.
