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
Hello there, the goal of this serie is to describe a real world implementation of Azure Update Management. This is a service designed to update any machine in your infrastructure, whether they are hosted on Azure or elsewhere, provided your OSs are technically supported. I'll try to give a comprehensive feedback and drop advices here and there. Have a nice reading.

#### Plan
- [Part 0 - Introduction](/blog/2021/Update-management-00/)
- [Part 1 - Architecture](/blog/2021/Update-management-01/)
- [Part 2 - Azure Policy](/blog/2021/Update-management-011/)
- [Part 3 - Azure ARC](/blog/2021/Update-management-02/)
- [Part 4 - Log Analytics agents (you're here)](/blog/2021/Update-management-03/)
- [Part 5 - Automation accounts](/blog/2021/Update-management-04/)
- [Part 6 - Monitoring](/blog/2021/Update-management-05/)
- <b>[Part 7 - Security patches on CentOS machines](/blog/2021/Update-management-06/)</b>

***

## Context
If you have CentOS machines, you probably faced issues when trying to apply <b>security</b> updates them : a ```yum update``` may crash your applications because it does not apply security-only updates, while ```yum update --security``` won't update any package because it relies on packages metadata, which are not set for CentOS packages. Alternatively, you could allow only security repositories, but this implies additional maintenance when installing non-security related stuff on your machines.

When using Azure Automation Update solution, the issue remains. if you take a look at the [Microsoft documentation](https://learn.microsoft.com/en-us/azure/automation/update-management/overview#logic-for-linux-updates-classification), you'll see that <i>Update Management classifies updates into three categories: <b>Security</b>, <b>Critical</b> or <b>Others</b></i>. This is fine, now you will also read that <i>Unlike other distributions, CentOS does not have classification data available from the package manager. If you have CentOS machines configured to return security data for the following command, Update Management can patch based on classifications.</i> The command to test is shown below. This is a real problem because in order for this command, you need to add metadata to package yourself, or usually pay for a service that does it.
{% highlight bash %}
sudo yum -q --security check-update
{% endhighlight %}

When observing what I just described for the first time, I couldn't believe it! There are not native nor simple solution to setup security updates on CentOS.
<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/centos_1.gif" class="img-fluid rounded z-depth-1" style="float: center" %}
</div>

***

## Solution
Of course, the reason I wrote this post is that I spent some time working on it, and I have a solution! To make it short, the solution is to periodically run a smart script, once a week, that you can deploy in an Azure Runbook. This script will create deployement schedules for each CentOS machine that needs security updates or critical updates. Simple right?

To be more specific, I actually wrote two very similar scripts : one for Azure VMs, the other for Azure ARC VMs. Why? Because writing a single script didn't match my architecture, but you could merge them in a single script.

In any case, the script behavior remains the same ans checks that all VMs match the two following criteria.
- The machine must be a CentOS machine;
- The machine must be tagged with a valid <b>patch</b> tag.
If a VM doesn't comply, it won't be patched.

An additional word about tags : in my case, we defined a tag policy in order for machines to be updated each week on a specific schedule. Here is the tag pattern : <b>^CENTOS-[PQR]-(MON|TUE|WED|THU|FRI|SAT|SUN)-(03|12|22):00$</b>.
- ```[PQR]``` is for the environment : P for Production, Q for Qualif, R for Recette. Basically, if we run the script in a production environment, machines containing the <b>P-</b> prefix will be updated, the others being ignored.
- ```(MON|TUE|WED|THU|FRI|SAT|SUN)``` is for the day of the week when the machine must be updated.
- ```(03|12|22):00``` is for the hour when to update the machine.


<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/centos_2.png" class="img-fluid rounded z-depth-1" %}
</div>

***

### Solution for Azure VMs
#### Step 1 - Iterate over all machines

First of all, we need to import a lot of Azure dependencies, because our script will use the Azure Python SDK to manage our machines. If you decide to run this code in a runbook, take a look at the [Microsoft documentation](https://learn.microsoft.com/en-us/azure/automation/python-3-packages?tabs=py3) to upload Python libraries Additionally, you will find documentation references if you need additional information.

{% highlight python %}
#!/usr/bin/env python3
################################################################################################################################
# MS API Docs - Automation account
# https://docs.microsoft.com/en-us/python/api/azure-mgmt-automation/azure.mgmt.automation.automationclient
# https://docs.microsoft.com/en-us/python/api/azure-mgmt-automation/azure.mgmt.automation.operations.softwareupdateconfigurationsoperations
# https://docs.microsoft.com/en-us/python/api/azure-mgmt-automation/azure.mgmt.automation.models.softwareupdateconfiguration
# https://docs.microsoft.com/en-us/python/api/azure-mgmt-automation/azure.mgmt.automation.models.scheduleproperties
# https://docs.microsoft.com/en-us/python/api/azure-mgmt-automation/azure.mgmt.automation.models.updateconfiguration

# MS API Docs - Log analytics
# https://docs.microsoft.com/en-us/python/api/overview/azure/monitor-query-readme?view=azure-python
# https://jihanjeeth.medium.com/log-retrieval-from-azure-log-analytics-using-python-52e8e8e5e870
################################################################################################################################

# Azure packages
from azure.identity import DefaultAzureCredential
from azure.mgmt.automation import AutomationClient
from azure.mgmt.automation.models import NonAzureQueryProperties, ScheduleCreateOrUpdateParameters, ScheduleFrequency, SoftwareUpdateConfiguration, UpdateConfiguration, ScheduleProperties, LinuxProperties, LinuxUpdateClasses, OperatingSystemType
from azure.mgmt.loganalytics import LogAnalyticsManagementClient
from azure.mgmt.compute import ComputeManagementClient
from azure.loganalytics import LogAnalyticsDataClient
from azure.loganalytics.models import QueryBody
from azure.monitor.query import LogsQueryClient, LogsQueryStatus
from datetime import datetime, timezone, timedelta
import automationassets
from automationassets import AutomationAssetNotFound
from azure.mgmt.subscription import SubscriptionClient

# Various pacakges
import pandas as pd

# System packages
import sys
import json
import re
import pytz
{% endhighlight %}

In this next piece of code, we load variables values from the Automation Account variable. In order to set these variables in automation accounts, check the [Microsoft documentation](https://learn.microsoft.com/en-us/azure/automation/shared-resources/variables?tabs=azure-powershell). Here, variables are the current automation account name, its resource group and its subscription.
{% highlight python %}

#####################################################################################
# 0. GET AUTOMATION ACCOUNT VARIABLES
automation_account_name = automationassets.get_automation_variable("aa_name")
resource_group_name = automationassets.get_automation_variable("resource_group_name")
subscription_id = automationassets.get_automation_variable("subscription_id")
{% endhighlight %}

Here, we create credentials using the DefaultAzureCredential() function, which will use the automation account managed identity to make the APIs calls. We will see later which permissions should be configured on the automation account. Then, we instanciate different clients to communicate with the Azure API.
{% highlight python %}
# Instanciate all clients
token_credential = DefaultAzureCredential()
automation_client = AutomationClient(token_credential, subscription_id)
log_analytics_client = LogAnalyticsManagementClient(token_credential, subscription_id)
log_analytics_data_client = LogsQueryClient(token_credential)
subscriptions_client = SubscriptionClient(token_credential)
{% endhighlight %}

This next piece of code shows two variables that will be used later in the code. <b>DAYS</b> will be used to find the next available schedule to update the machine. The <b>regex</b> will be used to evaluate machines tags: as seen earlier, the pattern differs, based on the production or non production environment.
{% highlight python %}
DAYS = {
	"MON":0,
	"TUE":1,
	"WED":2,
	"THU":3,
	"FRI":4,
	"SAT":5,
	"SUN":6
}
# Match only non prod machines
regex = re.compile("^CENTOS-[RQ]-(MON|TUE|WED|THU|FRI|SAT|SUN)-(03|12|22):00$")
# Match only prod machines
# regex = re.compile("^CENTOS-[P]-(MON|TUE|WED|THU|FRI|SAT|SUN)-(03|12|22):00$")
{% endhighlight %}

Ok, here begins the interesting part : we iterate over all susbcriptions, and for each subscription, we iterate over all Azure VMs (there is a small variation for Azure ARC VMs, I'll discuss this later). In this double loop, we ensure the <b>patch</b> tag is compliant with what we defined, and we also ensure that the VM is a CentOS VM. <u>Note</u> : when checking the OS, we need to check both custom OS and Azure-provided OS.

{% highlight python %}
###############################################################
# 1. GET AND SANITIZE ALL CENTOS VMs THAT NEED TO BE PATCHED
# 	 First, we check that the 'patch' tag is compliant
# 	 Then, We check that the OS is indeed CentOS

centOSs = []
for subscription in subscriptions_client.subscriptions.list():
	compute_management_client = ComputeManagementClient(token_credential, subscription.subscription_id)
	#print("Successfully instanciated using " + str(subscription) + "subscription")
	for item in compute_management_client.virtual_machines.list_all():
		# Check PATCH value
		if not item.tags or not 'patch' in item.tags.keys():
			continue
		if not regex.search(item.tags['patch']):
			continue

		# Check OS value
		if item.storage_profile.image_reference:
			# Check when OS is custom
			if item.storage_profile.image_reference.id and "CentOS" in item.storage_profile.image_reference.id:
				centOSs.append(item)
			# Check when OS is Azure based
			elif item.storage_profile.image_reference.offer and "CentOS" in item.storage_profile.image_reference.offer:
				centOSs.append(item)
			else:
				continue
		else:
			continue
{% endhighlight %}

We now have a list of machines to update, and we need to get all available updates for each machine. For this, we send a KQL query to the Log Analytics workspaces that collects updates information : we store this in the <b>df</b> variable.

{% highlight python %}
###########################################################################
# 2. RETIEVE ALL SECURITY AND CRITICAL UPDATES NEEDED BY ALL LINUX MACHINES
# 	 All machines that are not in this table :
# 		a. Do not report to the Log Analytics => To troubleshoot
# 		b. Or do not need update => Fine
query = '''Update
		| where TimeGenerated>ago(5h) and OSType=="Linux"
		| summarize hint.strategy=partitioned arg_max(TimeGenerated, UpdateState, Classification, BulletinUrl, BulletinID) by ResourceId, Computer, SourceComputerId, Product, ProductArch
		| where UpdateState=~"Needed"
		| project-away UpdateState, TimeGenerated
		| summarize computersCount=dcount(SourceComputerId, 2), ClassificationWeight=max(iff(Classification has "Critical", 4, iff(Classification has "Security", 2, 1))) by ResourceId, Computer, id=strcat(Product, "_", ProductArch), displayName=Product, productArch=ProductArch, classification=Classification, InformationId=BulletinID, InformationUrl=tostring(split(BulletinUrl, ";", 0)[0]), osType=1
		| sort by ClassificationWeight desc, computersCount desc, displayName asc
		| extend informationLink=(iff(isnotempty(InformationId) and isnotempty(InformationUrl), toobject(strcat('{ "uri": "', InformationUrl, '", "text": "', InformationId, '", "target": "blank" }')), toobject('')))
		| project-away ClassificationWeight, InformationId, InformationUrl
		| where classification has "Security" or classification has "Critical"'''
workspace_name 			= automation_client.linked_workspace.get(resource_group_name, automation_account_name).id.split('/')[-1]
workspace_id 			= log_analytics_client.workspaces.get(resource_group_name, workspace_name).customer_id
workspace 				= automation_client.linked_workspace.get(resource_group_name, automation_account_name)

# SEND REQUEST
end_time	= datetime.now(pytz.timezone("Europe/Paris"))
start_time 	= end_time - timedelta(days=1)
response 	= log_analytics_data_client.query_workspace(workspace_id, query, timespan=(start_time, end_time))

if response.status == LogsQueryStatus.PARTIAL:
	error = response.partial_error
	data = response.partial_data
	print("ERROR, unknown error when requesting Log Analytics")
elif response.status == LogsQueryStatus.SUCCESS:
	data = response.tables
df = pd.DataFrame(data=data[0].rows, columns=data[0].columns)
{% endhighlight %}


In this last part of the script, we iterate over our machines to update, and we check in our df variable if the current machine needs updates :
- if it doesn't, then we ignore it and directly go to the next loop iteration.
- if it does, we calculate the next schedule when the machine should be updated, based on its <b>patch</b> tag, and then we instanciate the necessary objects that will create the deployment schedules in our automation account.
{% highlight python %}
###########################################
# 3. ITERATE OVER OUR CENTOS :
#	Step 1 - Get needed updates
#	Step 2 - Calculate next schedule
#	Step 3 - Create the deployment schedule
for centOS in centOSs:
	# Step 1 - Get needed updates
	updates = []
	# The following line filters dataFrame to return all lines for which
	# the column ResourceId contains centOS.id.
	# Note : we convert all strings to lower() to avoid any issue
	updates_df = df[df['ResourceId'].str.lower().str.contains(centOS.id.lower())]
	for index, row in updates_df.iterrows():
		updates.append(row[3])
	if len(updates) == 0:
		print("This " + centOS.id + " do not need to be patched")
		continue
	
	# Step 2 - Calculate next schedule
	# Get info from tag, and get current date
	target_day = DAYS[centOS.tags['patch'].split('-')[2]]
	target_hour= int(centOS.tags['patch'].split('-')[3].split(':')[0])
	now = datetime.now()

	# Find how many days until the next update
	# E.g. if the script runs SUNDAY, and we need to patch WEDNESDAY :
	#	SUNDAY - WEDNESDAY = 6 - 3 = 3 days
	# So next occurence will be the current date + 3 days.
	days_from_now = (target_day - now.weekday()) % 7
	target_date = now + timedelta(days=days_from_now)

	# Modify our target date with appropriate hours, minutes and seconds
	target_date = datetime(target_date.year, target_date.month, target_date.day, target_hour, 0, 0)
	# This conditions handles the case where days_from_now = 0
	if target_date < now:
		target_date = target_date + timedelta(days=7)

	# Step 3 - Create the deployment schedule
	schedule_info = ScheduleProperties(
		start_time 	= target_date,
		time_zone 	= "Europe/Paris",
		is_enabled 	= True,
		frequency 	= ScheduleFrequency.ONE_TIME)

	linux_properties = LinuxProperties(
		included_package_classifications	= None,
		included_package_name_masks 		= updates,
		reboot_setting 						= "Always")

	update_configuration = UpdateConfiguration(
		operating_system 		= OperatingSystemType.LINUX,
		linux 					= linux_properties,
		duration 				= timedelta(hours=2),
		azure_virtual_machines 	= [centOS.id])
	
	software_update_configuration = SoftwareUpdateConfiguration(
		update_configuration 	= update_configuration,
		schedule_info 			= schedule_info)
	
	
	#software_update_configuration_name = "ari-iaasupdate-we-securityux-CENTOS-" + centOS.name
	software_update_configuration_name = "ari-iaasupdate-we-securityux-" + centOS.tags['patch'] + " => " + centOS.name
	automation_client.software_update_configurations.create(resource_group_name, automation_account_name, software_update_configuration_name, software_update_configuration)
	print("VERBOSE, new deployment schedule created for CentOS VM : " + centOS.id)
{% endhighlight %}

## Permissions
In order for this script to work, the identity you use must have multiple permissions :
- <b>Virtual Machines contributor</b> role on each subscription or resource group where you have CentOS VMs to update
- <b>Log Analytics reader</b> role on each subscription or resource group where you have CentOS VMs to update
- <b>Automation contributor</b> on the Automation Account you use for patch management.

If you run the script on Azure, you must assign these permissions using <b>system-assigned managed identities</b>, c.f. [Microsoft documentation](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal-managed-identity).

## Run the script
Once you ran the script, it will deploy one deployment schedule per VM, as shown below. If you click on the deployment schedule, you will see that in the <b>Include/exclude updates</b>, you will see the explicit list of all updates to be installed on the VM. When configuring a deployment schedule like this, Azure no longer use the <b>yum install --security</b> command, but it uses the <b>yum install <package1> <package2> <packagen></b> command!
<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/centos_2.png" class="img-fluid rounded z-depth-1" %}
</div>

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/centos_3.png" class="img-fluid rounded z-depth-1" %}
</div>

What I didn't tell you is that those deployment schedules are run only once, because they contains specific packages to update that may not be the same on next week. Once the deployment schedule is executed, you will see that <i>Next run time</i> is set to None. You have two options :
- You want your deployment schedule to be clean, and remove those unused deployment schedules;
- You don't care because the next time the same VM will be overwriten with a new set of needed updates.

If you rather want to clean it, here is a small script that you can run as a Runbook in the same automation account.
{% highlight python %}
#!/usr/bin/env python3
# Azure packages
from azure.identity import DefaultAzureCredential
from azure.mgmt.automation import AutomationClient
from azure.mgmt.automation.models import NonAzureQueryProperties, ScheduleCreateOrUpdateParameters, ScheduleFrequency, SoftwareUpdateConfiguration, UpdateConfiguration, ScheduleProperties, LinuxProperties, LinuxUpdateClasses, OperatingSystemType
from azure.mgmt.loganalytics import LogAnalyticsManagementClient
from azure.loganalytics import LogAnalyticsDataClient
from azure.loganalytics.models import QueryBody
from azure.monitor.query import LogsQueryClient, LogsQueryStatus
from datetime import datetime, timezone, timedelta
import automationassets
from automationassets import AutomationAssetNotFound

# System packages
import sys 
import json


##################################################################################################################
# 0. INITIALIZE API ENDPOINTS
automation_account_name = automationassets.get_automation_variable("aa_name")
resource_group_name = automationassets.get_automation_variable("resource_group_name")
subscription_id = automationassets.get_automation_variable("subscription_id")

# Instanciate all clients
token_credential = DefaultAzureCredential()
automation_client = AutomationClient(token_credential, subscription_id)
log_analytics_client = LogAnalyticsManagementClient(token_credential, subscription_id)
log_analytics_data_client = LogsQueryClient(token_credential)

deployment_schedules = automation_client.software_update_configurations.list(resource_group_name, automation_account_name)
for deployment_schedule in deployment_schedules.value:
	if deployment_schedule.next_run == None and "CENTOS" in deployment_schedule.name:
		automation_client.software_update_configurations.delete(resource_group_name, automation_account_name, deployment_schedule.name)
		print("Deleted : " + deployment_schedule.name)

{% endhighlight %}

***

### Solution for Azure VMs
The script is almost the same because VM type is <i>Microsoft.HybridCompute/machines</i> rather than <i>Microsoft.Compute/virtualMachines</i>. Similarly, permissions are not the same : the <b>Virtual Machines contributor</b> must be replaced with <b>Azure Connected Machine Resource Administrator</b>.