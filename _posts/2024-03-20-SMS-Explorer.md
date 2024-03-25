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

## 1. Introduction
### 1.1. What are Public SMS Services?
While browsing the Internet, you probably searched for a temporary email address. For example, you created a user account on some web app, and you didn’t want to provide real information. Alternatively, you may need a temporary phone number if the app requires it: in this case, you probably used a SMS Service.

It may be simple to understand, but let's give a proper definition to what I mean by SMS Service : "An online service that provides virtual phone numbers to their customers in order to receive SMSs.". Of course, such service providers may propose additional services such as Voice Over IP (VoIP), but this is out of our scope.

There are plenty of such services out there. Using any simple online research such as <b>receive sms online</b> using your favorite search engine will lead you to multiple websites. By taking a look at the results, we can quickly notice something : some services need you to register or pay, others directly provide phone numbers.
- <b>Paid services</b> – These are services that allows customers to rent a phone number.
- <b>Registered services </b> – These are services where users are not forced to pay to access the service, but that have limited features.
- <b>Free services</b> – These are the services that are fully accessible without any form of authentication. It is interesting to note that this kind of service has a tremendous amount of trackers and ads that can even lead the web browser to freeze.


As you may think, paid and registered services provide access control, so that received SMSs can’t be seen by other users: we will call these services Private SMS Service. Similarly, since free service allow any user to read any message of any phone number, we will call these services Public SMS Services (PSS), and this is what I will discuss. Among all these PSSs, I tried to focus on [https://receive-smss.com/](https://receive-smss.com/), for multiple reasons that will be detailed in the automation section.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_0.png" class="img-fluid rounded imgc" style="display: block;margin: auto;" %}
</div>

***

### 1.2. Give it a try!
Open a new tab to have a look at https://smstome.com for example, select a phone number and check all the already received messages.
<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_2.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_3.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

You may notice that some messages seem truncated, and they actually are. I don’t know why, but this is a common issue shared by many PSS, which maybe need to optimize their data storage, especially for free PSSs. Anyway, this website is an example of Public SMS Service Provider (PSSP).

### 1.3. A few insights
In order to get some information about the interest in PSSs, we can use any trend analysis tool. Here are my results for keywords free sms online on [https://semrush.com](https://semrush.com). We can see in the bottom right corner that these keywords belong to a cluster where users seem to look for test or verification phone number. This already give insight about data we will find.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_4.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

Additionally, we can take a specific domain to analyze its traffic and see how it is used, which may be useful in order to select a high traffic PSSs for analysis.

### 1.4. About the data
Legitimately, you may wonder why would anyone spend their time reading strangers SMS. Well, let me give an example: while refreshing a PSS “received messages” web page to see the message I was waiting for, I looked at already received messages, and one of them was something like the following: “Hi <username>, your new access code is <code>. You can now connect on <website>”.

Since I couldn’t believe it, I quickly configured an anonymization pipeline and accessed the account. The website was a cryptocurrency application that allowed users to access their different wallets. I had access to personal information and to a significative amount of 65 Bitcoins.

Besides the message received represented itself a flaw, I had three hypotheses in mind:
-	<i>This is a human mistake</i> – The person owning the account used an Open SMS service ignoring what the consequences could be.
-	<i>This is a development environment</i> – In this case, the account is not a real one, or at least this is a test account, with mocking data.
-	<i>This is some kind of aggressive honeypot</i> – Since the message is received on an Open SMS service, and that it leaks the three key information needed to access the account (website, username and password), it seemed too obvious not to be a honeypot.
In the end, the domain was created three months ago, so it probably was a development environment.

Before digging more into the data that can be found, I’ll quickly discuss the automation process.


### 1.5. State of the art?
As far as I know, there isn’t any research about this topic. The only source I could find is from the Canadian ZATAZ that [quickly said a word about PSSPs](https://www.zataz.com/osint-comment-lire-les-sms-dinconnus/).

***

## 2. Automation
### 2.1. Why and how?
Before explaining anything, you may wonder why automation is needed? Because of the amount of SMSs received, our primary need is simply to collect data to duplicate the database, and develop a searching feature that would allow to identify interesting patterns. Additionally, we want our database to grow as we gather more and more SMSs, which is usually not done by PSSP as only the last 40 SMSs are displayed, at least for https://receive-smss.com/.

#### 2.1.1. Version 0.1
I developed a first version of the program which was very basic. It parses https://receive-smss.com/ and stores any unique SMS identified. After a few hours of automated SMS collection, I was excited to analyze the data, and a quickly figured out that there were two interesting cases.
-	The first one is when messages directly contain sensitive data, like credentials or PII. However, this is a very rare case, and it is complex to automate without false positive, and I didn’t want my program to generate false positive. 
-	The second one is when messages contain URLs. This is a much more frequent case, and it represents 99% of interesting messages at this point. Why? Because anything can hide behind a URL, a scam or an account takeover.
I abandoned the first case for now, and I focused on messages that contain URLs


#### 2.1.2. Version 0.2
To make the analysis a bit simpler, I developed a web interface where I can look for messages collected, and easily navigate to URLs. Thanks to this tiny improvement, I built a list of all URLs and I manually navigated to each one of them, and for some of them simply gave me access to user accounts. At this point, I decided to develop the last core feature: the <b>aggressive mode</b>. This can be turned on to navigate to URLs known to hide interesting data, then it will add this data to the database!

However, I faced a challenge that is still not resolved: <b>race conditions</b>. Some URLs, such as password reset URLs, have either an expiration date, or can be accessed only once to prevent someone else to reset a password a second time. I decided not to focus on that problem for now, because there are a lot of URLs to explore.

#### 2.1.3. Version 1.0
The whole architecture is made of Docker containers. The entry point from the user perspective is the <b>Flask</b> web server. Its role is to render and serve web pages, and to enqueue different jobs in a <b>Redis queue</b>:
-	A job that will initialize various things
-	A job that will collect messages (infinite loop)
-	A job that will collect data, only in aggressive mode (infinite loop)
These jobs will be executed by two <b>workers</b>. Finally, we have a database with SMSs, targets, configuration, and so on.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_automation_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

#### 2.1.4. Version 1.1
Aggressive mode is great to collect data, but the issue is that each domain must be parsed differently to retrieve data. To resolve this, I created a parent class named <b>DataModule</b> with several generic methods that will used by different children classes.

Let me explain a practical example: in multiple cases, data can be retrieved simply by reading the <b>Location</b> header of a <b>HTTP 302 Redirect</b>, which directly contains data. Thus, I wrote a method for the DataModule class named <b>_retrieve_data_redirect()</b> that simply return data found in the location header. Now, when I want to add a new parser for a given domain, I simply need to call this method, which makes the parser very simple to implement c.f. complete class for AirIndia parser below.

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

### 2.2. Current limits
#### 2.2.1 The passive mode
##### 2.2.1.1 Additional sources
At some point, additional data source may be needed for this tool to handle more data. I didn’t plan to work on that yet, but still, I thought about some criterions that seems interesting to consider when selecting a PSSP as a data source.

#### 2.2.2 The aggressive mode
##### 2.2.2.1.Race conditions
As quickly mentioned before, data collection may be limited by race conditions. This occurs either when the URL expires, or when the URL can be accessed only once.

##### 2.2.2.2. Security measures
Another limit for data collection is security.
- First, when navigating really often to some websites, a <b>HTTP 429 Too Many Requests</b> may be issued by the server.
- Second, when navigating to a URL after a legitimate user already navigated to, it may trigger security controls such as location.

##### 2.2.2.3. Format
Also, some URLs can be accessed only from the corresponding mobile application. In some cases, the HTTP response prevents access and does not provide any useful data. In some other cases, this behavior may be bypassed by changing the user agent.

### Data model


***

## 3. Results
Now is the time to finally talk about cybersecurity, OSINT, CTI, and OffSec! This section describes the different use cases I encountered while analyzing hundreds of thousands of SMS. For each use case, I tried to dig as deeper as I could to provide a real world impact analysis.
Also, from an attacker perspective, all the presented results are opportunistic, which means the attacker is not able to choose its target and entirely relies on which account or data URLs may redirect to.

Whether we could expect it or not, some URLs redirect to an authenticated session on various websites or mobile applications. Every
### 3.1. Use cases
#### 3.1.1. Password reset URLs
The first scenario for that use case is very simple, and I detected it for multiple companies: the message received contains a password reset URL. For example:
> <i>You can reset your password by clicking on this link: [https://pprod.mycompany.com/account?reset-token=RANDOM_GUID](https://pprod.mycompany.com/account?reset-token=RANDOM_GUID)</i>

As you noticed with the pprod subdomain, I only noticed that use case for non-production environment. But still, I set up an anonymization pipeline and I was able to access the password reset page, set a new password, then access the application as an administrator.

#### 3.1.2. Bypass access control
This scenario is even simpler, and simply gives access as an authenticated user when clicking on the URL. For example:
> <i>Click here to access your account : [https://shrtn.com/0123456789abcdefgh](https://shrtn.com/0123456789abcdefgh)</i>

#### 3.1.3. Data exposure
While this scenario is not as critical as the previous ones, it gives the opportuniy to collect various types of data, base on the concerned application business.
> <i>Click here to check your analysis results : [https://shrtn.com/0123456789abcdefgh](https://shrtn.com/0123456789abcdefgh)</i>

#### 3.1.4. Data discovery
This scenario is very rare and not very sensitive. The only example is the enumeration of subdomains, where the URL is not accessible <i>i.e.</i> does not lead to account takeover from the public internet. For example :
> <i>Hello admin, click on the URL to reset your password : [https://internal.and.sensitive.domain.com/0123456789abcdefgh](https://internal.and.sensitive.domain.com/0123456789abcdefgh)</i>

#### 3.1.5. Missed opportunities
Missed opportunities are almost exclusively use cases where the whole URL can't be read, because it is too long. For example :
> <i>Hi, how are you? Did you ask for a password reset? Here is your URL, don't share it! [https://superlongandweirddomainnamerightquestionmark.com/?reset=A](https://superlongandweirddomainnamerightquestionmark.com/?reset=A) </i>

### 3.2. Statistics
Statictics must be added.

### 3.3. Targets
#### 3.3.1. Interesting targets
From a Cybersecurity Threat Intelligence (CTI) perspective, any identified account should be assumed compromised.
<table id="custom" class="t-border">
<caption style="text-align:center"><b>Resource deployed</b></caption>
  <tr>
    <th>Target</th>
    <th>Categories</th>
  </tr>
  <tr>
    <td>Venmo</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Instagram</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Bieldronka</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Job logic</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Free ads</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Payfone</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>AirIndia</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Texts from my ex</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Validahealth</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Stickermule</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Wallester</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Bankinter</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Abinitio Solutions</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Communidad Madrid</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>UPS Scam</td>
    <td>update management</td>
  </tr>
  <tr>
    <td>Experian</td>
    <td>update management</td>
  </tr>
</table>


#### 3|2|1|1  Venmo
Venmo is some sort of banking application that allows users to send payment to their friends and so on. This is a sensitive business. The message received for Venmo is “Reset your Venmo password: https://venmo.com/account/password-new?utm_medium=phone&reset_key=40b11eeecc001f5a6d2cc4f40eae408c7bc953f11478eb028032c4f86“.
When navigating to the URL, no other information than just setting a password. You can access the interface even if the link has expired, which is an interesting way of handling password reset procedures as it may result in a waste of time from an attacker perspective.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_results_venmo_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

#### 3|2|1|3 Instagram
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