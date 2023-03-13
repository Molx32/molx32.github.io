---
layout: post
title: Azure Update Management - Part 7 - Apply security patches to CentOS!
date: 2023-02-16 10:00:00
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
- [Part 5 - Automation accounts](/blog/2023/Update-management-04/)
- [Part 6 - Monitoring](/blog/2023/Update-management-05/)
- <b>[Part 7 - Security patches on Azure ARC (you're here)](/blog/2023/Update-management-06/)</b>
- [Part 8 - Security patches on CentOS machines](/blog/2023/Update-management-07/)

***

## Introduction
So what's wrong with Azure ARC? We saw many things in the previous posts, but what we did not see is how to integrate Azure ARC VMs on dynamic patches using tags. Let's go!

## Context
So, what’s wrong with Azure ARC? Let’s compare dynamic onboarding between Azure VMs and Azure ARC VMs.

##### Dynamic onboarding for Azure VM
In that case, as the image shows, we can define up to four criteria on which the deployment schedule will rely to onboard VMs at patching time. Unfortunately, this is only available for Azure VMs.

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/patch_arc_1.png" class="img-fluid rounded z-depth-1" %}
</div>

##### Dynamic onboarding for Azure ARC VM
As you can see here, Azure ARC VMs can’t be onboarded using the same four criteria. Instead, we need to configure a <b>saved search</b>, which is a simple KQL query saved within the Log Analytics workspace attached to our Automation Account.

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/patch_arc_2.png" class="img-fluid rounded z-depth-1" %}
</div>

If we take a look at the queries [Microsoft documentation](https://learn.microsoft.com/en-us/azure/automation/update-management/configure-groups#define-dynamic-groups-for-non-azure-machines), we can see that saved searches, also known as <b>computer groups</b> rely on tables such as <i>Heartbeat</i>, or <i>Update</i>. The following example shows a request that returns all computers that have the <b>srv</b> string in their name. So this would be interesting to filter on the computer tag rather then its name… but <u>this is not possible : we can’t find any tag column in any table</u> of the Log Analytics workspace. <b>This is what’s wrong this Azure ARC VMs</b>.
```
Heartbeat
| where Computer contains "srv"
| distinct Computer
```
The next section describe our workaround to still dynamically onboard those Azure ARC VMs dynamically, based on their tags.

***

## The insane workaround!
The basic intent is very simple : automate the process of assigning an Azure ARC VM to a deployment schedule. To do it manually, go on a deployment schedule, click on <b>Machines to update</b>, look for the machines to onboard, click on them and save.

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/patch_arc_3.png" class="img-fluid rounded z-depth-1" %}
</div>

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/patch_arc_4.png" class="img-fluid rounded z-depth-1" %}
</div>

### Script logic
The script in run every day between 15:00 and 18:00, week ends included. When a tag is changed on a VMs, schedules will be updated after the script run. There is one script per CSP resource group as shown on the scheme below.

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/patch_arc_5.png" class="img-fluid rounded z-depth-1" %}
</div>

#### Step 0
{% highlight python %}
#!/usr/bin/env python3
from azure.identity import DefaultAzureCredential
from azure.mgmt.automation import AutomationClient
from azure.mgmt.automation.models import NonAzureQueryProperties
from azure.mgmt.hybridcompute import HybridComputeManagementClient
from azure.mgmt.loganalytics import LogAnalyticsManagementClient
from azure.loganalytics import LogAnalyticsDataClient
from azure.loganalytics.models import QueryBody
from azure.monitor.query import LogsQueryClient, LogsQueryStatus
from datetime import datetime, timezone, timedelta
import automationassets
from automationassets import AutomationAssetNotFound
from azure.core.exceptions import HttpResponseError

# Handle query output
import pandas as pd

# System lib
import sys
import random
import time
import os

# Retrieve all variable that are set in our automation account
automation_account_name = automationassets.get_automation_variable("aa_name")
resource_group_name = automationassets.get_automation_variable("resource_group_name")
subscription_id = automationassets.get_automation_variable("subscription_id")
saved_searche_id = automationassets.get_automation_variable("saved_search_name")

# Authenticate
# This function will use the automation account identity to get an authentication token.
# This token will then be used to access all the APIs we need.
token_credential = DefaultAzureCredential()

# Instanciate clients
# In order to communicate with the different APIs, it is necessary to instanciate clients.
#   - The 'AutomationClient' will be used to get and update deployment schedules
#   - The 'LogAnalyticsManagementClient' will be used to get the Log Analytics that is associated to our Automation Account
#   - The 'HybridComputeManagementClient' will be used to get and modify Azure ARC VMs
#   - The 'LogsQueryClient' will be used to send KQL queries to the Log Analytics workspace and fetch results
automation_client = AutomationClient(token_credential, subscription_id)
log_analytics_client = LogAnalyticsManagementClient(token_credential, subscription_id)
azure_arc_client = HybridComputeManagementClient(token_credential, subscription_id)
log_analytics_data_client = LogsQueryClient(token_credential)
{% endhighlight %}


#### Step 1
In this next step, we want to make sure that each machine that we handle is actually connected to Azure, otherwise this will generate errors or unexpected behaviors. To achieve this, we send a request to the Log Analytics workspace that collects all the Heartbeats.

Then, for each connected machine, we must retrieve its tag. Since the Heartbeat table doesn’t return tags, we look for them in the Azure ARC service.
1. Query Log Analytics workspace to get VMs connected to Azure
2. For each connected machine
  - Get its tags using the Azure ARC API

{% highlight python %}
### 1. GET ALL CONNECTED MACHINES ###
# This block of code is used to retrieve the Log Analytics workspace ID. This ID will be used to send
# the KQL query below to the Log Analytics workspace.
query = "Heartbeat | distinct Computer, ResourceGroup, OSType, Resource"
workspace_resource_id = automation_client.linked_workspace.get(resource_group_name, automation_account_name).id
workspace_name = automation_client.linked_workspace.get(resource_group_name, automation_account_name).id.split('/')[-1]
workspace_id = log_analytics_client.workspaces.get(resource_group_name, workspace_name).customer_id
workspace = automation_client.linked_workspace.get(resource_group_name, automation_account_name)

# The following block of code will send the KQL query to the Log Analytics workspace. We pass a 'timespan' parameter in the query
# to tell the Log Analytics workspace that we only want to make the query on the last 24 hours (1 day) of data.
end_time=datetime.now(timezone.utc)
start_time = end_time - timedelta(days=1)
response = log_analytics_data_client.query_workspace(workspace_id, query, timespan=(start_time, end_time))

# Block of code to handle the result of the query that is store in the 'data' variable.
if response.status == LogsQueryStatus.PARTIAL:
	error = response.partial_error
	data = response.partial_data
	print("Failed to retrieve connected machines")
elif response.status == LogsQueryStatus.SUCCESS:
	data = response.tables

### 2. RETRIEVE TAG FOR EACH CONNECTED MACHINE ###
# In this next block of code, we iterate over all our connected machine, and we make a request to the Azure ARC to get the VM tags.
# Once all the data is retrieved, it is added to a list of dictionnaries named 'connected_machines'.
connected_machines = []
for table in data:
	df = pd.DataFrame(data=table.rows, columns=table.columns)
	# print(df)
	for index, row in df.iterrows():
		# Local vars
		machine_name = row[0]
		machine_rg = row[1]
		machine_os = row[2]
		machine_resource = row[3]
		machine_tag = ""

		# Get tag using Azure ARC API
		try:
			arc_vm = azure_arc_client.machines.get(machine_rg, machine_resource)
			machine_tag = arc_vm.tags.get('patch')
		except:
			print(machine_resource)
			print(machine_rg)
			print("|---- Machine " + machine_name + " not found in Azure ARC")

		dic = {'Computer':machine_name, 'ResourceGroup':machine_rg, 'OSType':machine_os, 'Resource':machine_resource, 'Tag':machine_tag}
		connected_machines.append(dic)
{% endhighlight %}

At the end of this second step we have a list of machines that are eligible to be updated. In the next step, we will assign all these machines to different deployment schedules based on their tags.



#### Step 2
1. Iterate over all deployment schedules
  1. :gear: Flush all machines that were assigned to the current deployment schedule
  2. :arrows_counterclockwise: Iterate over all our connected machines
    * :white_check_mark: Check if the machine tag and the deployment schedule match, otherwise continue
    * :gear: Assign the machine to the deployment schedule
  3. :gear: Update the deployment schedule

{% highlight python %}
### 2. ITERATE OVER ALL DEPLOYMENT SCHEDULES AND REASSIGN MACHINES ###
# When using the Azure portal, deployment schedules have an associated schedule that defines when the patch will run.
# However, when using the Azure API, deployment schedules are actually made of :
#   - UpdateConfiguration objets, that defines which OS is targeted, which packages, etc.
#   - Schedule objets, that defines the reccurence, the start time, etc.
# The weird thing is that we can't retreive the Schedule from the UpdateConfiguration object.
# Thus, to resolve this, we get the list of UpdateConfiguration, we also get the list of Schedules, and we will make them match later.
update_configurations = automation_client.software_update_configurations.list(resource_group_name,automation_account_name).value
schedule_configurations = automation_client.schedule.list_by_automation_account(resource_group_name,automation_account_name)

# In this long loop, we iterate over each UpdateConfiguration. 
for update_configuration in update_configurations:
    # Local vars
    schedule = None

    # In this block of code, we clear the list of VMs that are currently assigned to the deployment schedule.
    # We need to do this because, if a VM tag is changed, the machine would appear in two deployments schedules.
    software_update_configuration_p = automation_client.software_update_configurations.get_by_name(resource_group_name, automation_account_name, update_configuration.name)
    software_update_configuration_p.update_configuration.non_azure_computer_names.clear()

    # Here, we check the OS for which the deployment schedule is configured.
    update_configuration_os = update_configuration.update_configuration.additional_properties.get('operatingSystem')

    # In this block of code, we decide to ignore two types of deployment schedules :
    #   - MANUAL deployment schedules - Those deployment schedule are created to patch on-demand, they are part of another process
    #   - CENTOS deployment schedules - Those deployment schedule are created to patch CENTOS, they are part of another process
    if "MANUAL" in update_configuration.name or "CENTOS" in update_configuration.name:
        continue
    
    
    # Remember line 9 when we retrieved the list of Schedules? We use this list to find the schedule associated to the current 
    # deployment schedule, based on name matching. We need to get this schedule to update it with a new start time. If we don't 
    # do this, the deployment schedule will be updated with a schedule that has a start_time in the past, and we will get an error.
    # Thus, we simply need to recalculate the start_time.
    for schedule_configuration in schedule_configurations:
        # If update configuration and schedules match, then we found the right schedule
        if update_configuration.name in schedule_configuration.name:
            schedule = automation_client.schedule.get(resource_group_name, automation_account_name, schedule_configuration.name)


    # In this block of code, we iterate over our connected machine to check if we should assign them to the current
    # deployment schedule by checking OS and tags. If the VM matches our criteria, it is added to the list of 
    # machines that will be assigned to the deployment schedule.
    for connected_machine in connected_machines:
        # Get connected machine attributes
        machine_rg = connected_machine['ResourceGroup']
        machine_name = connected_machine['Computer']
        machine_resource = connected_machine['Resource']
        machine_os = connected_machine['OSType']
        machine_tag = connected_machine['Tag']

        if machine_tag:
            if (machine_os in update_configuration_os) and (machine_tag.replace(':','-') in update_configuration.name):
                if machine_name not in software_update_configuration_p.update_configuration.non_azure_computer_names:
                    # Add machine to the list
                    software_update_configuration_p.update_configuration.non_azure_computer_names.append(machine_name)

    # In this next block of code, we calculate the new start time for the schedule.
	# IMPORTANT : because the runbook runs between 15:00 and 18:00, we must schedule it as follows :
	#	   - For 03:00 schedules => Schedule it on 'day+1', since 03:00 of the current day is in the past
	#	   - For 12:00 schedules => Schedule it on 'day+1', since 03:00 of the current day is in the past
	#	   - For 22:00 schedules => Schedule it on 'day', since 22:00 of the current day is in a few hours
	hours = software_update_configuration_p.schedule_info.start_time
	date = datetime(datetime.now().year, datetime.now().month, datetime.now().day)
	if "22-00" in update_configuration.name:
		day = date.day
	else:
		day = date.day+1
	software_update_configuration_p.schedule_info.start_time = datetime(date.year, date.month, day, hours.hour, hours.minute, 0)
	
    # When creating a deployment schedule, VMs must be assigned to it, otherwise we will get an error at deployment time.
    # Here are the different options to assign machines to a deployment schedule :
    #   - Assign machines statically : this is what our script do
    #   - Assign machines dynamically using a query
    # Thus, when we have no machine assigned to a deployment schedule, we need to create a create and assign it a query.
    # The query does not return any VM, it only allows us to deploy our delpoyment schedule without any error.
    non_azure_query = NonAzureQueryProperties()
    non_azure_query.function_alias = saved_searche_id
    non_azure_query.workspace_id = workspace_resource_id
    if not software_update_configuration_p.update_configuration.targets:
        print("Error processing this maintenance schdule : " + update_configuration.name)
        continue
    software_update_configuration_p.update_configuration.targets.non_azure_queries = [non_azure_query]

    # Finally we update our deployment schedule, and we handle an error that Azure may raise which is related to the number of request we send.
    # If we receive the error, the script will sleep for a random amount of time between 30 and 90 seconds.
    while 1:
        try:
            automation_client.software_update_configurations.create(resource_group_name, automation_account_name, update_configuration.name, software_update_configuration_p)
            break
        except HttpResponseError:
            print ("Exception HttpResponseError (Erreur 429 : too much API calls)")
            s = random.randint(30, 90)
            time.sleep(s)
{% endhighlight %}