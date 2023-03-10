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
    <td>R-MON-03:00</td>
    <td>R-TUE-03:00</td>
    <td>R-WED-03:00</td>
    <td>R-THU-03:00</td>
    <td>R-FRI-03:00</td>
    <td>R-SAT-03:00</td>
    <td>R-SUN-03:00</td>
  </tr>
  <tr>
    <td>R-MON-12:00</td>
    <td>R-TUE-12:00</td>
    <td>R-WED-12:00</td>
    <td>R-THU-12:00</td>
    <td>R-FRI-12:00</td>
    <td>R-SAT-12:00</td>
    <td>R-SUN-12:00</td>
  </tr>
  <tr>
    <td>R-MON-22:00</td>
    <td>R-TUE-22:00</td>
    <td>R-WED-22:00</td>
    <td>R-THU-22:00</td>
    <td>R-FRI-22:00</td>
    <td>R-SAT-22:00</td>
    <td>R-SUN-22:00</td>
  </tr>
</table>

