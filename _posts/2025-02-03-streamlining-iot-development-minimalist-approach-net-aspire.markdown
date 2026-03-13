---
title: "Streamlining IoT Development: A Minimalist Approach with .NET Aspire, MQTT, and TimescaleDB"
date: 2025-02-03 00:00:00
description: "Build a starter IoT app with .NET Aspire, MQTT, TimescaleDB, and Azure SignalR — the full distributed system orchestrated in ~40 lines of C#."
categories: [cloud, dotnet]
tags: [dotnet, aspire, iot, mqtt, azure, signalr, timescaledb, cloud-native, microservices, csharp, containers, devops]
canonical: https://www.linkedin.com/pulse/streamlining-iot-development-minimalist-approach-net-aspire-janeski-t2rdf/
---

Local development is a serious challenge when developing distributed systems, especially IoT solutions. The development team often requires end-to-end local development to complete the job. For example, imagine you have to tweak the real-time update on your charts in the web app — you either need an IoT device simulator or plug into the development environment. In both cases, you need the process automated. Otherwise, it will become cumbersome and too expensive.

This article presents a simple approach to building a starter IoT application with .NET Aspire. Before I discuss the technical details, I will elaborate on the building components. Since this is a starter IoT application, I select only the minimal required components to establish an end-to-end skeleton app that facilitates further development in all directions: IoT device development, cloud, and web development.

## Architecture

The first component I need is an **MQTT broker**. The MQTT broker is the linking component between the IoT devices and the cloud application. For production, you can choose your preferred MQTT implementation. For local development, I use **Eclipse Mosquitto**.

The next component is a **time series database** where I store the IoT device telemetry data. My preference is **TimescaleDB**. One of the reasons I prefer TimescaleDB is that it is an extension of PostgreSQL — technically, I get both the relational database and the time-series database in a single instance.

The next step is a component linking the MQTT broker and the time series DB. This component very often holds the business logic of your solution. At the same time, an API will probably be responsible for exploring the historical data of the time series and adjusting the business logic. For simplicity, in the starter IoT application, the MQTT "handler" and the API co-exist in the same **.NET service**. The MQTT "handler" is implemented as a background worker job, making it easy to refactor into an additional service.

Additionally, the starter solution has a **web application** and **SignalR service** (running in the cloud: Azure SignalR).

The final architecture for the IoT starter application includes:

1. **MQTT Server** (Eclipse Mosquitto) — running locally as a Docker container
2. **TimescaleDB Server** — running locally as a Docker container
3. **Custom .NET Service** — manages MQTT traffic and provides a minimal API, running locally as a Docker container
4. **Azure SignalR Service** — hosted in Azure, referenced by the solution
5. **Angular Web Application** — running locally as a Docker container

## .NET Aspire

Following the .NET Aspire documentation, the above services are defined and orchestrated in the App Host/Orchestrator project. The main benefit of .NET Aspire here is that the distributed system's orchestration, dependency, and interconnectivity are done automatically, with **~40 lines of C# code**.

Additionally, .NET Aspire does not add additional source code or complexity behind it. It solves the challenges automatically by following common practices. As a bonus, when the AppHost application runs, all the services are up and running and monitored from the **Aspire Dashboard** — one of the standout Aspire features.

## Takeaways

1. **Aspire accelerates PoC delivery** — It enables teams to move quickly from idea to working prototype.
2. **Aspire automates best practices** — It doesn't introduce extra libraries; it streamlines proven approaches for specific use cases.
3. **Aspire bridges the gap between architecture and development** — Bringing architecture closer to developers by defining it in source code.
4. **Aspire makes architecture part of the code** — There is no separate "architecture as code"; with Aspire, architecture *is* the code, written in C#, the language developers speak.

View full code on [GitHub](https://github.com/janeski/iot-cloud-native).
