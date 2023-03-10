---
layout: post
title: Azure Update Management - Part 4 - Log Analytics agents
date: 2022-09-02 10:00:00
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
- <b>[Part 4 - Log Analytics agents (you're here)](/blog/2022/Update-management-03/)</b>
- [Part 5 - Automation accounts](/blog/2023/Update-management-04/)
- [Part 6 - Monitoring](/blog/2023/Update-management-05/)
- [Part 7 - Security patches on Azure ARC](/blog/2023/Update-management-06/)
- [Part 8 - Security patches on CentOS machines](/blog/2023/Update-management-07/)

***

## Introduction
In the previous post, we saw how to deploy Azure ARC. Now that all our machines are ready to be onboarded on the solution, let's take a closer look at the core architecture, begining with Log Analytics.

## Log Analytics workspaces
### Useful information
The Log Analytics service was designed to store logs for a defined period of time (retention). You can retrieve these logs by sending queries to the Log Analytics using a powerful language named Kusto Query Language (KQL). If you have monitoring routines, you can save queries as <b>Saved Searches</b> : this is important for the next part. The last thing that is useful to know in the context of this project is about billing : billing should be always configured on a Pay-As-You-Go basis when first deploying the resource (it can be changed later).

### ARM template
The Log Analytics workspace looks like this in an ARM object. We have the main object which is the Log Analytics, and we have a child object which is a <b>savedSearches</b>. There are some properties like "feature" that are mandatory but we can ignore, I already told you about the most important ones.
{% highlight json %}
{
  "type": "Microsoft.OperationalInsights/workspaces",
  "apiVersion": "2020-08-01",
  "name": "[parameters('workspaceName')]",
  "location": "westeurope",
  "properties": {
      "sku": {
          "name": "[parameters('sku')]"
      },
      "retentionInDays": "[parameters('dataRetention')]",
      "features": {
          "searchVersion": 1,
          "legacy": 0
      }
  },
  "resources": [{
      "apiVersion": "2020-08-01",
      "type": "savedSearches",
      "name": "[variables('saved_search')]",
      "dependsOn": [
          "[concat('Microsoft.OperationalInsights/workspaces/', parameters('workspaceName'))]"
      ],
      "properties": {
          "category": "UpdateManagement",
          "displayName": "[variables('saved_search')]",
          "functionAlias": null,
          "functionParameters": null,
          "query": "Heartbeat | where Computer contains 'a-string-that-cant-be-matched' | distinct Computer",
          "tags": [{
              "name": "Group",
              "value": "Computer"
          }],
          "version": 2
      }
  }]
}
{% endhighlight json %}

## Log Analytics agents
### About Microsoft agents
Let's take some time to talk about Microsoft agents, because I know this is often confusing. There is a [Microsoft documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/agents-overview) page that will help you understand all of this. Let's me sum up this in the following table.

#### Windows
<table id="custom" class="t-border">
<caption style="text-align:center"><b>For Windows machines</b></caption>
  <tr>
    <th style="text-align:center"></th>
    <th style="text-align:center">Azure Monitor Agent</th>
    <th style="text-align:center">Log Analytics Agent</th>
    <th style="text-align:center">Diagnostics extension (WAD)</th>
  </tr>
  <tr>
    <td style="text-align:center"><b>Data collected</b></td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td style="text-align:center">Event Logs</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td style="text-align:center">Performance</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td style="text-align:center">File based logs</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td>IIS logs</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td>ETW events</td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td>.NET app logs</td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td>Crash dumps</td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td>Agent diagnostics logs</td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td><b>Services and features supported</b></td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>Microsoft Sentinel</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>VM Insights</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>Microsoft Defender for Cloud</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>Update Management</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>Change Tracking</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>SQL Best Practices Assessment</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
  </tr>
</table>

#### Linux
<table id="custom" class="t-border">
<caption style="text-align:center"><b>For Linux machines</b></caption>
  <tr>
    <th style="text-align:center"></th>
    <th style="text-align:center">Azure Monitor Agent</th>
    <th style="text-align:center">Log Analytics Agent</th>
    <th style="text-align:center">Diagnostics extension (LAD)</th>
  </tr>
  <tr>
    <td style="text-align:center"><b>Data collected</b></td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td style="text-align:center">Syslog</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td style="text-align:center">Performance</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
  </tr>
  <tr>
    <td><b>Services and features supported</b></td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>Microsoft Sentinel</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>VM Insights</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>Microsoft Defender for Cloud</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>Update Management</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>Change Tracking</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
  </tr>
  <tr>
    <td>SQL Best Practices Assessment</td>
    <td style="text-align:center">X</td>
    <td style="text-align:center"></td>
    <td style="text-align:center"></td>
  </tr>
</table>

### Deployment methods
#### Extension
#### Script
#### Policy