---
layout: post
title: Azure Update Management - Part 2 - Azure Policy
date: 2022-08-02 10:00:00
description: This post describes how to implement Azure policies
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

***

#### Plan
- [Part 0 - Introduction](/blog/2022/Update-management-00/)
- [Part 1 - Architecture](/blog/2022/Update-management-01/)
- [Part 2 - Azure Policy (you're here)](/blog/2022/Update-management-011/)
- [Part 3 - Azure ARC](/blog/2022/Update-management-02/)
- [Part 4 - Log Analytics agents](/blog/2022/Update-management-03/)
- [Part 5 - Automation accounts](/blog/2022/Update-management-04/)
- [Part 6 - Monitoring](/blog/2022/Update-management-05/)
- [Part 7 - Security patches on Azure ARC](/blog/2022/Update-management-07/)
- [Part 8 - Security patches on CentOS machines](/blog/2022/Update-management-06/)

***

## What is Azure Policy
Azure Policy is a service that allows you to define rules to properly manage your resources. You can imagine a lot a controls to apply, here are a few examples :
- Enforce a specific set of tags on all resource groups
- Prevent users from deploying VM with an expensive SKU
- Prevent deployment of Storage Account with public access

Technically, these rules are called <i>policy definitions</i>. Once a policy definition is created, nothing happen : you need to assign the policy definition to a specific scope. For instance, if I want to enforce a specific set of tags on all resource groups, I may first assign this policy definition to my "Non production" subscription first, before assigning it to my "Production" subscription.

What is our goal?
We want to create a policy definition that <i>enforces all Azure VMs and all Azure ARC VMs to be onboarded on Azure Automation Update</i>. To be more specific and technical, what we want to is <i>enforce Log Analytics agents to be installed on all Azure VMs and all Azure ARC VMs to be</i>. Let's discuss this in the next section.


## Azure Policy deployment
Our goal is to have something that looks like the scheme below. To explain this, I'll repeat what I described earlier, but with more specific words. We create four policy assignments<i>policy assignments</i>:
  1. An assignment at the subscription level, that must apply to all Azure VMs, that enforces all Azure VMs to report to ari-loga-azure-001 Log Analytics.
  2. An assignment at the ari_rg_ovh_001 resource group, that must apply to all OVH Azure ARC VMs Azure VMs, that enforces all OVH Azure ARC VMs to report to ari-loga-ovh-001 Log Analytics.
  3. An assignment at the ari_rg_oci_001 resource group, that must apply to all OCI Azure ARC VMs Azure VMs, that enforces all OCI Azure ARC VMs to report to ari-loga-oci-001 Log Analytics.
  4. An assignment at the ari_rg_gcp_001 resource group, that must apply to all GCP Azure ARC VMs Azure VMs, that enforces all GCP Azure ARC VMs to report to ari-loga-gcp-001 Log Analytics.

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/arch_3.png" class="img-fluid rounded z-depth-1" %}
</div>

### Create the policy definitions
We actually need to create multiple policy definitions. Before sharing the policies to implement, let's take a look at the policy definition code. This first part shows two parameters :
- <b>policyRule</b> - It defines in which conditions the policy will be applied. In that case, the policy will be applied to resources with <i>Microsoft.HybridCompute/machines</i> type (i.e. Azure ARC resources), and to resources which run Linux.
{% highlight json %}
{
    "if": {
        "allOf": [
        {
            "field": "type",
            "equals": "Microsoft.HybridCompute/machines"
        },
        {
            "field": "Microsoft.HybridCompute/machines/osName",
            "equals": "linux"
        }
        ]
    },
{% endhighlight %}

This second part describes the effect to apply when resources match the conditions aforementioned.
- <b>effect</b> : this setting is parametrized. This means we can set the value when assigning the policy. The effect we will choose is DeployIfNotExists, which will deploy the agent on the machine when not detected.
- <b>existenceConditions</b> : this setting is used to say that if resources already have the agent installed, then we do nothing. So it is important to ensure that machines have no agent installed before deployment (you can check this in extensions).
{% highlight json %}
    "then": {
      "effect": "[parameters('effect')]",
      "details": {
        "type": "Microsoft.HybridCompute/machines/extensions",
        "roleDefinitionIds": [
          "/providers/Microsoft.Authorization/roleDefinitions/92aaf0da-9dab-42b6-94a3-d43ce8d16293"
        ],
        "existenceCondition": {
          "allOf": [
            {
              "field": "Microsoft.HybridCompute/machines/extensions/type",
              "equals": "OmsAgentForLinux"
            },
            {
              "field": "Microsoft.HybridCompute/machines/extensions/publisher",
              "equals": "Microsoft.EnterpriseCloud.Monitoring"
            },
            {
              "field": "Microsoft.HybridCompute/machines/extensions/provisioningState",
              "equals": "Succeeded"
            }
          ]
        },
{% endhighlight %}

Finally, this last part describes what will be deployed. To make it short, the agent is deployed as an Azure VM Extension, and will be automatically configured with the appropriate Log Analytics workspace thanks to parameters defined at deployment time.
{% highlight json %}
        "deployment": {
          "properties": {
            "mode": "incremental",
            "template": {
              "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
              "contentVersion": "1.0.0.0",
              "parameters": {
                "vmName": {
                  "type": "string"
                },
                "location": {
                  "type": "string"
                },
                "logAnalytics": {
                  "type": "string"
                }
              },
              "variables": {
                "vmExtensionName": "OMSAgentForLinux",
                "vmExtensionPublisher": "Microsoft.EnterpriseCloud.Monitoring",
                "vmExtensionType": "OmsAgentForLinux"
              },
              "resources": [
                {
                  "name": "[concat(parameters('vmName'), '/', variables('vmExtensionName'))]",
                  "type": "Microsoft.HybridCompute/machines/extensions",
                  "location": "[parameters('location')]",
                  "apiVersion": "2019-12-12",
                  "properties": {
                    "publisher": "[variables('vmExtensionPublisher')]",
                    "type": "[variables('vmExtensionType')]",
                    "settings": {
                      "workspaceId": "[reference(parameters('logAnalytics'), '2015-03-20').customerId]",
                      "stopOnMultipleConnections": "true"
                    },
                    "protectedSettings": {
                      "workspaceKey": "[listKeys(parameters('logAnalytics'), '2015-03-20').primarySharedKey]"
                    }
                  }
                }
              ],
            }
        }
    }
}
{% endhighlight %}

#### Policy definition for Azure ARC Windows VMs
You'll find the necessary files below :
- [policy_arc_windows.rules.json](https://github.com/Molx32/AzureUpdateManagement/blob/main/policies/policy_arc_windows.rules.json)
- [policy_arc_windows.param.json](https://github.com/Molx32/AzureUpdateManagement/blob/main/policies/policy_arc_windows.param.json)
{% highlight bash %}
text="Enforce Log Analytics agent install on Azure ARC Windows VMs"
az policy definition create --name "$text" --display-name "$text" --description "$text" --rules policy_arc_windows.rules.json --params policy_arc_windows.param.json --mode Indexed
{% endhighlight %}

#### Policy definition for Azure ARC Linux VMs
You'll find the necessary files below :
- [policy_arc_windows.rules.json](https://github.com/Molx32/AzureUpdateManagement/blob/main/policies/policy_arc_linux.rules.json)
- [policy_arc_windows.param.json](https://github.com/Molx32/AzureUpdateManagement/blob/main/policies/policy_arc_linux.param.json)
{% highlight bash %}
text="Enforce Log Analytics agent install on Azure ARC Linux VMs"
az policy definition create --name "$text" --display-name "$text" --description "$text" --rules policy_arc_linux.rules.json --params policy_arc_linux.param.json --mode Indexed
{% endhighlight %}

#### Policy definition for Azure Windows VMs
You'll find the necessary files below :
- [policy_arc_windows.rules.json](https://github.com/Molx32/AzureUpdateManagement/blob/main/policies/policy_windows.rules.json)
- [policy_arc_windows.param.json](https://github.com/Molx32/AzureUpdateManagement/blob/main/policies/policy_windows.param.json)
{% highlight bash %}
text="Enforce Log Analytics agent install on Azure Windows VMs"
az policy definition create --name "$text" --display-name "$text" --description "$text" --rules policy_windows.rules.json --params policy_windows.param.json --mode Indexed
{% endhighlight %}

#### Policy definition for Azure Linux VMs
You'll find the necessary files below :
- [policy_arc_windows.rules.json](https://github.com/Molx32/AzureUpdateManagement/blob/main/policies/policy_linux.rules.json)
- [policy_arc_windows.param.json](https://github.com/Molx32/AzureUpdateManagement/blob/main/policies/policy_linux.param.json)
{% highlight bash %}
text="Enforce Log Analytics agent install on Azure Linux VMs"
az policy definition create --name "$text" --display-name "$text" --description "$text" --rules policy_linux.rules.json --params policy_linux.param.json --mode Indexed
{% endhighlight %}


### Assign policies
We can now assign our policies to different scopes. According to the scheme, we first need to assign the following policies at the subscription level : 
- <b>Enforce Log Analytics agent install on Azure Windows VMs</b>
- <b>Enforce Log Analytics agent install on Azure Linux VMs</b>
{% highlight bash %}
policy="Enforce Log Analytics agent install on Azure Windows VMs"
name="Update management onboarding for Azure Windows VMs"
az policy assignment create --name "$name" --scope '/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' --policy "$policy"

policy="Enforce Log Analytics agent install on Azure Linux VMs"
name="Update management onboarding for Azure Linux VMs"
az policy assignment create --name "$name" --scope '/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' --policy "$policy"
{% endhighlight %}

Then, we can assign policies on OVH, OCI and GCP resource groups.
- <b>Enforce Log Analytics agent install on Azure ARC Windows VMs</b>
- <b>Enforce Log Analytics agent install on Azure ARC Linux VMs</b>
{% highlight bash %}
# For Windows machines
policy="Enforce Log Analytics agent install on Azure ARC Windows VMs"
# OVH
name="Enforce Log Analytics agent install on Azure ARC Windows VMs - OVH"
az policy assignment create --name "$name" --scope '/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' --policy "$policy"
# OCI
name="Enforce Log Analytics agent install on Azure ARC Windows VMs - OCI"
az policy assignment create --name "$name" --scope '/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' --policy "$policy"
# GCP
name="Enforce Log Analytics agent install on Azure ARC Windows VMs - GCP"
az policy assignment create --name "$name" --scope '/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' --policy "$policy"

# For Linux machines
policy="Enforce Log Analytics agent install on Azure ARC Linux VMs"
# OVH
name="Enforce Log Analytics agent install on Azure ARC Linux VMs - OVH"
az policy assignment create --name "$name" --scope '/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myGroup' --policy "$policy"
# OCI
name="Enforce Log Analytics agent install on Azure ARC Linux VMs - OCI"
az policy assignment create --name "$name" --scope '/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myGroup' --policy "$policy"
# GCP
name="Enforce Log Analytics agent install on Azure ARC Linux VMs - GCP"
az policy assignment create --name "$name" --scope '/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myGroup' --policy "$policy"
{% endhighlight %}


### Limits
If you take a look at the Windows policy, you'll notice that the policy is more complex, and that all supported Windows OS are explicitly described in the policy <b>if</b> statement. Altough this policy should work in most cases, it does not work if you use custom OS, and here is why : when using custom OS, the OS property in the VM object is referenced as <b>storageProfile.imageReference.id</b>, as you can see in the sample.

{% highlight json %}
"properties": {
        "vmId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "hardwareProfile": {
            "vmSize": "Standard_F4s_v2"
        },
        "additionalCapabilities": {
            "ultraSSDEnabled": false
        },
        "storageProfile": {
            "imageReference": {
                "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg_iac_001/providers/Microsoft.Compute/galleries/sig_iac_001/images/iac-Windows2019-Template/versions/0.0.2",
                "exactVersion": "0.0.2"
            },
{% endhighlight %}

That being said, let's now take a look at the policy : you can see that the policy never evaluates the property <b>storageProfile.imageReference.id</b>. Thus, VMs with custom OSs won't be evaluated by the policy, and the Log Analytics agent won't be onboarded. To make this work, you need to tweak the policy.
{% highlight json %}
"if": {
    "allOf": [
        {
        "field": "type",
        "equals": "Microsoft.Compute/virtualMachines"
        },
        {
        "anyOf": [
            {
            "field": "Microsoft.Compute/imageId",
            "in": "[parameters('listOfImageIdToInclude')]"
            },
            {
            "allOf": [
                {
                "field": "Microsoft.Compute/imagePublisher",
                "equals": "RedHat"
                },
                {
                "field": "Microsoft.Compute/imageOffer",
                "in": [
                    "RHEL",
                    "RHEL-SAP-HANA"
                ]
                },
                {
                "anyOf": [
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "6.*"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "7*"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "8*"
                    }
                ]
                }
            ]
            },
            {
            "allOf": [
                {
                "field": "Microsoft.Compute/imagePublisher",
                "equals": "SUSE"
                },
                {
                "anyOf": [
                    {
                    "allOf": [
                        {
                        "field": "Microsoft.Compute/imageOffer",
                        "in": [
                            "SLES",
                            "SLES-HPC",
                            "SLES-HPC-Priority",
                            "SLES-SAP",
                            "SLES-SAP-BYOS",
                            "SLES-Priority",
                            "SLES-BYOS",
                            "SLES-SAPCAL",
                            "SLES-Standard"
                        ]
                        },
                        {
                        "anyOf": [
                            {
                            "field": "Microsoft.Compute/imageSKU",
                            "like": "12*"
                            },
                            {
                            "field": "Microsoft.Compute/imageSKU",
                            "like": "15*"
                            }
                        ]
                        }
                    ]
                    },
                    {
                    "allOf": [
                        {
                        "anyOf": [
                            {
                            "field": "Microsoft.Compute/imageOffer",
                            "like": "sles-12-sp*"
                            },
                            {
                            "field": "Microsoft.Compute/imageOffer",
                            "like": "sles-15-sp*"
                            }
                        ]
                        },
                        {
                        "field": "Microsoft.Compute/imageSKU",
                        "in": [
                            "gen1",
                            "gen2"
                        ]
                        }
                    ]
                    }
                ]
                }
            ]
            },
            {
            "allOf": [
                {
                "field": "Microsoft.Compute/imagePublisher",
                "equals": "Canonical"
                },
                {
                "field": "Microsoft.Compute/imageOffer",
                "in": [
                    "UbuntuServer",
                    "0001-com-ubuntu-server-focal"
                ]
                },
                {
                "anyOf": [
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "14.04*LTS"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "16.04*LTS"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "16_04*lts-gen2"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "18.04*LTS"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "18_04*lts-gen2"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "20_04*lts"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "20_04*lts-gen2"
                    }
                ]
                }
            ]
            },
            {
            "allOf": [
                {
                "field": "Microsoft.Compute/imagePublisher",
                "equals": "credativ"
                },
                {
                "field": "Microsoft.Compute/imageOffer",
                "equals": "Debian"
                },
                {
                "anyOf": [
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "8*"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "9*"
                    }
                ]
                }
            ]
            },
            {
            "allOf": [
                {
                "field": "Microsoft.Compute/imagePublisher",
                "equals": "Oracle"
                },
                {
                "field": "Microsoft.Compute/imageOffer",
                "equals": "Oracle-Linux"
                },
                {
                "anyOf": [
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "6.*"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "7.*"
                    }
                ]
                }
            ]
            },
            {
            "allOf": [
                {
                "field": "Microsoft.Compute/imagePublisher",
                "equals": "OpenLogic"
                },
                {
                "field": "Microsoft.Compute/imageOffer",
                "in": [
                    "CentOS",
                    "Centos-LVM",
                    "CentOS-SRIOV"
                ]
                },
                {
                "anyOf": [
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "6.*"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "7*"
                    },
                    {
                    "field": "Microsoft.Compute/imageSKU",
                    "like": "8*"
                    }
                ]
                }
            ]
            },
            {
            "allOf": [
                {
                "field": "Microsoft.Compute/imagePublisher",
                "equals": "cloudera"
                },
                {
                "field": "Microsoft.Compute/imageOffer",
                "equals": "cloudera-centos-os"
                },
                {
                "field": "Microsoft.Compute/imageSKU",
                "like": "7*"
                }
            ]
            }
        ]
        }
    ]
    }
{% endhighlight json %}