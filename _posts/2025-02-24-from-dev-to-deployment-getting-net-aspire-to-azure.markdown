---
title: "From Dev to Deployment: Getting .NET Aspire to Azure with Azure Developer CLI (azd)"
date: 2025-02-24 00:00:00
description: "Deploy a .NET Aspire app to Azure Container Apps with a single azd up command — covering azd init, infra synth, and the full deployment pipeline."
categories: [cloud, dotnet]
tags: [dotnet, aspire, azure, azure-container-apps, azd, cloud-native, devops, containers, csharp, bicep, deployment, microservices]
canonical: https://www.linkedin.com/pulse/from-dev-deployment-getting-net-aspire-azure-miroslav-janeski-ymrde
image: /images/posts/1740080596540.jpg
image_credit: 'Photo by <a href="https://www.pexels.com/photo/person-holding-space-rocket-toy-3697818/" target="_blank" rel="noopener">Matheus Bertelli</a> on Pexels'
---

.NET Aspire simplifies local development and brings the architecture closer to the development team. These are the main takeaways from my [last article]({% post_url 2025-02-03-streamlining-iot-development-minimalist-approach-net-aspire %}). However, we cannot talk about cloud-native applications if we only focus on local development. The main idea behind cloud-native applications is the new perspective on efficient development and moving to production. The main task of the tools in the cloud-native landscape is to make the whole process much easier, simple, and generally efficient.

## Azure Developer CLI (azd)

The **Azure Developer CLI (azd)** is an open-source tool that accelerates the migration from a local development environment to Azure. `azd` provides a set of developer-friendly commands that map to the main steps in the DevOps process (code, build, deploy, monitor). The team behind `azd` has built great new features to make .NET Aspire development and deployment to Azure a friction-free experience.

.NET Aspire projects are designed to run in containerized environments, and `azd` has been extended to smooth the release of .NET Aspire projects to Azure Container Apps.

## azd in Action

After installing `azd` (`winget install microsoft.azd`), the next step is to navigate to your .NET Aspire solution folder and execute `azd init`. This initializes a new application, based either on the Aspire solution or a template.

The `azd init` command scans the Aspire solution and creates two files:

- `azure.yaml`
- `next-steps.md`

![azd init](/images/posts/azd_init.png)

The `azure.yaml` file contains a service referencing your project's App Host. The yaml files for the rest of the services in the Aspire application are generated and stored in memory only when needed.

To get an overview of the infrastructure, the command `azd infra synth` generates all the required yaml files. This command is not enabled by default since it's in the alpha stage — enable it with: `azd config set alpha.infraSynth on`

After running the command, azd generates yaml files for all the applications in the Aspire solution.

![azd infra synth](/images/posts/azd_infra_synth.png)

## azd up: One Command to Deploy Everything

Once the azd project has been initialized, the next step is to take the project to Azure. `azd up` is a single command that runs the whole magic. Here's a detailed overview of what it does:

![azd up](/images/posts/azd_up.png)

1. When azd targets a .NET Aspire project, it starts the AppHost with a special command that produces the Aspire manifest file.
2. The sub-command logic interrogates the manifest file to generate Bicep files in memory (by default).
3. After generating the Bicep files, a deployment is triggered using Azure's ARM APIs targeting the subscription and resource group provided.
4. Once the underlying Azure resources are configured, the `azd deploy` sub-command logic is executed using the same Aspire manifest file.
5. As part of the deployment, azd calls `dotnet publish` using .NET's built-in container publishing support to generate container images.
6. Once azd has built the container images, it pushes them to the ACR registry created during the provisioning phase.
7. Finally, once the container image is in ACR, azd updates the resource using ARM to start using the new container image.

![azd deploy](/images/posts/azd_deploy.png)

After the deployment, the **Aspire Dashboard application** is also hosted in Azure Container Apps. It shows all the services defined in the Aspire solution.

![Aspire Dashboard in Azure Container Apps](/images/posts/Aspire_Dashboard_Azure_Container_Apps.png)

An interesting detail: checking the Azure Resource Group reveals that all the required services to run the Aspire application have been provisioned and configured automatically. This eliminates the necessity for manual configuration management for different environments — previously one of the biggest headaches in enterprise projects.

![Azure Resources azd script](/images/posts/Azure_Resources_azd_script.png)

## Takeaways

**Pros:**

1. Aspire applications are very well supported in azd. The experience deploying an Aspire application to Azure Container Apps is smooth and effortless.
2. No more connection strings and environment variables to manage manually. The service interconnection is done automatically and securely.
3. azd makes feature branch releases straightforward and can be easily integrated into a DevOps pipeline.
4. Aspire + azd = the perfect example of cloud-native application development tools.

**Cons:**

1. azd only supports Azure Container Apps natively. It partially supports Azure Kubernetes Service, but you have to run Aspir8 to generate Kubernetes manifests.
2. azd requires at least one custom application in the Aspire solution. If there are only Docker file services, the provisioning cannot be instantiated.
3. azd has certain limitations. For instance, deploying a time-series database that requires an initialization script — azd created the Azure File Share and mounted it to the container app, but the initialization script file was not uploaded automatically. A manual upload was required.

## Read More

- [.NET Aspire](https://learn.microsoft.com/en-us/dotnet/aspire/get-started/aspire-overview?wt.mc_id=MVP_395548)
- [Azure Developer CLI (azd)](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview?wt.mc_id=MVP_395548)
- [Deploy a .NET Aspire solution to Azure](https://learn.microsoft.com/en-us/dotnet/aspire/deployment/azure/aca-deployment?wt.mc_id=MVP_395548)
