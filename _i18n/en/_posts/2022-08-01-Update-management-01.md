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
- <b>[Part 1 - Architecture (you're here)](/blog/2022/Update-management-01/)</b>
- [Part 2 - Azure Policy](/blog/2022/Update-management-011/)
- [Part 3 - Azure ARC](/blog/2022/Update-management-02/)
- [Part 4 - Log Analytics agents](/blog/2022/Update-management-03/)
- [Part 5 - Automation accounts](/blog/2023/Update-management-04/)
- [Part 6 - Monitoring](/blog/2023/Update-management-05/)
- [Part 7 - Security patches on Azure ARC](/blog/2023/Update-management-06/)
- [Part 8 - Security patches on CentOS machines](/blog/2023/Update-management-07/)

***

## Introduction
In the previous, we just scratched the surface of this wide topic. In this post, we'll see the complete architecture as deployed in my production environment. I'll describe each service and their interaction, as well as design considerations.

## Architecture
### Services
Here is the architecture, where you can see four steps.
1. <b>Azure ARC</b> - First of all, non Azure VMs need an additional step : connecting the VM to Azure. This is can be achieved with Azure ARC.

2. <b>Reporting with Log Analytics agents</b> - Now that we can manage both Azure and non-Azure VM directly from Azure, we can set up an Azure Policy. Basically, an Azure policy can enforce a pre-defined configuration on our VM. In that case, we enforce the Log Analytics agent to be installed on each VM (both Azure and non-Azure) and report update information to a Log Analytics workspace. Be doing this, any newly created VM will be onboarded on Update Management.

<div class="col-sm mt-3 mt-md-0">
  {% include blog/figure.html path="assets/img/arch_1.png" class="img-fluid rounded z-depth-1" %}
</div>

3. <b>Monitoring with workbooks</b> - In order to monitor service health, update status, and some other information, we can create an Azure Workbook, which is a dashboard. The workbook is based on the Log Analytics workspace that will collect all the update information from all VMs.

4. <b>Updating with Automation Accounts.</b> - The core service of Update Management in Azure is <b>Automation Account</b>. This is the service where we can configure our update policy <i>i.e.</i> the machines we want to update, whether machines should reboot or not, what the update frequency should be, and so on. In order to know which updates should be applied, the Automation Account is <u>linked</u> to our Log Analytics so that it can monitor missing updates reported accross all our VMs. If the Automation Accounts sees a VM with a missing patch, it will consider the VM as a candidate for update.

### Resources organization
Now, to explain how resources are organized, let me bring some context from my real-world scenario. We have four Cloud Service Providers (CSP) : OVH, GCP, OCI and Azure. The question is : how do we organize our resources to match the architecture presented above?

#### Design consideration - Environment
Our first consideration was to integrate this properly with the existing architecture, which looked like this. As you can see, there are multiple resource groups (RG) and one subscription per environment : the production environment, and the non production environment. Thus, to comply with the existing architecture, we could create two resource groups:
1. A resource group named "RG-UPDT-MGMT" where we would deploy production resources
2. A resource group named "RG-NP-UPDT-MGMT" where we would deploy non production resources

We didn't implement this, because we also wanted to consider CSPs.

<div class="col-sm mt-3 mt-md-0">
  {% include blog/figure.html path="assets/img/arch_11.png" class="img-fluid rounded z-depth-1 imgc" %}
</div>

#### Design consideration - CSP
A second design consideration was our CSPs. Imagine onboarding hundreds of VMs into Azure using Azure ARC : you don't want extra maintenance effort on Azure. For this reasons, we wanted to be able to manage those VMs easily by regrouping all OVH machines in the same resource group, and doing the same with OCI and GCP. Of course, we couldn't do it for Azure VMs because we already had many projects and resources with Azure VMs everywhere, so this ease of management only applies to Azure ARC. So this gives the following scheme.

<div class="col-sm mt-3 mt-md-0">
  {% include blog/figure.html path="assets/img/arch_12.png" class="img-fluid rounded z-depth-1 imgc" %}
</div>

<div class="col-sm mt-3 mt-md-0">
  {% include blog/figure.html path="assets/img/arch_13.png" class="img-fluid rounded z-depth-1 imgc" %}
</div>

<u>Notes</u> :
- To make the scheme more readable, I removed the example resource groups that you can see on the previous scheme.
- I split the scheme into two images

#### Additional design decisions
##### Monitoring
Monitoring could be a design consideration, but if you plan to use Azure Monitor and workbooks, you can monitor updates accross resource groups and subscriptions.

##### Azure limits
If you have many resources in your environment, make sure to check Azure services limits.

##### Permissions
If you split your resources in many resource groups (e.g. one resource group per Azure region, per environment, per CSP, per country), permissions will be painful to manage.

#### Final core architecture
So in my case, the final architecture looks like this. It is important to note that Automation Account configuration is slightly different to handle Azure ARC machines : I will discuss these technical details in the next parts of the serie. Additionally, you can see that some services are missing : they will also be discussed in the next posts.

<div class="col-sm mt-3 mt-md-0">
  {% include blog/figure.html path="assets/img/arch_2.png" class="img-fluid rounded z-depth-1" %}
</div>

<u>Note</u> : I won't post the same picture for non production environment since this is the same, expect for resource groups and resources names.

## Deployment
Wait a minute! You may want to deploy resources at this point : I invite you to do some testing if you want to, but the production deployment should be done using Infrastructure As Code (IAC). If you take a look a the next posts, I will share an ARM template to do so, probably on <b>Part 4</b>, so be patient! :)