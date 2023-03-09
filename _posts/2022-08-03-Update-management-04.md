---
layout: post
title: Azure Update Management - Part 5 - Automation accounts
date: 2023-02-02 10:00:00
description: This post is describes the architecture of the solution
tags: updatemanagement
authors:
  - name: Molx32

bibliography: 2022-08-01-Update-management.bib

toc:
  - name: Equations
    # if a section has subsections, you can add them as follows:
    # subsections:
    #   - name: Example Child Subsection 1
    #   - name: Example Child Subsection 2
  - name: Citations
  - name: Footnotes
  - name: Code Blocks
  - name: Layouts
  - name: Other Typography?
---
<i>Hello there, the goal of this serie is to describe a real world implementation of <b>Azure Update Management</b>.
This service was designed to update any machine in your infrastructure, whether they are hosted on Azure or elsewhere,
provided your OSs are technically supported.</i>

<i>In order for every reader to understand and find answers to their need, to I'll try to give a comprehensive feedback from my experience, as well as sharing tips about design and architecture, automation, effectivement, troubleshooting issues, and so on!</i>

<i>Have a nice reading.</i>

***

#### Plan
- [Part 0 - Introduction](/blog/2022/Update-management-00/)
- [Part 1 - Architecture](/blog/2022/Update-management-01/)
- [Part 2 - Azure Policy](/blog/2022/Update-management-011/)
- [Part 3 - Azure ARC](/blog/2022/Update-management-02/)
- <b>[Part 4 - Log Analytics agents](/blog/2022/Update-management-03/)</b>
- [Part 5 - Automation accounts (you're here)](/blog/2023/Update-management-04/)
- [Part 6 - Monitoring](/blog/2023/Update-management-05/)
- [Part 7 - Security patches on Azure ARC](/blog/2023/Update-management-06/)
- [Part 8 - Security patches on CentOS machines](/blog/2023/Update-management-07/)

***

## Context
In the previous posts, we saw :
1. How to programmatically install the Log Analytics agent using Azure Policy
2. How to deploy Azure ARC on non-Azure VMs

At this point, the Log Analytics agent should be installed on most of your VMs. However, it is important to keep in mind how the Log Analytics agent works, what are the alternative deployment methods, what the limits are, and so on. This is what this post deals with. 

## Log Analytics workspaces
The Log Analytics service is one of the most used services in Azure : almost all monitoring services rely on it. How does this service work?
1. It collects and stores data, just like a database. To be more accurate, clients send data to the service, and the service store these data for a defined period of time.
2. Client can send queries to the Log Analytics service that will send back requested data.

This is as simple as that, and it is very powerfull. A lot of Azure services rely on Log Analytics : Microsoft Sentinel, Microsoft Defender for Cloud, Application Insights, and any monitoring solution that relies on logs, such as... Azure Automation Update, of course!
Billing

## Log Analytics agent
Behavior, etc.

## Deployment
### Using extensions
### Using script
### Using policies
### 

## 