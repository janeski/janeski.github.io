---
title: "When to Go Serverless with Azure Functions: A Rule of Thumb for Enterprise Architecture"
date: 2024-06-01 00:00:00
description: "A rule of thumb for adopting serverless with Azure Functions — use metrics to spot ad-hoc spikes, the key signal that serverless is the right architectural choice."
categories: [cloud, azure]
tags: [azure-functions, serverless, cloud-native, azure, microservices, architecture, dotnet, scalability, devops]
canonical: https://www.linkedin.com/pulse/rule-thumb-when-go-serverless-miroslav-janeski-onv4f/
---

If you are wondering when you should go for serverless, especially in the context of an enterprise solution, I have a rule of thumb for you.

Use Azure Functions to avoid spikes in your metrics (CPU, Memory, etc.). Metrics spikes come from use cases that are not part of your system's usual usage. Ad-hoc use cases that greatly impact your system's performance are your worst nightmare. These use cases will make your customers leave and stop using your system.

## Check Your Metrics

If you are working on a brownfield solution, you can check your existing metrics. There are a couple of types of metrics that you can get.

The first type is **flat metric lines**. That means you have a well-architected system with a stable workload. Your architect and your DevOps engineers did a good job. Though, your sales and marketing team were not doing well — flat metric lines mean you are not engaging new customers, but that's not the subject of this post.

If you engage new customers, your metrics show lines that **grow proportionally** (CPU, Memory) in line with new customers. You might get step lines or zig-zag lines if you have auto-scale policies. In both cases, the increased workload is planned and expected. Therefore, you can apply fine-tuned auto-scale policies to ensure a smooth experience for your existing and new customers.

The third type of line in your metrics is **ad-hoc spikes**. Spikes are unpredictable, and handling them with an auto-scale policy is almost impossible. Therefore, they can seriously affect your system's performance and customer experience (delays and timeouts).

## The Rule of Thumb

If you can recognize the use cases that create or will create spikes in your metrics, then those use cases are perfect candidates for serverless technology. Serverless technology enables auto-scale, but not with the resources from your main system (CPU, Memory). In that way, your system stays stable, and your customers are happy.

## Example Use Cases

You can consider a serverless approach for any use cases with unpredictable usage that are not part of the most common usage (for example, user login). Here are a few typical examples:

- **Expose an OData API** for an existing system
- **Big file upload** (raw data for post-processing)
- **Ad-hoc reports and calculations** (custom billing or usage reports)

Of course, if you build a greenfield solution, you cannot consider metrics analysis. In such a scenario, you have to detect the sporadic use cases in collaboration with the business analyst and the rest of the stakeholders.

In my previous blog post, I elaborated on the new consumption model for Azure Functions: [Flex Consumption]({% post_url 2024-05-22-azure-functions-flex-consumption-solves-cold-start %}), which is a complete game changer in the serverless world.

What is your experience with serverless?
