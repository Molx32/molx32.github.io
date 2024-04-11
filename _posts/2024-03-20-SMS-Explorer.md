---
layout: post
title: SMS Explorer - An misknown blah
date: 2023-03-21 10:00:00
description: This research 
tags: SMSExplorer
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
<i>I recently needed to register on an web application, but I didn’t want to provide my real phone number, so I used a temporary phone number to receive a validation SMS. To achieve this, I took a look at those websites proposing free phone numbers to receive SMSs, visible by anyone: this is what I call <b>Public SMS Services (PSSs)</b>. The reason I decided to investigate PSSs is that their content may be useful from both a Cyber-Threat Intelligence (CTI) and Open-Source Intelligence (OSINT) perspective, and even Offensive Security (OffSec).</i>

***

#### Plan
- [1. Introduction](#1-introduction)
- [2. Automation](#2-automation)
- [3. Results](#3-results)

***

## Introduction
#### What are Public SMS Services?
While browsing the Internet, you probably searched for a temporary email address. For example, you created a user account on some web app, and you didn’t want to provide real information. In the same way, you may need a temporary phone number if the app requires it: in this case, you probably used a <b>virtual phone number</b>, which allows you to receive a confirmation SMS or so, in order to validate your registration on the concerned web app. Although this use case is quite simple, let's give a proper definition of what I call <b>SMS services</b>.

> A <b>SMS Service</b> is an online service that provides virtual phone numbers to their customers in order to receive SMSs. If a service provides additional services, such as VoIP, it will still be regarded as a SMS Service.

There are plenty of such services out there. Using any simple online research such as [receive sms online](https://letmegooglethat.com/?q=receive+sms+online) using your favorite search engine will lead you to multiple websites.

#### An example
Let's take an example real quick and have a look at https://smstome.com for example, select a phone number and check all the already received messages.
<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_2.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_3.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

You may notice that some messages seem truncated, and they actually are. I don’t know why, but this is a common issue shared by many PSS, which maybe need to optimize their data storage, especially for free PSSs.

#### A few insights
In order to get some information about the interest in PSSs, we can use any trend analysis tool. Here are my results for keywords free sms online on [https://semrush.com](https://semrush.com). We can see in the bottom right corner that these keywords belong to a cluster where users seem to look for test or verification phone number. This already give insight about data we will find.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_4.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

Additionally, we can take a specific domain to analyze its traffic and see how it is used, which may be useful in order to select a high traffic PSSs for analysis.

#### Security issue
When looking at these services, we can see that some are paid, others are free. There are two main differences between the two :
- <p><b>Privacy</b> - For users looking for privacy, free services are prefered because they don't require users to create an account, while paid services require users to provide personal information.</p>
- <p><b>Access control</b> - When subscribing to a paid service, virtual phone numbers are private i.e. a user can have multiple virtual phone number, but a virtual phone number can be owned only by one user. However with free services there is not form of access control, and virtual phone numbers can be shared, this SMS are too.</p>

Obviously, the only thing we can exploit here is free SMS services, that I will call <b>Public SMS Services</b> from now.

> <p style="margin-top:none;">A <b>Public SMS Service</b> is a SMS Service that do not provide any access control mechanism, and allows multiple users to share the same virtual phone numbers, and to read each other messages.</p>

Here is a quick summary of the different SMS Services categories.
<table id="custom" class="t-border">
  <caption style="text-align:center"><b>SMS Services categories</b></caption>
  <tr>
    <th>Target</th>
    <th>Categories</th>
    <th>Privacy</th>
    <th>Access control</th>
  </tr>
  <tr>
    <td>Paid services</td>
    <td>These are services that allows customers to pay for a virtual phone numbers, usually to send and receive SMSs. With such services, a phone number will be dedicated to only one user, which means access control is implemented which is good. However, this also implies that users need to create an account, which doesn't help us with privacy.</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">✅</td>
  </tr>
  <tr>
    <td>Registered services</td>
    <td>These are services that allows customers to get a virtual phone number only if they create an account. Again, access control can be implemented but the account creation is not privacy friendly.</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">⚠️</td>
  </tr>
  <tr>
    <td>Free services</td>
    <td>These are the services that are freely accessible without any form of authentication. Here, a phone number will not be dedicated to only one user, which prevents to implement access control.</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
  </tr>
</table>

To be more specific about our study scope, here is a scheme. You'll notice that I exclusively rely on [https://receive-smss.com/](https://receive-smss.com/) for now, for multiple reasons that will be detailed in the automation section.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_0.png" class="img-fluid rounded imgc" style="display: block;margin: auto;" %}
</div>

#### Exploiting the security issue
Legitimately, you may wonder why would anyone spend their time reading strangers SMS. Well, let me give an example: while refreshing a the "received messages” web page on a Public SMS Service to see the message I was waiting for, I took a look at already received messages, and one of them was something like the following: <code>Hi redacted@gmail.com, your new access code is 458953. You can now connect on https://REDACTED.com.</code>.
<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_1.gif" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

Since I couldn’t believe it, I quickly configured an anonymization pipeline and accessed the account. The website was a cryptocurrency application that allowed users to access their different wallets. I had access to personal information and to a significative amount of 65 Bitcoins.

Besides the message received represented itself a flaw, I had three hypotheses in mind:
1.	<b>This is a human mistake</b> – The person owning the account used a Public SMS Service ignoring what the consequences could be.
2.	<b>This is a development environment</b> – In this case, the account is not a real one, or at least this is a test account, with mocking data.
3.	<b>This is some kind of aggressive honeypot</b> – Since the message is received on an Open SMS service, and that it leaks the three key information needed to access the account (website, username and password), it seemed too obvious not to be a honeypot.
In the end, the domain was created three months ago, and all features were not implemented yet, so it probably was a development environment.

#### State of the art
A really quick word about existing reasearch on this topic : the only source I could find is from the Canadian ZATAZ that [published a short post about Public SMS Services](https://www.zataz.com/osint-comment-lire-les-sms-dinconnus/).

***
<br>
<br>
<br>


## Automation
<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_automation_1.jpg" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

#### Why?
First, automation is needed because of the amount of SMSs received. My primary need was to collect data and store it in a database, becauses the source data source I rely on only provides the last 40 SMS received. Then, I needed to develop a searching feature that would allow to identify interesting patterns. Finally, once patterns are identified, a last feature was needed to exploit them.

#### How?
##### First approach
I developed a first version of the program which was very basic. It parses [https://receive-smss.com/](https://receive-smss.com/) and stores any unique SMS identified.
After a few hours spent collecting SMSs, I was excited to analyze the data, and I quickly figured out that there were two interesting cases.
-	The first one is when messages directly contain sensitive data, like credentials or PII. However, this is a very rare case, and it is complex to automate without false positive, and I didn’t want my program to generate false positive. 
-	The second one is when messages contain URLs. This is a much more frequent case, and it represents 99% of interesting messages at this point. Why? Because anything can hide behind a URL, a scam or an account takeover.

I abandoned the first case for now, and I focused on messages that contain URLs


##### Improvements
To make the analysis a bit simpler, I developed a web interface where I can look for messages collected, and easily navigate to URLs. Thanks to this tiny improvement, I built a list of all URLs and I manually navigated to each one of them, and for some of them simply gave me access to user accounts. At this point, I decided to develop the last core feature: the <b>aggressive mode</b>. This can be turned on to navigate to URLs known to hide interesting data, then it will add this data to the database!

However, I faced a challenge that is still not resolved: <b>race conditions</b>. Some URLs, such as password reset URLs, have either an expiration date, or can be accessed only once to prevent someone else to reset a password a second time. I decided not to focus on that problem for now, because there are a lot of URLs to explore.

##### Architecture
The whole architecture is made of Docker containers. The entry point from the user perspective is the <b>Flask</b> web server. Its role is to render and serve web pages, and to enqueue different jobs in a <b>Redis queue</b>:
-	A job that will initialize various things
-	A job that will collect messages (infinite loop)
-	A job that will collect data, only in aggressive mode (infinite loop)

These jobs will be executed by two <b>workers</b>. Finally, we have a database with SMSs, targets, configuration, and so on.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_automation_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

##### Latest release
Aggressive mode is great to collect data, but the issue is that each domain must be parsed differently to retrieve data. To resolve this, I created a parent class named <b>DataModule</b> with several generic methods that will used by different children classes.

Let me explain a practical example: in multiple cases, data can be retrieved simply by reading the <b>Location</b> header of a <b>HTTP 302 Redirect</b>, which directly contains data. Thus, I wrote a method for the DataModule class named <b>_retrieve_data_redirect()</b> that simply return data found in the location header. Now, when I want to add a new parser for a given domain, I simply need to call this method, which makes the parser very simple to implement <i>c.f.</i> complete class for AirIndia parser below.

{% highlight python %}
class AirIndia(DataModule):
    def __init__(self):
        name        = 'AirIndia'
        base_url    = 'https://nps.airindia.in/'
        super().__init__(name, base_url)
    
    def retrieve_data(self, url, msg):
        return self._retrieve_data_redirect(url)
{% endhighlight %}

***

#### Limitations
##### SMSs sources
As previously mentioned, the only SMS source is [https://receive-smss.com/](https://receive-smss.com/). At some point, additional SMS sources may be needed for this tool to handle more data. I didn’t plan to work on that yet.

##### Data collection
Again, as mentioned before, data collection is limited in multiple ways :
- Race conditions, which happens to be a problem when the URL expires, or when the URL can be accessed only once.
- Requests rate limiting, which causes the targeted server to return a <code>HTTP 429 Too Many Requests</code> and prevents us to parse additional data for a few seconds or minutes.
- Other security controls, that may be related to users agents, locations, and so on.
- Sometimes, URLs can be accessed only from the corresponding mobile application. In some cases, the HTTP response prevents access and does not provide any useful data. In some other cases, this behavior may be bypassed by changing the user agent.


***

## Results
Now is the time to finally talk about cybersecurity, OSINT, CTI, and OffSec! This section describes the different use cases I encountered while analyzing hundreds of thousands of SMS. For each use case, I tried to dig as deeper as I could to provide a real world impact analysis.
Also, from an attacker perspective, all the presented results are <b>opportunistic</b>, which means the attacker is not able to choose its target and entirely relies on which account or data URLs may redirect to.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_results_1.gif" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

Whether we could expect it or not, some URLs redirect to an authenticated session on various websites or mobile applications. Every

 The following table shows the different use cases.

### Use cases

<table id="custom" class="t-border">
  <caption style="text-align:center"><b>SMS security qualification</b></caption>
  <tr>
    <th>Use case</th>
    <th>Description</th>
    <th>Application</th>
  </tr>
  <tr>
    <td> <a href="#password-reset-urls">Password reset URLs</a> </td>
    <td>Account takeover</td>
    <td>CTI, OffSec, OSINT</td>
  </tr>
  <tr>
    <td><a href="#bypass-access-control">Bypass access control</a></td>
    <td>Account takeover</td>
    <td>CTI, OffSec, OSINT</td>
  </tr>
  <tr>
    <td><a href="#data-exposure">Data exposure</a></td>
    <td>Allows users to gather information without compromising any account.</td>
    <td>CTI, OffSec, OSINT</td>
  </tr>
  <tr>
    <td><a href="#data-discovery">Data discovery</a></td>
    <td>Allows users to gather information about a specific target <i>e.g.</i> subdomain discovery.</td>
    <td>CTI, OffSec, OSINT</td>
  </tr>
  <tr>
    <td><a href="#missed-opportunities">Missed opportunities</a></td>
    <td>URLs that can't be exploited, mainly because they are truncated.</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="#scams">Scams</a></td>
    <td>URLs that lead to explicit scam <i>e.g.</i> fake UPS website.</td>
    <td>-</td>
  </tr>
</table>

##### Password reset URLs
The first scenario for that use case is very simple, and I detected it for multiple companies: the message received contains a password reset URL. For example:
> <i>You can reset your password by clicking on this link: [https://pprod.mycompany.com/account?reset-token=RANDOM_GUID](https://pprod.mycompany.com/account?reset-token=RANDOM_GUID)</i>

As you noticed with the pprod subdomain, I only noticed that use case for non-production environment. But still, I set up an anonymization pipeline and I was able to access the password reset page, set a new password, then access the application as an administrator.

##### Bypass access control
This scenario is even simpler, and simply gives access as an authenticated user when clicking on the URL. For example:
> <i>Click here to access your account : [https://shrtn.com/0123456789abcdefgh](https://shrtn.com/0123456789abcdefgh)</i>

##### Data exposure
While this scenario is not as critical as the previous ones, it gives the opportuniy to collect various types of data, base on the concerned application business.
> <i>Click here to check your analysis results : [https://shrtn.com/0123456789abcdefgh](https://shrtn.com/0123456789abcdefgh)</i>

##### Data discovery
This scenario is very rare and not very sensitive. The only example is the enumeration of subdomains, where the URL is not accessible <i>i.e.</i> does not lead to account takeover from the public internet. For example :
> <i>Hello admin, click on the URL to reset your password : [https://internal.and.sensitive.domain.com/0123456789abcdefgh](https://internal.and.sensitive.domain.com/0123456789abcdefgh)</i>

##### Missed opportunities
Missed opportunities are almost exclusively use cases where the whole URL can't be read, because it is too long. For example :
> <i>Hi, how are you? Did you ask for a password reset? Here is your URL, don't share it! [https://superlongandweirddomainnamerightquestionmark.com/?reset=A](https://superlongandweirddomainnamerightquestionmark.com/?reset=A) </i>

### Statistics
Statictics must be added.

### Real world examples

<table id="custom" class="t-border">
<caption style="text-align:center"><b>Resource deployed</b></caption>
  <tr>
    <th>Target</th>
    <th>Password reset URLs</th>
    <th>Bypass access control</th>
    <th>Data exposure</th>
    <th>Data discovery</th>
    <th>Missed opportunities</th>
  </tr>
  <tr>
    <td>Venmo</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Instagram</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Bieldronka</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Job logic</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Free ads</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Payfone</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>AirIndia</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Texts from my ex</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Validahealth</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Stickermule</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Wallester</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Bankinter</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Abinitio Solutions</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Communidad Madrid</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>UPS Scam</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td>Experian</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
</table>

<!-- VENMO -->
##### Venmo
Venmo is some sort of banking application that allows users to send payment to their friends and so on. This is a sensitive business. The message received for Venmo is :
> <i>“Reset your Venmo password: [https://venmo.com/account/password-new?utm_medium=phone&reset_key=40b11eeecc001f5a6d2cc4f40eae408c7bc953f11478eb028032c4f86](https://venmo.com/account/password-new?utm_medium=phone&reset_key=40b11eeecc001f5a6d2cc4f40eae408c7bc953f11478eb028032c4f86)</i>

When navigating to the URL, no other information than just setting a password. You can access the interface even if the link has expired, which is an interesting way of handling password reset procedures as it may result in a waste of time from an attacker perspective.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_results_venmo_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<!-- INSTAGRAM -->
##### Instagram
This second scenario is very similar to the previous one, but applies to Instagram, which seems very exciting at first glance. I detected this simply because there were a lot of Instagram shorten URLs e.g. https://ig.me/0123456789abcdef. This kind of URL is not necessarily sensitive and may for instance redirect to a simple public picture hosted on https://instagram.com/.
Navigating to those URLs would redirect to an Instagram error page displaying something like “The URL has expired”. I needed to dig deeper, created an Instagram account, and triggered a password reset procedure using a temporary phone number. Then, I was able to confirm that these URLs would allow an opportunistic account takeover. The text message received look like these:
-	Tap to reset your Instagram password: https://ig.me/1ZZOeHuLUwa4Ttq – In this case, we can just skip the password reset and be logged in as the user.
-	Instagram link: https://ig.me/r6LPOYK87HRyT73. Don't share it. – In this case, we must complete the password reset procedure.
Note: depending on the carrier, messages may never be sent to the temporary phone. For instance, for France-based temporary phone numbers, it seems that these phone numbers are somehow blacklisted.

So, what is the impact? A common intuition would be that one Instagram account is associated with one phone number. Thus, if a PSS proposes a hundred phone number, an attacker could only takeover a maximum of a hundred user accounts. Spoiler alert: this is wrong. I observed that for a given PSS phone number, multiple Instagram password reset URLs were received, leading to different user accounts. This seemed irrelevant to me. After some testing, I observed that it is not possible to register a new Instagram account with a phone number when the phone number is already used by an existing account. However, it is possible to edit the contact information to add a phone number. Here is an example:

Imagine that Instagram has a user jdoe@email.com registered with +4591457271. If this user triggers a password reset, a text message will be received on the PSS hosting this phone number, and we could steal the account. Now, imagine that a new user creates a new account with an email address leoagata@proton.me, and then edit their contact information and set their phone number to +4591457271. The immediate consequence is that this the phone number will be removed from jdoe@email.com account. Now, if leoagata triggers a password reset, a text message will be received on the PSS hosting this phone number, and we could steal the account. This means that we can receive multiple passwords reset URLs from different user accounts on the same phone number.


### 3|2|2  Bypass access control
#### 3|2|2|1  Job Logic
JobLogic is an Enterprise Resource Management SaaS application that provides many different services such as payment integration, jobs management, planning, and so on. Here is the message received : “Your Job has been completed Please follow the link for details https://staging-portal.joblogic.com/Portal?id=eb5ba3f3-ba5d-4483-a7f1-3f9b7cc29aab.”

Navigating to this URL directly provides access to a portal, authenticated as a user. As you noticed, this is a staging environment, which means that no real data should be found.

### 3.2.2. Missed opportunities