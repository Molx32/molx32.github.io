---
layout: post
title: Azure Update Management - Part 6 - Monitoring
date: 2023-02-02 10:00:00
description: This post is describes the architecture of the solution
tags: updatemanagement
authors:
  - name: Molx32

bibliography: 2022-08-01-Update management.bib

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
- [Part 5 - Automation accounts](/blog/2023/Update-management-04/)
- <b>[Part 6 - Monitoring (you're here)](/blog/2023/Update-management-05/)</b>
- [Part 7 - Security patches on Azure ARC](/blog/2023/Update-management-06/)
- [Part 8 - Security patches on CentOS machines](/blog/2023/Update-management-07/)

## Purpose
In this article, I assume you connected all your virtual machines. What you may want now is monitoring you virtual machines. The usual way to monitor updates is navigate to the automation account, then take a look at your VMs.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/updt_mgmt_monitoring_01.png" class="img-fluid rounded z-depth-1" %}
</div>

This is a very poor view, and we sure can do way better! And I actually developed a ready to use monitoring workbook that you need to slightly tweak to meet your needs. Ready to discover that sick workbook? Here is a short video.
<div class="col-sm mt-3 mt-md-0">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/6asRTPWqYmg" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

## Deploy
To get the same awesome dashboard, just copy the code hosted on my [Github](https://github.com/Molx32/AwesomeAzureWorkbooks/blob/main/Workbooks/update-management.json), and follow these steps :
1. Create a new workbook
2. Edit the workbook and select the advanced editor (gallery template)
3. Paste the code
4. Save

You should end up with something that looks like this. Let's adapt this to make it work.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/updt_mgmt_monitoring_00.png" class="img-fluid rounded z-depth-1" %}
</div>


## Adapt
### Customize general filters
The goal of this section is to show how to create filters button so that you can customize them. In my case, the update management architecture is made of the of multiple Log Analytics workspaces that collect VMs update data. In order to monitor data only on those Log Analytics workspaces, we tag them with the tag <b>application:update management</b> to that we can filter them. Then we also add the following tags to our log analytics based on their CSP and their environment. This is summed up in the following table.


<table id="custom" class="t-border">
<caption style="text-align:center"><b>Resource deployed</b></caption>
  <tr>
    <th>Log Analytics name</th>
    <th>Second priority</th>
    <th>CSP tag</th>
    <th>Environment tag</th>
    <th>Application tag</th>
  </tr>
  <tr>
    <td>loga-azure-001</td>
    <td>Azure</td>
    <td>azure</td>
    <td>prod</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>loga-oci-001</td>
    <td>Azure</td>
    <td>oci</td>
    <td>prod</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>loga-gcp-001</td>
    <td>Azure</td>
    <td>gcp</td>
    <td>prod</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>loga-ovh-001</td>
    <td>Azure</td>
    <td>ovh</td>
    <td>prod</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>loga-np-azure-001</td>
    <td>Azure</td>
    <td>azure</td>
    <td>no prod</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>loga-np-oci-001</td>
    <td>Azure</td>
    <td>oci</td>
    <td>no prod</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>loga-np-gcp-001</td>
    <td>Azure</td>
    <td>gcp</td>
    <td>no prod</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>loga-np-ovh-001</td>
    <td>Azure</td>
    <td>ovh</td>
    <td>no prod</td>
    <td>update management</td>
  </tr>
</table>

Once this has been done, let's create our filter buttons by editing and add <b>Add links/tabs</b>.
#### Create a first parameter named CSP
This will create a drop down button where we can select the different CSPs where our VMs are deployed, in order to filter our view by CSP.
1. Make it a drop down
2. Make it a required parameter
3. Allow multiple selection
4. Set a static JSON and save

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/updt_mgmt_monitoring_021.png" class="img-fluid rounded z-depth-1" %}
</div>

#### Create a first parameter named Environment
This will create a drop down button where we can select the environment (prod, no prod) in order to filter our view by environment.
1. Make it a resource picker
2. Make it a required parameter
3. Allow multiple selection
4. Set the query as shown on the screen shot

This request selects all Log Analytics workspaces that have the tag <b>application:update management</b>, and returns the list of all unique <b>environment</b> tags positionned on these Log Analytics.

{% highlight kql %}
where type =~ 'microsoft.operationalinsights/workspaces'
<td>where tolower(tostring(tags.application)) == "update management"
<td>distinct tostring(tags.environment)
{% endhighlight %}

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/updt_mgmt_monitoring_022.png" class="img-fluid rounded z-depth-1" %}
</div>

#### Create a first parameter named Workspaces
The selected values in our two previous buttons will make this third and last button to return different values.
1. Make it a resource picker
2. Make it a required parameter
3. Allow multiple selection
4. Check the "Hide parameter in reading mode" feature
5. Set the query as shown on the screen shot


This request selects all Log Analytics workspaces that have the tag <b>application:update management</b>, and return all Log Analytics workspaces that have CSP tag and Environment tags that matches the selection we made using the previous buttons.
{% highlight kql %}
where type =~ 'microsoft.operationalinsights/workspaces'
<td>where tolower(tostring(tags.application)) == "update management"
<td>distinct tostring(tags.environment)
{% endhighlight %}

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/updt_mgmt_monitoring_023.png" class="img-fluid rounded z-depth-1" %}
</div>

***

If you didn't change too much things, the result for this first step should look like this.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/updt_mgmt_monitoring_024.png" class="img-fluid rounded z-depth-1" %}
</div>


### Customize job filters
In my case, VMs have specific tags to easily know when they are supposed to update and reboot. For instance, a production machine (P) that updates and reboots on wednesday (WED) at 12:00 (12:00) will be tagged with <b>patch:P-WED-12:00</b>. Because I have a lot of machines, I wanted to filter the jobs so that I can observe the jobs of all machines that updated on a specific day, or a specific hour.

Based on your needs, you either remove or adapt what is highlighted on the following screenshot.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/updt_mgmt_monitoring_03.png" class="img-fluid rounded z-depth-1" %}
</div>


## Additional info
So what do we have to monitor our VMs? Our VMs send the followinf logs, thanks to the Log Analytics agent that are runnin on them:
- The <i>Update</i> table represents updates available and their installation status for a machine. This is table we will mainly use.
- The <i>UpdateRunProgress</i> table provides update deployment status of a scheduled deployment by machine. It will show the status of update deployments i.e. whether it was successful or not.
- The <i>UpdateSummary</i> table provides update summary by machine. This is some sort of agregrate of the Update table.

You can find some query examples using these tables in the [Microsoft documentation](https://learn.microsoft.com/en-us/azure/automation/update-management/query-logs).
