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
- [Part 4 - Log Analytics agents](/blog/2022/Update-management-03/)
- <b>[Part 5 - Automation accounts (you're here)](/blog/2023/Update-management-04/)</b>
- [Part 6 - Monitoring](/blog/2023/Update-management-05/)
- [Part 7 - Security patches on Azure ARC](/blog/2023/Update-management-06/)
- [Part 8 - Security patches on CentOS machines](/blog/2023/Update-management-07/)

***

## Introduction
In the previous post we saw how the basic and simple properties of a Log Analytics workspace. We also saw how to deploy Log Analytics agents with various methods. In this method, you will find all you need to know about Automation Account and their update management feature.

## Context
This part will be very important since it define all the strategy to configure the Automation Accounts. So, if you remember my introduction post, we want to update machines bases on the following rules : 
- Apply only security and critical patches
- Apply patches once a week
- Reboot only if needed

All of this can be configured in In addition to those rules, we will configure <b>deployment schedules</b>, which are some sort of  automation accounts sub-resources. For each automation account, we will 21 deployement schedule! This is a lot, be let me explain : the goal of this solution is to allow any business application to patched. To make it possible, we need to propose many schedules to comply with all application constraints. So we decided to propose 3 schedule maintenance per day : at 3:00, 12:00, and 22:00 (3*7 = 21 schedules).

My ultimate goal with this is to create a deployment schedule that select the appropriate machines, based on machines tags. Just imagine how simple it would be : if a machine must be updated, I assign a tag to it (the tag <b>MON-03:00</b> for instance), and I know that my deployment schedules will update it. Let's see how to do this!

<table id="custom" class="t-border">
<caption style="text-align:center"><b>Schedules available for VM updates</b></caption>
  <tr>
    <th>MON</th>
    <th>TUE</th>
    <th>WED</th>
    <th>THU</th>
    <th>FRI</th>
    <th>SAT</th>
    <th>SUN</th>
  </tr>
  <tr>
    <td>MON-03:00</td>
    <td>TUE-03:00</td>
    <td>WED-03:00</td>
    <td>THU-03:00</td>
    <td>FRI-03:00</td>
    <td>SAT-03:00</td>
    <td>SUN-03:00</td>
  </tr>
  <tr>
    <td>MON-12:00</td>
    <td>TUE-12:00</td>
    <td>WED-12:00</td>
    <td>THU-12:00</td>
    <td>FRI-12:00</td>
    <td>SAT-12:00</td>
    <td>SUN-12:00</td>
  </tr>
  <tr>
    <td>MON-22:00</td>
    <td>TUE-22:00</td>
    <td>WED-22:00</td>
    <td>THU-22:00</td>
    <td>FRI-22:00</td>
    <td>SAT-22:00</td>
    <td>SUN-22:00</td>
  </tr>
</table>

***

## Automation accounts
### Create deployment schedule
We can create a first deployment schedule to check what it looks like.
1. Create an Automation Account if not done
2. Navigate to your Automation Account
3. Navigate to the <b>Update management</b> pane
4. Click on <b>Schedule update deployment</b>
5. You will see settings to be configured as described in the following table.

<table id="custom" class="t-border">
<caption style="text-align:center"><b>Deployment schedule settings</b></caption>
  <tr>
    <th>Name</th>
    <th>Description</th>
    <th>Value</th>
  </tr>
  <tr>
    <td>Name</td>
    <td>This is the name of the deployment schedule</td>
    <td>LIN-MON-03:00</td>
  </tr>
  <tr>
    <td>Operating System</td>
    <td>Surprise! A deployment schedule is dedicated to only one OS.</td>
    <td>LINUX</td>
  </tr>
  <tr>
    <td>Groups to update</td>
    <td>Dynamic selection of machines to update</td>
    <td>C.f. below</td>
  </tr>
  <tr>
    <td>Machines to update</td>
    <td>Selection of machines to update</td>
    <td>C.f. below</td>
  </tr>
  <tr>
    <td>Update classification</td>
    <td>I mentioned before, we only want security and critical here</td>
    <td>Critical and security updates</td>
  </tr>
  <tr>
    <td>Include/exclude updates</td>
    <td>You may want to include/exclude specific updates. We can leave this blank.</td>
    <td>-</td>
  </tr>
  <tr>
    <td>Pre-script + Post script</td>
    <td>You may want to push and execute script before or after updates. We can leave this blank</td>
    <td>-</td>
  </tr>
  <tr>
    <td>Maintenance window</td>
    <td>Duration of the maintenance schedule.</td>
    <td>120</td>
  </tr>
  <tr>
    <td>Reboot options</td>
    <td>Reboot if required</td>
    <td>120</td>
  </tr>
</table>

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/automation_account_1.png" class="img-fluid rounded z-depth-1" %}
</div>


What we observe so far is that we can't merge Linux and Windows in the same deployment schedule, so our deployment schedules will now look like this : 
<table id="custom" class="t-border">
<caption style="text-align:center"><b>Schedules available for VM updates</b></caption>
  <tr>
    <th>OS</th>
    <th>MON</th>
    <th>TUE</th>
    <th>WED</th>
    <th>THU</th>
    <th>FRI</th>
    <th>SAT</th>
    <th>SUN</th>
  </tr>
  <tr>
    <td>Windows</td>
    <td>WIN-MON-03:00</td>
    <td>WIN-TUE-03:00</td>
    <td>WIN-WED-03:00</td>
    <td>WIN-THU-03:00</td>
    <td>WIN-FRI-03:00</td>
    <td>WIN-SAT-03:00</td>
    <td>WIN-SUN-03:00</td>
  </tr>
  <tr>
    <td>Windows</td>
    <td>WIN-MON-12:00</td>
    <td>WIN-TUE-12:00</td>
    <td>WIN-WED-12:00</td>
    <td>WIN-THU-12:00</td>
    <td>WIN-FRI-12:00</td>
    <td>WIN-SAT-12:00</td>
    <td>WIN-SUN-12:00</td>
  </tr>
  <tr>
    <td>Windows</td>
    <td>WIN-MON-22:00</td>
    <td>WIN-TUE-22:00</td>
    <td>WIN-WED-22:00</td>
    <td>WIN-THU-22:00</td>
    <td>WIN-FRI-22:00</td>
    <td>WIN-SAT-22:00</td>
    <td>WIN-SUN-22:00</td>
  </tr>
    <tr>
    <td>Linux</td>
    <td>LIN-MON-03:00</td>
    <td>LIN-TUE-03:00</td>
    <td>LIN-WED-03:00</td>
    <td>LIN-THU-03:00</td>
    <td>LIN-FRI-03:00</td>
    <td>LIN-SAT-03:00</td>
    <td>LIN-SUN-03:00</td>
  </tr>
  <tr>
    <td>Linux</td>
    <td>LIN-MON-12:00</td>
    <td>LIN-TUE-12:00</td>
    <td>LIN-WED-12:00</td>
    <td>LIN-THU-12:00</td>
    <td>LIN-FRI-12:00</td>
    <td>LIN-SAT-12:00</td>
    <td>LIN-SUN-12:00</td>
  </tr>
  <tr>
    <td>Linux</td>
    <td>LIN-MON-22:00</td>
    <td>LIN-TUE-22:00</td>
    <td>LIN-WED-22:00</td>
    <td>LIN-THU-22:00</td>
    <td>LIN-FRI-22:00</td>
    <td>LIN-SAT-22:00</td>
    <td>LIN-SUN-22:00</td>
  </tr>
</table>

Let's continue with the settings groups to update and machines to update, which are both settings that allow you to choose which machines should be updated with the current deployment schedule.

#### Machines to update
When using this feature, we can define a group of machines to target with the current deployment schedule. This group of machines can be created from multiple sources as discussed in the [Microsoft documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/computer-groups) :
- Log query
- Log search API
- Active Directory
- Configuration Manager
- Windows Server Update Services

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/automation_account_2.png" class="img-fluid rounded z-depth-1" %}
</div>

As mentioned in the first part of this post, I want my VMs to be updated if they have a specific tag. Unfortunately, with this setting, we can't update machines based on their tags! So let's take a look at the <b>Groups to update</b> feature then.

#### Groups to update
As described in the Microsoft documentation, we can filter machines based on their tags, awesome right? So this will be our solution, but our solution has an important limitation : this does not apply to Azure ARC VMs! This is immediately not so awesome... But I have a workaround that I will describe in a future post, so let's continue with Azure VM only for now.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/automation_account_3.png" class="img-fluid rounded z-depth-1" %}
</div>

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/automation_account_4.png" class="img-fluid rounded z-depth-1" %}
</div>


I recommend to do some testing if you want to get familiar with deployment schedules capabilities. If you want to go straight foreward, let's deploy some stuff now!

***

## Deployment
Of course you won't have to do anything here except running some commands, because I provided all the ARM templates that you need to deploy a small core infrastructure.
1. Create a resource group
2. 

```
az login
az account set --subscription "67c4c038-b3aa-4157-afab-73f1d00dfe82"
az deployment group create --name myDeployment --resource-group updtmgmt-rg --template-file  --parameters
```

Here is a sample output for deployment.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/automation_account_5.png" class="img-fluid rounded z-depth-1" %}
</div>

***

What did we deployed?
We deployed the three following resources.
<table id="custom" class="t-border">
<caption style="text-align:center"><b>Resource deployed</b></caption>
  <tr>
    <th>Resource name</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>Updates(updtmgmt-la)</td>
    <td>This is a resource that is deployed on top of a Log Analytics, and that usually provide graphs. To be honest I never reversed engineered this to understand how it works, but I think this enables charts to be displayed within the automation account.</td>
  </tr>
  <tr>
    <td>updtmgmt-aa</td>
    <td>This is the automation account, which is linked to the Log Analytics workspace</td>
  </tr>
  <tr>
    <td>updtmgmt-la</td>
    <td>This is the log analytics workspace that collects our update data</td>
  </tr>
</table>

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/automation_account_6.png" class="img-fluid rounded z-depth-1" %}
</div>

We can't see deployment schedule here : we can view them from the automation account, so navigate to the resource, and you will see two deployment schedules that look the same, expect one is for Windows OS, the other for Linux.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/automation_account_7.png" class="img-fluid rounded z-depth-1" %}
</div>

If you take a look at the Linux deployment schedule, you will see the configuration we pushed through our ARM template. In the <b>Groups to update</b> setting, we look for machines located in the current subscription and tagged with <b>patch:LINUX-MON-03:00</b> tag.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/automation_account_8.png" class="img-fluid rounded z-depth-1" %}
</div>