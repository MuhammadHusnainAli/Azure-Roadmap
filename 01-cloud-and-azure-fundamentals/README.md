# 01 · Cloud and Azure Fundamentals

> Tracks: E ● A ● C ● · Part A · Foundations

The vocabulary and mental model everything else in this roadmap builds on. Nothing here is AI-specific, but every AI design decision later (which region, which subscription, whose responsibility, what SLA) is an application of these ideas.

## What you will be able to do

- Explain service models and pricing models to a non-technical stakeholder.
- Say who is responsible for what in any Azure service, including AI services.
- Choose a region and a redundancy level and defend the choice.
- Read and design a management group, subscription and resource group layout.
- Explain what happens when you click **Create** in the portal.
- Calculate a composite SLA and spot a quota problem before it blocks a deployment.
- Pick the right category of service for a given requirement.

## Prerequisites

Basic IT literacy: what a server, a network, a database and an API are. No Azure experience needed.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Cloud Computing Concepts](01-cloud-computing-concepts.md) | 40 min | – |
| 02 | [Shared Responsibility Model](02-shared-responsibility-model.md) | 30 min | – |
| 03 | [Azure Global Infrastructure](03-azure-global-infrastructure.md) | 45 min | CLI |
| 04 | [Azure Resource Hierarchy](04-azure-resource-hierarchy.md) | 45 min | CLI |
| 05 | [Azure Resource Manager](05-azure-resource-manager.md) | 60 min | CLI |
| 06 | [Accounts, Subscriptions and Offers](06-accounts-subscriptions-and-offers.md) | 40 min | Portal |
| 07 | [Portal, Cloud Shell and Mobile App](07-portal-cloud-shell-and-mobile-app.md) | 40 min | Portal |
| 08 | [Service Lifecycle, SLAs and Quotas](08-service-lifecycle-slas-and-quotas.md) | 50 min | CLI |
| 09 | [Azure Service Map](09-azure-service-map.md) | 45 min | – |

Read 01 to 05 in order. 06 to 09 can be read in any order.

## Completion checklist

- [ ] I can classify any service as IaaS, PaaS, SaaS or serverless and say what I still manage.
- [ ] I can draw the shared responsibility split for a VM, an App Service app and a Foundry model deployment.
- [ ] I can explain region, availability zone, region pair and geography, and when each matters.
- [ ] I have created a resource group, tagged it, locked it and deleted it from the CLI.
- [ ] I can explain control plane vs data plane with an example.
- [ ] I know which billing account type my subscription sits under.
- [ ] I can compute the composite SLA of a three-tier application.
- [ ] I have checked a quota and know how to request an increase.
- [ ] I can name the main service in each Azure category without looking it up.

## Maps to

- **AZ-900**: describe cloud concepts; describe Azure architecture and services (core architectural components).
- **AZ-104**: manage subscriptions and resource groups.
- **AZ-305**: design governance; design for high availability (region and zone concepts).

## Next

[02 · Identity and Access with Microsoft Entra](../Roadmap.md#02--identity-and-access-with-microsoft-entra)
