---
layout: post
title: Azure Update Management - Part 5 - Monitoring
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
- [Part 1 - Architecture](/al-folio/blog/2021/Update-management-01/)
- [Part 2 - Azure ARC](/al-folio/blog/2021/Update-management-02/)
- [Part 3 - Log Analytics agents](/al-folio/blog/2021/Update-management-03/)
- [Part 4 - Automation accounts](/al-folio/blog/2021/Update-management-04/)
- <b>[Part 5 - Monitoring (you're here)](/al-folio/blog/2021/Update-management-05/)</b>

