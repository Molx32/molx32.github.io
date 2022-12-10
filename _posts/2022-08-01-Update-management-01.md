---
layout: post
title: Azure Update Management - Part 1 - Architecture
date: 2021-08-02 10:00:00
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
Hello there, the goal of this serie is to describe a real world implementation of Azure Update Management. This is a service designed to update any machine in your infrastructure, whether they are hosted on Azure or elsewhere, provided your OSs are technically supported. I'll try to give a comprehensive feedback and drop advices here and there. Have a nice reading.

#### Plan
- [Part 0 - Introduction](/al-folio/blog/2021/Update-management-00/)
- <b>[Part 1 - Architecture (you're here)](/al-folio/blog/2021/Update-management-01/)</b>
- [Part 2 - Azure ARC](/al-folio/blog/2021/Update-management-02/)
- [Part 3 - Log Analytics agents](/al-folio/blog/2021/Update-management-03/)s
- [Part 4 - Automation accounts](/al-folio/blog/2021/Update-management-04/)
- [Part 5 - Monitoring](/al-folio/blog/2021/Update-management-05/)

## Architecture
In this guide to Azure Update Management, we are going to dig into the details of the service. Azure Update Management is composed of multiple Azure services.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/arch_1.png" class="img-fluid rounded z-depth-1" %}
</div>
1. <b>Azure ARC</b> - First of all, non Azure VMs need an additional step : connecting the VM to Azure. This is can be achieved with Azure ARC
2. <b>Reporting with Log Analytics agents</b> - Now that we can manage both Azure and non-Azure VM directly from Azure, we can set up an Azure Policy. Basically, an Azure policy can enforce a pre-defined configuration on our VM. In that case, we enforce the Log Analytics agent to be installed on each VM (both Azure and non-Azure) and report logs to a Log Analytics workspace. Be doing this, any newly created VM will be onboarded on Update Management.
3. <b>Monitoring with workbooks</b> - In order to monitor service health, update status, and some other information, we can create an Azure Workbook, which is a dashboard. The workbook is based on the Log Analytics workspace that will collect all the update information from all VMs.
4. <b>Updating with Automation Accounts.</b> - The core service of Update Management in Azure is <b>Automation Account</b>. This is the service where we can configure our update policy i.e. the machines we want to update, whether machines should reboot or not, what the update frequency should be, etc. In order to know which updates should be applied, the Automation Account is <u>linked</u> to our Log Analytics so that it can monitor missing updates reported accross all our VMs. If the Automation Accounts sees a VM with a missing patch, it will update it based on the configuration we set up.


