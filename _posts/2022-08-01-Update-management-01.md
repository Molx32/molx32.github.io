---
layout: post
title: Azure Update Management - Part 1 - Architecture
date: 2022-08-02 10:00:00
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
- [Part 1 - Architecture (you're here)](/blog/2022/Update-management-01/)
- [Part 2 - Azure Policy](/blog/2022/Update-management-011/)
- [Part 3 - Azure ARC](/blog/2022/Update-management-02/)
- [Part 4 - Log Analytics agents](/blog/2022/Update-management-03/)
- [Part 5 - Automation accounts](/blog/2023/Update-management-04/)
- [Part 6 - Monitoring](/blog/2023/Update-management-05/)
- [Part 7 - Security patches on Azure ARC](/blog/2023/Update-management-06/)
- [Part 8 - Security patches on CentOS machines](/blog/2023/Update-management-07/)

## Architecture
In this guide to Azure Update Management, we are going to dig into the details of the service. Azure Update Management is composed of multiple Azure services.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/arch_1.png" class="img-fluid rounded z-depth-1" %}
</div>
1. <b>Azure ARC</b> - First of all, non Azure VMs need an additional step : connecting the VM to Azure. This is can be achieved with Azure ARC
2. <b>Reporting with Log Analytics agents</b> - Now that we can manage both Azure and non-Azure VM directly from Azure, we can set up an Azure Policy. Basically, an Azure policy can enforce a pre-defined configuration on our VM. In that case, we enforce the Log Analytics agent to be installed on each VM (both Azure and non-Azure) and report logs to a Log Analytics workspace. Be doing this, any newly created VM will be onboarded on Update Management.
3. <b>Monitoring with workbooks</b> - In order to monitor service health, update status, and some other information, we can create an Azure Workbook, which is a dashboard. The workbook is based on the Log Analytics workspace that will collect all the update information from all VMs.
4. <b>Updating with Automation Accounts.</b> - The core service of Update Management in Azure is <b>Automation Account</b>. This is the service where we can configure our update policy i.e. the machines we want to update, whether machines should reboot or not, what the update frequency should be, etc. In order to know which updates should be applied, the Automation Account is <u>linked</u> to our Log Analytics so that it can monitor missing updates reported accross all our VMs. If the Automation Accounts sees a VM with a missing patch, it will update it based on the configuration we set up.

## Resources organization
Let's dive into the management of resources in my environment.
A first design decision was to split our architecture based on the different Cloud Service Providers (CSP) we use, which are Azure, GCP, OCI and OVH.
- The first reason why we made this choice is that we wanted to sort machines by CSP as shown on the scheme below. Of course, Azure VMs being already deployed in resource groups, the only resources we deployed in the Azure RG are Log Analytics and Automation Account.
- The second reason why we made this choice is that Azure has some limitation in deploying resources. For example, you can't deploy as much VM as you want. To prevent this kind of limitation, we split the architecture as discussed, but we also split production and non production environnement, altough their organization remains the same.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/arch_2.png" class="img-fluid rounded z-depth-1" %}
</div>

Let's go back to our Azure Policy service. Our goal here is to create the following rules :
- Enforce the Log Analytics agent to be installed on all Azure VMs, and make them report to ari-loga-azure-001
- Enforce the Log Analytics agent to be installed on all OVH Azure ARC VMs, and make them report to ari-loga-ovh-001
- Enforce the Log Analytics agent to be installed on all GCP Azure ARC VMs, and make them report to ari-loga-gcp-001
- Enforce the Log Analytics agent to be installed on all OCI Azure ARC VMs, and make them report to ari-loga-oci-001
By doing this, we ensure that any newly created Azure VM, or any newly onboarded Azure ARC VM will be integrated with the appropriate Log Analytics. In the end, this allows us to fully automate onboarding and update monitoring.
