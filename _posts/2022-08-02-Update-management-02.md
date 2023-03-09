---
layout: post
title: Azure Update Management - Part 3 - Deploy Azure ARC
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
- [Part 1 - Architecture](/blog/2022/Update-management-01/)
- [Part 2 - Azure Policy](/blog/2022/Update-management-011/)
- <b>[Part 3 - Azure ARC (you're here)](/blog/2022/Update-management-02/)</b>
- [Part 4 - Log Analytics agents](/blog/2022/Update-management-03/)
- [Part 5 - Automation accounts](/blog/2023/Update-management-04/)
- [Part 6 - Monitoring](/blog/2023/Update-management-05/)
- [Part 7 - Security patches on Azure ARC](/blog/2023/Update-management-06/)
- [Part 8 - Security patches on CentOS machines](/blog/2023/Update-management-07/)

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
According to [Microsoft documentation](https://learn.microsoft.com/en-us/azure/azure-arc/servers/network-requirements?tabs=azure-cloud), there are several network flows to open. To achieve this, we can identify three cases :
1. Your network devices are on Azure - In that case, you can leverage service tags in NSGs to open flows easily.
2. If your network devices support domain-based filtering, you can configure them directly.
3. Finally, if your network devices do not match the previous criteria, this means that you need to open flows based on IPs. In order to achieve this, you can [download](https://www.microsoft.com/download/details.aspx?id=56519) the list of all IPs associated service tags. Here is a script that defines a list of needed service tags, that parses the Microsoft JSON file, and gives you a CSV with all IPs to whitelist. You can add automation to run this script every two weekd and interact with your network devices to update them directly with the appropriate IPs.

{% highlight python %}
import json
import re

def is_ipv4(ip):
    regex = re.compile(r"^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])$")
    return regex.search(ip)

tags = [
    'GuestAndHybridManagement',
    # Storage
    'Storage.WestEurope',
    'Storage.FranceCentral',
    'Storage.FranceSouth',
    'Storage.UKSouth',
    'Storage.SouthCentralUS',

    # AzureMonitor 
    'AzureMonitor.WestEurope',
    'AzureMonitor.FranceCentral',
    'AzureMonitor.FranceSouth',
    'AzureMonitor.UKSouth',

    # ServiceBus
    'ServiceBus.WestEurope',
    'ServiceBus.FranceCentral',
    'ServiceBus.FranceSouth',
    'ServiceBus.UKSouth',

    # AzureResourceManager
    'AzureResourceManager',

    # AzureActiveDirectory
    'AzureActiveDirectory',

    # AzureArcInfrastructure
    'AzureArcInfrastructure.WestEurope',
    'AzureArcInfrastructure.FranceCentral',
    'AzureArcInfrastructure.FranceSouth',
    'AzureArcInfrastructure.UKSouth',

    # AzureFrontDoor.FirstParty
    'AzureFrontDoor.FirstParty'
]


f = open('ServiceTags_Public_20220808.json')
csv = open('updtmgmt_ips.csv', "w")
csv.write('tag,ip\n')
data = json.loads(f.read())


for tag in data['values']:
    if tag['name'] in tags:
        for ip in tag['properties']['addressPrefixes']:
            # Remove / if it is range
            ip_to_test = ip.split('/')[0]
            if is_ipv4(ip_to_test):
                csv.write(tag['name'] + ',' + ip + "\n")
f.close()
csv.close()
{% endhighlight %}




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

{% highlight bash %}
az login
az account set --subscription "{Your Subscription Name}"
az provider register --namespace 'Microsoft.HybridCompute'
az provider register --namespace 'Microsoft.GuestConfiguration'
az provider register --namespace 'Microsoft.HybridConnectivity'
{% endhighlight %}

## Deployment
### Create a service principal
In order for the VMs to be onboarded, we need to use a service principal that will authenticate using a secret. This service principal need the <b>Azure Connected Machine Contributor</b> role on every subscription where you want to deploy Azure ARC machines. Here is the [Microsoft documentation](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal) that describes how to create a service principal from the portal.

### Set up deployment script
When deploying from the portal, Microsoft provides a script to run on your servers. You need to specify a few arguments for this script :
- <b>Service principal client ID and secret</b> - From the service principal previously created
- <b>TenantId</b> - You can retrieve it from Azure AD
- <b>SubscriptionId</b> - This is the subscription where you will deploy Azure ARC machines.
- <b>ResourceGroup</b> - This is the resource group where you will deploy Azure ARC machines.
- <b>Location</b> - This is the Azure region (e.g. westeu) where the Azure ARC machines will be deployed.
- <b>Tags</b> - If you want to assign tags to your machines when deploying, use the <b>--tags</b> option and pass it key=value arguments e.g. "Datacenter='02/21/2022 15:21:12'".

### Run the scripts
In order to run the scripts provided by Azure, you need to open flow to the following URLs. Unfortunately, the <i>aka.ms</i> URL are not predictable since the response you receive is an HTTP 302 Redirect that may change in the future.
- https://aka.ms/azcmagent-windows, which currently redirects to https://gbl.his.arc.azure.com
- https://aka.ms/AzureConnectedMachineAgent, which currently redirects to https://download.microsoft.com
- https://packages.microsoft.com/*
For this reason, I'll describe the procedure both with and without Internet access on the server, so that you can install it offiline, as I had to do.


#### Deploy with Internet access
Depending whether the server is running Windows or Linux, run the following scripts.
<u>Script for Windows</u>
{% highlight powershell %}
try {
    # Add the service principal application ID and secret here
    $servicePrincipalClientId="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";
    $servicePrincipalSecret="supersecret";

    $env:SUBSCRIPTION_ID = "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy";
    $env:RESOURCE_GROUP = "<YOUR RESOURCE GROUP NAME>";
    $env:TENANT_ID = "zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz";
    $env:LOCATION = "<YOUR REGION>";
    $env:AUTH_TYPE = "principal";
    $env:CORRELATION_ID = "6b777709-0403-4e37-b05c-da346249ffaf";
    $env:CLOUD = "AzureCloud";

    [Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor 3072;

    # Download the installation package
    Invoke-WebRequest -UseBasicParsing -Uri "https://aka.ms/azcmagent-windows" -TimeoutSec 30 -OutFile "$env:TEMP\install_windows_azcmagent.ps1";

    # Install the hybrid agent
    & "$env:TEMP\install_windows_azcmagent.ps1";
    if ($LASTEXITCODE -ne 0) { exit 1; }

    # Run connect command
    & "$env:ProgramW6432\AzureConnectedMachineAgent\azcmagent.exe" connect --service-principal-id "$servicePrincipalClientId" --service-principal-secret "$servicePrincipalSecret" --resource-group "$env:RESOURCE_GROUP" --tenant-id "$env:TENANT_ID" --location "$env:LOCATION" --subscription-id "$env:SUBSCRIPTION_ID" --cloud "$env:CLOUD" --correlation-id "$env:CORRELATION_ID" --tags "Datacenter='02/21/2022 15:21:12'";
}
catch {
    $logBody = @{subscriptionId="$env:SUBSCRIPTION_ID";resourceGroup="$env:RESOURCE_GROUP";tenantId="$env:TENANT_ID";location="$env:LOCATION";correlationId="$env:CORRELATION_ID";authType="$env:AUTH_TYPE";operation="onboarding";messageType=$_.FullyQualifiedErrorId;message="$_";};
    Invoke-WebRequest -UseBasicParsing -Uri "https://gbl.his.arc.azure.com/log" -Method "PUT" -Body ($logBody | ConvertTo-Json) | out-null;
    Write-Host  -ForegroundColor red $_.Exception;
}
{% endhighlight %}

<u>Script for Linux</u>

{% highlight bash %}
# Add the service principal application ID and secret here
servicePrincipalClientId="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";
servicePrincipalSecret="supersecret";


export subscriptionId="yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy";
export resourceGroup="<YOUR RESOURCE GROUP NAME>";
export tenantId="zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz";
export location="<YOUR REGION>";
export authType="principal";
export correlationId="6b777709-0403-4e37-b05c-da346249ffaf";
export cloud="AzureCloud";

# Download the installation package
output=$(wget https://aka.ms/azcmagent -O ~/install_linux_azcmagent.sh 2>&1);
if [ $? != 0 ]; then wget -qO- --method=PUT --body-data="{\"subscriptionId\":\"$subscriptionId\",\"resourceGroup\":\"$resourceGroup\",\"tenantId\":\"$tenantId\",\"location\":\"$location\",\"correlationId\":\"$correlationId\",\"authType\":\"$authType\",\"operation\":\"onboarding\",\"messageType\":\"DownloadScriptFailed\",\"message\":\"$output\"}" "https://gbl.his.arc.azure.com/log" &> /dev/null || true; fi;
echo "$output";

# Install the hybrid agent
bash ~/install_linux_azcmagent.sh;

# Run connect command
sudo azcmagent connect --service-principal-id "$servicePrincipalClientId" --service-principal-secret "$servicePrincipalSecret" --resource-group "$resourceGroup" --tenant-id "$tenantId" --location "$location" --subscription-id "$subscriptionId" --cloud "$cloud" --tags "Datacenter='02/21/2022 15:21:12'" --correlation-id "$correlationId";
{% endhighlight %}


#### Deploy without Internet access
##### Install azcmagent
The first step for Windows machines is to [download the AzureConnectedMachineAgent MSI file](https://aka.ms/AzureConnectedMachineAgent) from a machine that has Internet. Then you can push it to your machines and install it using the Microsoft script I adapted for offline install, [available on my Github](https://github.com/Molx32/AzureUpdateManagement/blob/main/install_windows_azcmagent.ps1), that you must run in the directory where the MSI file is located.

The first step for Linux machines is to download the AzureConnectedMachineAgent .deb or .rpm files based on your OS. You can retrieve them at https://packages.microsoft.com. Here are some examples :
- For CentOS : https://packages.microsoft.com/centos/8/prod/Packages/a/azcmagent-1.9.21208-007.x86_64.rpm
- For Debian : https://packages.microsoft.com/debian/10/prod/pool/main/a/azcmagent/azcmagent_1.26.02210.615_amd64.deb
- For Ubuntu : https://packages.microsoft.com/ubuntu/20.04/prod/pool/main/a/azcmagent/azcmagent_1.26.02210.615_amd64.deb
Then you can push it to your machines and install it using the Microsoft script I adapted for offline install, [available on my Github](https://github.com/Molx32/AzureUpdateManagement/blob/main/install_linux_azcmagent.sh), that you must run in the directory where the MSI file is located. <u>Note</u>: you can tweak the script to change the version number of azcmagent :
{% highlight bash %}
# Install the azcmagent
if [ $apt -eq 1 ]; then
	sudo -E apt install -y ./packages/azcmagent_1.17.01931.118_amd64.deb
elif [ $zypper -eq 1 ]; then
	sudo -E zypper install -y ./packages/azcmagent-1.17.01931-135.x86_64.rpm
else
    sudo -E yum -y localinstall ./packages/azcmagent-1.17.01931-135.x86_64.rpm
fi
{% endhighlight %}

##### Deploy Azure ARC
Once the machine is deployed, all you need to do is to run the azcmagent command as shown below.
<u>For Windows machines</u>
{% highlight powershell %}
try {
    # Add the service principal application ID and secret here
    $servicePrincipalClientId="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";
    $servicePrincipalSecret="supersecret";

    $env:SUBSCRIPTION_ID = "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy";
    $env:RESOURCE_GROUP = "<YOUR RESOURCE GROUP NAME>";
    $env:TENANT_ID = "zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz";
    $env:LOCATION = "<YOUR REGION>";
    $env:AUTH_TYPE = "principal";
    $env:CORRELATION_ID = "6b777709-0403-4e37-b05c-da346249ffaf";
    $env:CLOUD = "AzureCloud";

    [Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor 3072;

    # Run connect command
    & "$env:ProgramW6432\AzureConnectedMachineAgent\azcmagent.exe" connect --service-principal-id "$servicePrincipalClientId" --service-principal-secret "$servicePrincipalSecret" --resource-group "$env:RESOURCE_GROUP" --tenant-id "$env:TENANT_ID" --location "$env:LOCATION" --subscription-id "$env:SUBSCRIPTION_ID" --cloud "$env:CLOUD" --correlation-id "$env:CORRELATION_ID" --tags "Datacenter='02/21/2022 15:21:12'";
}
catch {
    $logBody = @{subscriptionId="$env:SUBSCRIPTION_ID";resourceGroup="$env:RESOURCE_GROUP";tenantId="$env:TENANT_ID";location="$env:LOCATION";correlationId="$env:CORRELATION_ID";authType="$env:AUTH_TYPE";operation="onboarding";messageType=$_.FullyQualifiedErrorId;message="$_";};
    Invoke-WebRequest -UseBasicParsing -Uri "https://gbl.his.arc.azure.com/log" -Method "PUT" -Body ($logBody | ConvertTo-Json) | out-null;
    Write-Host  -ForegroundColor red $_.Exception;
}
{% endhighlight %}

<u>For Linux machines</u>
{% highlight bash %}
# Add the service principal application ID and secret here
servicePrincipalClientId="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";
servicePrincipalSecret="supersecret";


export subscriptionId="yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy";
export resourceGroup="<YOUR RESOURCE GROUP NAME>";
export tenantId="zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz";
export location="<YOUR REGION>";
export authType="principal";
export correlationId="6b777709-0403-4e37-b05c-da346249ffaf";
export cloud="AzureCloud";

# Run connect command
sudo azcmagent connect --service-principal-id "$servicePrincipalClientId" --service-principal-secret "$servicePrincipalSecret" --resource-group "$resourceGroup" --tenant-id "$tenantId" --location "$location" --subscription-id "$subscriptionId" --cloud "$cloud" --tags "Datacenter='02/21/2022 15:21:12'" --correlation-id "$correlationId";
{% endhighlight %}

## Validation
You can ensure Azure ARC is deployed by taking a look at the Azure portal.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/azure_arc_01.png" class="img-fluid rounded z-depth-1" %}
</div>