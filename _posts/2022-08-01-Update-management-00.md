---
layout: post
title: Azure Update Management - Part 0 - Introduction
date: 2022-08-01 10:00:00
description: This post is an introduction that describes the business environment
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
- <b>[Part 0 - Introduction (you're here)](/blog/2021/Update-management-00/)</b>
- [Part 1 - Architecture](/blog/2021/Update-management-01/)
- [Part 2 - Azure Policy](/blog/2021/Update-management-011/)
- [Part 3 - Azure ARC](/blog/2021/Update-management-02/)
- [Part 4 - Log Analytics agents](/blog/2021/Update-management-03/)
- [Part 5 - Automation accounts](/blog/2021/Update-management-04/)
- [Part 6 - Monitoring](/blog/2021/Update-management-05/)

## <b>Scenario</b>
This scenario may differ from your use cases, but keep in mind that this is a real world scenario ! Thus, you may find interesting tips and information to build a service that fits your needs.
Because it is an overview, you will find a lot of information in this section, and you may not have a clear picture of how to set up all of this. Don't worry, technical stuff will be described in the next posts.
### Update policy
It is important that you define an update policy in the early stage of the project. In my case, the update policy is the following :
1. <b>We want to apply only security or critical patches.</b> Here are the descriptions of each type, from the [Microsoft documentation](https://docs.microsoft.com/en-us/troubleshoot/windows-client/deployment/standard-terminology-software-updates)
  - <i>Critical updates</i> - A widely released fix for a specific problem that addresses a critical, non-security-related bug.
  - <i>Security updates</i> - An update that collects all the new security updates for a given month and for a given product, addressing security-related vulnerabilities. It's distributed through Windows Server Update Services (WSUS), System Center Configuration Manager and Microsoft Update Catalog. Security vulnerabilities are rated by their severity. The severity rating is indicated in the Microsoft security bulletin as critical, important, moderate, or low. This Security-only update would be displayed under the title Security Only Quality Update when you download or install the update. It will be classified as an Important update.
2. <b>We want to apply those patches once a week.</b> This can be considered being a high update frequency : the reason of this choice is that the security team required defined SLA to handle and mitigate vulnerabilities, based on their [CVSS](https://www.first.org/cvss/). By applying patches once a week, we ensure that critical vulnerabilities that need a patch will be patched within a week.
  - For a vulnerability with CVSS > 10, remediate as soon as possible
  - For a vulnerability with CVSS >= 9, remediate within 2 weeks
  - For a vulnerability with CVSS >= 7, remediate within 3 months
  - For a vulnerability with CVSS < 7, remediate within 6 months
3. <b>We want to reboot only if needed.</b> One of the major constraints regarding patches is the need to reboot the VM to apply patches, because servers may host critical business applications and must provide a service continuity. Altough security patches usually don't require to reboot servers, it is still necessary to define maintenance schedules i.e. a timeframe when we can reboot the server. In our case the maintenance schedule will last 2 hours. If the machine can't update during this timeframe, then the patching process is stopped.
4. <b>We want to update only backuped machines.</b> In the case the applied updates have a side effect, we want to make sure that all the machines are backuped, so we can rollback any time. A concrete example on Windows Server 2016 is the <b>[C01A001D](https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/unresponsive-vm-apply-windows-update#resolution).

<u>Note</u> : it is important to keep in mind that update management is not the one and only solution to keep your VMs safe. Indeed, OS may no longer be supported, and vulnerabilities must sometimes be mitigated by configuring specific settings rather than updating.


### Environment
In this scenario, we assume we have a thousand servers distributed across Azure and multiple other cloud or non-cloud environments :
* OCI
* OVH
* GCP
* On-premise
* Etc.

Since Update management is an Azure service, you may think that this is easier to set up for Azure VM, and this is true.
* For Azure VMs, we only need to deploy an agent, the <i>Log Analytics agent</i> on the machine to collect its logs and know which updates are missing.
* For non Azure VMs, we first need to establish a link between the VM and Azure using the Azure ARC agent, so that it can be managed like any other resource from Azure. Once this is done, the we only also to deploy the <i>Log Analytics agent</i>, just like Azure VMs.

### Operating systems
You can find a list of supported OS in the [Microsoft documentation](https://docs.microsoft.com/en-us/azure/automation/update-management/operating-system-requirements). 
<table class="t-border">
  <tr>
    <th>Windows OS</th>
    <th>Linux OS</th>
  </tr>
  <tr>
    <td>Windows Server 2019</td>
    <td>CentOS 6, 7, and 8</td>
  </tr>
  <tr>
    <td>Windows Server 2016</td>
    <td>Oracle Linux 6.x, 7.x, 8x</td>
  </tr>
  <tr>
    <td>Windows Server 2012 R2</td>
    <td>Red Hat Enterprise 6, 7, and 8</td>
  </tr>
  <tr>
    <td>Windows Server 2012</td>
    <td>SUSE Linux Enterprise Server 12, 15, and 15.1</td>
  </tr>
  <tr>
    <td>Windows Server 2008 R2</td>
    <td>Ubuntu 14.04 LTS, 16.04 LTS, 18.04 LTS, and 20.04 LTS</td>
  </tr>
</table>