---
layout: post
title: Azure Update Management - Part 2 - Deploy Azure ARC
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
- [Part 0 - Introduction](/blog/2021/Update-management-00/)
- [Part 1 - Architecture](/blog/2021/Update-management-01/)
- <b>[Part 2 - Azure ARC (you're here)](/blog/2021/Update-management-02/)</b>
- [Part 3 - Log Analytics agents](/blog/2021/Update-management-03/)
- [Part 4 - Automation accounts](/blog/2021/Update-management-04/)
- [Part 5 - Monitoring](/blog/2021/Update-management-05/)

In that part, I will describe what is Azure ARC, and how to deploy it.

## Prerequisites
All the prerequisites are documented by Microsoft in their [documentation](https://learn.microsoft.com/en-us/azure/azure-arc/servers/prerequisites).
### Supported OS
Unfortunately, not all OSs are supported by Azure ARC. According to the Microsoft documentation, here are the supported OSs.
- Microsoft
  - Windows Server 2022
  - Windows Server 2019
  - Windows Server 2016
  - Windows Server 2012 R2
  - Windows Server 2008 R2 SP1
  - Windows IoT Enterprise
  - Azure Stack HCI
- Linux
  - CentOS 7, 8
  - Rocky Linux 8
  - Oracle Linux 7, 8
  - RHEL 7, 8, 9
  - SLES 12, 15
  - Ubuntu 16.04, 18.04, 20.04, 22.04 LTS
  - Debian 10, 11
  - Amazon Linux 2

### Network requirements


### System requirements
For Windows systems :
- NET Framework 4.6 or later
- Windows PowerShell 4.0 or later

For Linux systems :
- systemd
- wget
- openssl
- gnupg

### Azure resource providers
As you may know, all the different services that exist in Azure are provided by what is called <b>resource providers</b>. These can be regarded as modules that may or may not be enabled by default. In order for Azure ARC to work properly, we need to make sure these providers are enabled.
- Microsoft.HybridCompute
- Microsoft.GuestConfiguration
- Microsoft.HybridConnectivity

You can use Azure Powershell or Azure CLI to enabled them.
{% highlight powershell %}
Connect-AzAccount
Set-AzContext -SubscriptionId [subscription you want to onboard]
Register-AzResourceProvider -ProviderNamespace Microsoft.HybridCompute
Register-AzResourceProvider -ProviderNamespace Microsoft.GuestConfiguration
Register-AzResourceProvider -ProviderNamespace Microsoft.HybridConnectivity
{% endhighlight %}

## Deployment
### Create a service principal
In order for the VMs to be onboarded, we need to use a service principal.