---
layout: post
title: SMS Explorer - An misknown blah
date: 2025-06-15 10:00:00
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
- [Introduction](#introduction)
- [Automation](#automation)
- [Results](#results)

***

## Introduction
A really quick word about existing reasearch on this topic : the only source I could find is from the Canadian ZATAZ that [published a short post about Public SMS Services](https://www.zataz.com/osint-comment-lire-les-sms-dinconnus/).

#### What are Public SMS Services?
While browsing the Internet, you probably searched for a temporary email address at least once. For example, you created a user account on some web app, and you didn’t want to provide real information. In the same way, you may need a temporary phone number if the app requires it: in this case, you probably used a <b>virtual phone number</b>, which allows you to receive a confirmation SMS or so, in order to validate your registration on the concerned web app. Although this use case is quite simple, let's give a proper definition of what I call <b>SMS services</b>.

> A <b>SMS Service</b> is an online service that provides virtual phone numbers to their customers in order to receive SMSs. If a service provides additional services, such as VoIP, it will still be considered a SMS Service.

There are plenty of such services out there. Using any simple online research such as [receive sms online](https://letmegooglethat.com/?q=receive+sms+online) using your favorite search engine will lead you to multiple websites.

#### An example
Let's take an example real quick and have a look at [https://smstome.com](https://smstome.com). Just select a phone number : congratulation, you can read all messages received.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_intro_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_intro_2.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

If you want to do some testing, simply try to register on a web app that requires a phone number, and check if you receive a verification SMS. Actually it might not work, because after some time, those phone numbers are considered malicious or suspicious by carriers and applications. To ensure the service is still available, SMS service providers simply retire old phone numbers and replace them with new ones.

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_intro_3.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

You may notice that some messages seem truncated, and they actually are. I don’t know why, but this is a common issue shared by many SMS services, which maybe need to optimize their data storage, especially for free services.

#### Security issue
When looking at these services, we can see that some are billed, others are free. There are two main differences between the two :
- <p><b>Privacy</b> - For users looking for privacy, free services are prefered because they don't require users to create an account, while paid services require to do so <i>i.e.</i> require users to provide personal information.</p>
- <p><b>Access control</b> - When subscribing to a paid service, virtual phone numbers are private <i>i.e.</i> a user can have multiple virtual phone numbers, but a virtual phone number can be owned only by one user. However with free services there is no form of access control, and virtual phone numbers can be shared, so are SMSs.</p>

Obviously, the only thing we can exploit here is free SMS services, that I will now call <b>Public SMS Services (PSS)</b>, which is not an official term.

> <p style="margin-top:none;">A <b>Public SMS Service (PSS)</b> is a SMS Service that do not provide any access control mechanism, and allows multiple users to share the same virtual phone numbers, and thus allows users to read each other messages.</p>

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
    <td>These are services that allow customers to pay for a virtual phone numbers, usually to send and receive SMSs. With such services, a phone number will be dedicated to only one user, which means access control is implemented, which is good. However, this also implies that users need to create an account, which doesn't help with privacy.</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">✅</td>
  </tr>
  <tr>
    <td>Registered services</td>
    <td>These are services that allow customers to get a virtual phone number only if they create an account. Again, access control can be implemented but the account creation is not privacy friendly.</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">⚠️</td>
  </tr>
  <tr>
    <td>Free services</td>
    <td>These are the services that are freely accessible without any form of authentication. Here, a phone number will not be dedicated to only one user, which is not compatible with access control.</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
  </tr>
</table>

Before digging those services, here is a scheme representing our study scope. As you can see, I only studied SMS received on [https://receive-smss.com/](https://receive-smss.com/), which is a public and free SMS service. I decided to focus only this data source because it was pretty simple to automate data collection, but also because this web sites received a lot of messages.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_intro_0.png" class="img-fluid rounded imgc" style="display: block;margin: auto;" %}
</div>

#### Exploiting the security issue
You may wonder why would anyone spend their time reading strangers SMS. Well, let me give an example: while refreshing a "received messages” web page on a PSS to see the message I was waiting for, I took a look at already received messages, and one of them was something like the following: <code>Hi redacted@gmail.com, your new access code is 458953. You can now connect on https://REDACTED.com.</code>.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_intro_1.gif" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

Since I couldn’t believe it, I quickly configured an anonymization pipeline and accessed the account. The website was a cryptocurrency application that allowed users to access their different wallets. I had access to personal information and to a significative amount of 65 Bitcoins.

Besides the message received that represented itself a flaw, I had three hypothesis in mind:
1.	<b>This is a human mistake</b> – The person owning the account used a Public SMS Service ignoring what the consequences could be.
2.	<b>This is a development environment</b> – In this case, the account is not a real one, or at least this is a test account, with mocking data.
3.	<b>This is some kind of aggressive honeypot</b> – Since the message is received on an PSS, and that it leaks the three key information needed to access the account (website, username and password), it seemed too obvious not to be a honeypot.

In the end, the domain was created three months ago, and all features were not implemented yet, so it probably was a development environment.

<br>
<br>

Altough this is a quite simple use case, let's define some kind of attack model. We have two major use cases which are SMSs without URL, and SMSs with URL.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_security_gif_1.gif" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Case 1 - SMS without URL
In this first case, <code>User A</code> gets a virtual phone number from a PSS, they use this phone number to register on a web application that will store all user-related data into its database.
At some point, the application will be triggered to send a SMS, for instance to invite the user to come back to the app : <code>Hey John DOE, you have a new offer on your house located at Av. de Cointe 5, 4000 Liège, Belgium!</code>.
The SMS will be sent to the phone number associated to <code>User A</code> and thus sent to the PSS.

Now, if a malicious <code>User B</code> spends their time watching SMSs on the PSS, they could read PII, which is the equivalent of an indirect read access to the application database. Alternatively, the SMS may also contain some PIN or password that can be used to authenticate on an app with the phone number as a user ID. In this case, <code>User B</code> could compromise the account, and have an indirect read/write access to <code>User A</code> information.




###### Case 2 - SMS with URL
In this second case, <code>User A</code> gets a virtual phone number from a PSS, they use this phone number to register on a web application that will store all user-related data into its database.
At some point, the application will be triggered to send a SMS, for example <code>Hey John DOE, you have a new match, send them a message : https://findselfesteem.com/messages?token=eyf652s4fz6e621sdf65z4szdfzrgzd62x1fz6ver5g1</code>. The SMS will be sent to the phone number associated to <code>User A</code> and thus sent to the PSS.

Now, if a malicious <code>User B</code> spends their time to watch SMSs on the PSS, they could navigate to the URL, and either reset a password, or directly compromise <code>User A</code> account, and have an indirect read/write access to <code>User A</code> information.

<br>

#### Regulatory compliance
##### GDPR
If you see a message containing personal information, this may not be [GDPR](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32016R0679) compliant because of the Article 32 that says the following.
<i>
> [...] the controller and the processor shall implement appropriate technical and organisational measures to ensure a level of security appropriate to the risk, including inter alia as appropriate:
>- <b>(a) the pseudonymisation and encryption of personal data;</b>
>- (b) the ability to ensure the ongoing confidentiality, integrity, availability and resilience of processing systems and services;
>- (c) the ability to restore the availability and access to personal data in a timely manner in the event of a physical or technical incident;
>- (d) a process for regularly testing, assessing and evaluating the effectiveness of technical and organisational measures for ensuring the security of the processing.
</i>

From the <b>PSS point of view</b>, data is simply received and displayed as is. We can compare it to a forum where users can post anything they want, but here users are actually <b>senders</b>.
From the <b>senders</b> point of view, the <b>end user</b> is supposed to provide real information. Based on that assumption, the sender will trust the provided phone number.
Finally, the <b>end user</b> is informed that a real information is expected, and that PSS will publish SMS associated to the app. The problem is that the end user doesn't know when SMS will be sent (except for password reset and verification), and what the content will be.

##### Electronic Communications Privacy Act

<br>

#### A few insights
In order to get some information about the interest in PSSs, we can use any trend analysis tool. Here are my results for keywords <code>free sms online</code > on [https://semrush.com](https://semrush.com). We can see in the bottom right corner that these keywords belong to a cluster where users seem to look for test or verification phone number. This already give insight about data we will find. Additionally, we can take a specific domain to analyze its traffic and see how it is used, which may be useful in order to select a high traffic PSS for analysis.

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_intro_4.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>
<br>
After a few week of SMS collection on [https://receive-smss.com/](https://receive-smss.com/), we can observe that the number of messages received on a hourly basis approaches 2000 SMS/h.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_stats_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>
<br>
Then, we can observe how phone numbers are used based on their countries. This is only an approximation because phone are periodically retired and replaced by new ones, not necessarily from the same country. What we can see on the following chart is the number of message receive for each country.
<br>
Then, if we divide this total by the number of phones belonging to that country, we get the mean number of messages received by phone by country, which is in other terms how frequently a phone will receive messages. We can see that Indian phone numbers are the most active with <b>47801</b> total messages received on the <b>7 last days</b>, with only two phone numbers available, while UK received <b>82937</b> with <b>14</b> phone numbers available.

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_stats_2.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_stats_2.1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>
<br>
Besides the amount of received message, the most valuable type of SMS are the ones that contain URL. The following chart shows the number of message containing URLs received per hour, which approches <b>49 SMS/h</b>, which which is <b>2.45%</b> of the total <b>2000 SMS/h</b>. Alternatively, if we take a look at the total of messages received over the <b>last 14 days</b>, this is approx. <b>2.49%</b> of messages containing URLs. But don't be fooled, this doesn't mean that <b>2.49%</b> of messages received are interesting : actually most messages containg URL are spams, ads, or are incomplete messages. That's why we need AUTOMATION!
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_stats_3.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_stats_4.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<br>
<br>

***

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_automation_1_gif.gif" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

## Automation
#### Why?
My primary need was to collect data and store it in a database, becauses the source data source I rely on only provides the last 40 SMS received. Then, I needed to develop a searching feature that would allow to identify interesting patterns <i>e.g.</i> messages with PII. Finally, once patterns are identified, a last feature was needed to exploit them.

#### How?
##### First approach
I developed a first very basic version of the solution. It parses [https://receive-smss.com/](https://receive-smss.com/) and stores any unique SMS identified.
After a few hours spent collecting SMSs, I was excited to analyze the data, and I quickly figured out that there were two interesting cases.
-	The first one is when messages directly contain sensitive data, like credentials or PII. However, this is a very rare case, and it is complex to automate without false positive, and I didn’t want to deal with false positive at this point. 
-	The second one is when messages contain URLs. This is a much more frequent case, and it represents a huge part of interesting messages. Why? Because anything can hide behind a URL, a scam or an account compromise.

I abandoned the first case and focused on messages that contain URLs.


##### Improvements
To make the analysis a bit simpler, I developed a web interface where I can look for messages collected, and easily navigate to URLs. Thanks to this tiny improvement, I built a list of all URLs and I manually navigated to each one of them, and some of them simply gave me access to user accounts. Thus, I decided to develop the last core feature: the <b>aggressive mode</b>. This can be turned on to navigate to URLs known to hide interesting data, then it will add this data to the database!

However, I faced a challenge that is still not resolved: <b>race conditions</b>. Some URLs, such as password reset URLs, have either an expiration date, or can be accessed only once to prevent someone else to reset a password a second time. I decided not to focus on that problem for now, because there are a lot of URLs to explore.

##### Architecture
The whole architecture is made of Docker containers. The entry point from the user perspective is the <b>Flask</b> web server. Its role is to render and serve web pages, and to enqueue different jobs in a <b>Redis queue</b>:
-	A job that will initialize various things
-	A job that will collect messages (infinite loop)
-	A job that will collect data, only in aggressive mode (infinite loop)

These jobs will be executed by two <b>workers</b>. Finally, we have a database with SMSs, targets, configuration, and so on.

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_automation_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

##### Latest release
Aggressive mode is great to collect data, but the issue is that each domain must be parsed differently to retrieve data. To resolve this, I created a parent class named <b>DataModule</b> with several generic methods that will used by different children classes.

Let me explain a practical example: in multiple cases, data can be retrieved simply by reading the <b>Location</b> header of a <b>HTTP 302 Redirect</b>, which directly contains data. Thus, I wrote a method for the DataModule class named <b>_retrieve_data_redirect()</b> that simply return data found in the location header. Now, when I want to add a new parser for a given domain, I simply need to call this method, which makes the parser very simple to implement <i>c.f.</i> complete class for REDACTED parser below.

{% highlight python %}
class REDACTED(DataModule):
    def __init__(self):
        name        = 'REDACTED'
        base_url    = 'https://nps.REDACTED.in/'
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

##### Quick demo
Long story short, here is a demo.
<div class="col-sm mt-3 mt-md-0">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/6asRTPWqYmg" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

***

## Results
Now is the time to finally talk about cybersecurity, OSINT, CTI, and OffSec! This section describes the different use cases I encountered while analyzing hundreds of thousands of SMS. For each use case, I tried to dig as deeper as I could to provide a real world impact analysis.

<table id="custom" class="t-border">
  <caption style="text-align:center"><b>Cybersecurity fields</b></caption>
  <tr>
    <th>Field</th>
    <th>Why is it useful?</th>
  </tr>
  <tr>
    <td>OSINT</td>
    <td>This may be the most useful usecase since a lot of user accounts can be identified, associated to a location, an email address, a physical address, and so on.</td>
  </tr>
  <tr>
    <td>CTI</td>
    <td>This is only useful when a company account can be tagged as compromised and then handled by security teams.</td>
  </tr>
  <tr>
    <td>Offensive Security</td>
    <td>This is only useful when a company account can be compromised, rather than a personnal account such as Instagram.</td>
  </tr>
</table>
Also, from an attacker perspective, all the presented results are <b>opportunistic</b>, which means the attacker is not able to choose its target and entirely relies on which account or data URLs may redirect to.

### Use cases
The following table shows the different use cases encountered.
<table id="custom" class="t-border">
  <caption style="text-align:center"><b>SMS security qualification</b></caption>
  <tr>
    <th>Use case</th>
    <th>Description</th>
    <th>Application</th>
  </tr>
  <tr>
    <td> <a href="#password-reset-urls">Password reset URLs</a> </td>
    <td>Account compromise by setting a new password</td>
    <td>CTI, OffSec, OSINT</td>
  </tr>
  <tr>
    <td><a href="#bypass-access-control">Bypass access control</a></td>
    <td>Account compromise by doing nothing</td>
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

###### Password reset URLs
This use case is very simple and common: the message received contains a password reset URL as shown below.
> <i>You can reset your password by clicking on this link: [https://pprod.mycompany.com/account?reset-token=RANDOM_GUID](https://pprod.mycompany.com/account?reset-token=RANDOM_GUID)</i>

###### Bypass access control
This scenario is even simpler, and simply gives access as an authenticated user when clicking on the URL. For example:
> <i>Click here to access your account : [https://shrtn.com/0123456789abcdefgh](https://shrtn.com/0123456789abcdefgh)</i>

###### Data exposure
While this scenario is not as critical as the previous ones, it gives the opportuniy to collect various types of data, base on the concerned application business.
> <i>Click here to check your analysis results : [https://shrtn.com/0123456789abcdefgh](https://shrtn.com/0123456789abcdefgh)</i>

###### Data discovery
This scenario is very rare and not very sensitive. The only example is the enumeration of subdomains, where the URL is not accessible <i>i.e.</i> does not lead to account compromise from the public Internet. For example :
> <i>Hello admin, click on the URL to reset your password : [https://internal.and.sensitive.domain.com/0123456789abcdefgh](https://internal.and.sensitive.domain.com/0123456789abcdefgh)</i>

###### Missed opportunities
Missed opportunities are almost exclusively use cases where the whole URL can't be read, because it is too long. For example :
> <i>Hi, how are you? Did you ask for a password reset? Here is your URL, don't share it! [https://superlongandweirddomainnamerightquestionmark.com/?reset=A](https://superlongandweirddomainnamerightquestionmark.com/?reset=A) </i>

###### Scams
Among all messages, many are scams that look like this.
> <i>Your shipment could not be delivered because the transport fee was not paid. Click here to pay.</i> 

### Statistics
Statictics must be added.


### Real world examples
<table id="custom" class="t-border">
<caption style="text-align:center"><b>Resource deployed</b></caption>
  <tr>
    <th>Target</th>
    <th>#</th>
    <th>Password reset URLs</th>
    <th>Bypass access control</th>
    <th>Data exposure</th>
    <th>Data discovery</th>
    <th>Missed opportunities</th>
  </tr>
  <tr>
    <td><a href="#venmo">Venmo</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#instagram">Instagram</a></td>
    <td style="text-align:center">1200/month</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#biedronka">Biedronka</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#job-logic">Job logic</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#free-ads">Free ads</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#payfone">Payfone</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#airindia">AirIndia</a></td>
    <td style="text-align:center">1500/month</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#texts-from-my-ex">Texts from my ex</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#validahealth">Validahealth</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#stickermule">Stickermule</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#wallester">Wallester</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="bankinter">Bankinter</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#abinitio-solutions">Abinitio Solutions</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#communidad-madrid">Communidad Madrid</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#ups-scam">UPS Scam</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#experian">Experian</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#booksy">cdl.booksy.com</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  <tr>
    <td><a href="#elilly">e.lilly</a></td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">✅</td>
    <td style="text-align:center">❌</td>
    <td style="text-align:center">❌</td>
  </tr>
  
</table>

***

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_1.gif" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

***

<!-- VENMO -->
##### Venmo
According to Wikipedia, Venmo is an american company owned by PayPal with approximately 70 millions users in 2021. This is some sort of banking application that allows users to send payment to their friends and to manage their business revenues, which makes it a sensitive business application. There is only one message pattern sent from Venmo to its users, and it looks like this.
> <i>Reset your Venmo password: [https://venmo.com/account/password-new?utm_medium=phone&reset_key=40b11eeecc001f5a6d2cc4f40eae408c7bc953f11478eb028032c4f86](https://venmo.com/account/password-new?utm_medium=phone&reset_key=40b11eeecc001f5a6d2cc4f40eae408c7bc953f11478eb028032c4f86)</i>

When navigating to the URL, no other information than just setting a password. In this case, an attacker could reset the password and then what?
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_venmo_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Impact
Well, as you can see on the picture, the phone number can be used as the identifier to login into the application.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_venmo_2.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

This scenario typically involves a <b>race condition</b> for the attacker to compromise the account, because the link may expire, and because the legitimate user may reset their password before the attacker does.
Also, when the link expires, it is still possible to navigate to the URL and set up a password. Obviously this won't work, but it is a waste of time from an attacker perspective.



###### Responsible disclosure
02/05/2024 - Issue reported.

<a href="#real-world-examples">Back to the table ⬆️</a>

***

<!-- INSTAGRAM -->
##### Instagram
Instagram is very similar to <a href="#venmo">Venmo</a>, which seems very exciting at first glance. I detected this simply because there were a lot of Instagram shorten URLs <i>e.g.</i> [https://ig.me/0123456789abcdef](https://ig.me/0123456789abcdef). This kind of URL is not necessarily sensitive and may for instance redirect to a simple public picture hosted on [https://instagram.com/](https://instagram.com/). You can use a simple Google dork like <code>site:ig.me</code> to get shorten URLs and get redirected to any kind of Instagram content.

About Instagram SMS collected, navigating to those URLs would redirect to an Instagram error page displaying something like <i>The URL has expired</i>. I needed to dig deeper, created an Instagram account, and triggered a password reset procedure using a temporary phone number. Then, I was able to confirm that these URLs would allow an opportunistic account compromise. The text messages received look like these.
- <code>Tap to reset your Instagram password: <a href="https://ig.me/1ZZOeHuLUwa4Ttq">https://ig.me/1ZZOeHuLUwa4Ttq</a></code>
- <code>Instagram link: <a href="https://ig.me/r6LPOYK87HRyT73">https://ig.me/r6LPOYK87HRyT73</a>. Don't share it.</code>

<br>

When navigating to those URLs, and based on the message pattern received, we are redirected to different web pages.
###### Redirection - First case
In the first case, we can just skip the password reset and be logged in as the user <i>i.e.</i> the URL gives direct access to the account. I actually rarely saw this behavior and I have no screenshot for now.

###### Redirection - Second case
In the second case, we must complete the password reset procedure, and then authenticate to the account with the phone number and password just set.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_instagram_2.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Redirection - Third case
In the third case, the account is locked, but we can still retrieve the username, either by analysing source code or by clicking on the top right menu icon. This may be used for OSINT and CTI.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_instagram_3.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_instagram_4.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Redirection - Fourth case
In the fourth case, the URL has expired or was used.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_instagram_5.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<br>

###### Impact
A common intuition would be that one Instagram account is associated with one phone number. Thus, if a Public SMS Service provides a hundred phone numbers, an attacker could only compromise a maximum of a hundred user accounts. Actually, this is wrong. I observed that for a given phone number, multiple Instagram password reset URLs were received, leading to different user accounts. This seemed irrelevant to me. After some testing, I observed that it is <b>not possible to register</b> a new Instagram account with a phone number <b>when the phone number is already used</b> by an existing account. However, it is possible to <b>edit</b> the contact information to add a phone number.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_instagram_6.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

Imagine that Instagram has a user <code>jdoe@email.com</code> registered with <code>+4591457271</code>. If this user triggers a password reset, a text message will be received on the Public SMS Service hosting this phone number, and the account could be stolen. Now, imagine that a new user creates a new account with an email address <code>leoagata@proton.me</code>, and then edit their contact information and set their phone number to <code>+4591457271</code>. The immediate consequence is that this phone number will be removed from <code>jdoe@email.com</code> account. Now, if <code>leoagata@proton.me</code> triggers a password reset, a text message will be received on the Public SMS Service hosting this phone number. This means that we can receive multiple passwords reset URLs from different user accounts on the same phone number. The following example shows two SMSs received in a 10 min timeframe, with two different URLs leading to two different user accounts, received on the same phone number.

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_instagram_7.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

In the end, it is not possible to create an Instagram account with those virtual phone number, because all of them are already used. This makes Instagram account compromise possible in a very specific case where a user edits its information to provide one of those virtual phone numbers, then triggers a password reset. This would probably lead to testing accounts compromise only.

###### Responsible disclosure
When reporting this behavior to Instagram, I was told that Instagram won't do anything about those phone numbers, although it would be very simple to implement a blacklist feature in my opinion.

<a href="#real-world-examples">Back to the table ⬆️</a>

***

##### Biedronka
According to Wikipedia, Biedronka is the largest chain of discount shops in Poland, with 70,000 employees and a nice 11B€ revenue in 2019. Messages received from Biedronka look like this.
- <code>Link do zmiany hasła https://bdr.page.link/S8CpNMGEgdUjL41B6 ważny 5 minut</code>

Yes this is Polish, and it means <code>Link to change password https://bdr.page.link/S8CpNMGEgdUjL41B6 valid 5 minutes</code>. If you're not familiar with <code>page.link</code>, this is a Google Firebase feature that was designed a few years ago to set up easy to share URLs that handle redirections (see [Google documentation](https://firebase.google.com/docs/dynamic-links/create-manually) for more details).

Anyways, navigating to the URL would lead this page, inviting user to reset their password, and of course we are able to login with the phone number and password we just set.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_biedronka_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

Another way to compromise a user account is simply to download the mobile app, and enter the targeted public virtual phone number. Then a message is received on the Public SMS Service hosting that virtual phone number, providing a code. Based on the phone number country <i>e.g.</i> German phone number, the SMS showned on the picture may be cropped, and the last digit must be "bruteforced", which is possible since the number of tries seems to be unlimited.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_biedronka_2.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_biedronka_3.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

Finally, I took a look at the application and analysed its HTTP traffic to see if the 6 digit code could be bruteforced, and luckily it can't be bruteforced in a reliable.

###### Impact
When connecting to a user account, there is not much to do. Of course it discloses email address when fullfilled, and maybe credit card information since Biedronka works with a partner called BLIK, but I didn't investigate this.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_biedronka_4.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Responsible disclosure
TODO

<a href="#real-world-examples">Back to the table ⬆️</a>

***

##### Job Logic
JobLogic is an Enterprise Resource Management SaaS application that provides many different services such as payment integration, jobs management, planning, and so on. Here is the message received : “Your Job has been completed Please follow the link for details https://staging-portal.joblogic.com/Portal?id=GUID.”

Navigating to this URL directly provides access to a portal, authenticated as a user. As you noticed, this is a staging environment, which means that no real data should be found.

###### Impact

###### Responsible disclosure
12/04/2023 - Issue reported
<a href="#real-world-examples">Back to the table ⬆️</a>

***

##### Free ads
Free Ads is a UK application that allows user to publish ads to sell whatever they want, for free. The service proposes a paid version for users who want their ad to have more visibility. The message received looks like this:
- <code>Hi REDACTED, you have a new message, click to go to your messages >> https://go.freeads.co.uk/REDACTED.</code>

This URL redirects to https://my.freeads.co.uk/messages/list/all?c=REDACTED%3D%3D&utm_source=message_reminder&utm_medium=sms&utm_campaign=message and sets a cookie session that allows to be authenticated as a user. Unlike most of these URLs, this one does not expires!
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_freeads_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Impact
This URL gives full access to the user account, and enables an attacker to read/write personal information. As far as I can tell, credit card information is not stored by FreeAds since a premium account is a one time payment.

###### Responsible disclosure
12/04/2023 - Issue reported

<a href="#real-world-examples">Back to the table ⬆️</a>

***

##### Free ads
###### Impact
###### Responsible disclosure
TODO

<a href="#real-world-examples">Back to the table ⬆️</a>

***

##### AirIndia
According to Wikipedia, AirIndia is the second-largest airline in India in terms of passengers carried, after IndiGo, as of July 2023. The text message received for Air India was the following.
- <code>Thank you for flying with us. Please take a few moments to share your thoughts. To rate your experience, click: https://nps.airindia.in/rEDaCt3d -Air</code>.

When navigating to the URL, we are redirected to Qualtrics, a SaaS solution to create various things, including surveys. In order for me to understand how it works, I created a test survey available [here](https://qualtricsxmfx7yt665q.qualtrics.com/jfe/form/SV_57KnpcbkNjHQ6uq) that reads a URL parameter <b>Name</b> and display its value in the first question. Example with a survey for [Molx](https://qualtricsxmfx7yt665q.qualtrics.com/jfe/form/SV_57KnpcbkNjHQ6uq?Name=Molx), or for [Bender](https://qualtricsxmfx7yt665q.qualtrics.com/jfe/form/SV_57KnpcbkNjHQ6uq?Name=Bender).
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_airindia_0.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

With AirIndia, the issue is that all surveys include very sensitive data, including PII and also the PNR unique identifier. Or course I double checked this data using services like [https://flightstats.com/](https://flightstats.com/), and all of it was real!
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_airindia_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

The following table shows all information disclosed by AirIndia as URL parameters.
<table id="custom" class="t-border">
  <caption style="text-align:center"><b>AirIndia disclosed data</b></caption>
  <tr>
    <th>Field</th>
    <th>Field description</th>
    <th>Raw value</th>
    <th>Value meaning</th>
  </tr>
  <tr>
    <td>PNR</td>
    <td>Reservation unique identifier</td>
    <td>REDACTED</td>
    <td>REDACTED</td>
  </tr>
  <tr>
    <td>ORG</td>
    <td>Origin (IATA code)</td>
    <td>DEL</td>
    <td>Indira Gandhi International Airport (DEL)</td>
  </tr>
  <tr>
    <td>DES</td>
    <td>Destination (IATA code)</td>
    <td>YVR</td>
    <td>Vancouver International Airport (YVR)</td>
  </tr>
  <tr>
    <td>PFN</td>
    <td>First name</td>
    <td>REDACTED</td>
    <td>REDACTED</td>
  </tr>
  <tr>
    <td>PLN</td>
    <td>Last name</td>
    <td>REDACTED</td>
    <td>REDACTED</td>
  </tr>
  <tr>
    <td>PEmail</td>
    <td>Email</td>
    <td>REDACTED</td>
    <td>REDACTED</td>
  </tr>
  <tr>
    <td>PPhone</td>
    <td>Phone</td>
    <td>447487710863</td>
    <td>447487710863</td>
  </tr>
  <tr>
    <td>FlightNumber</td>
    <td>Flight number</td>
    <td>REDACTED</td>
    <td>REDACTED</td>
  </tr>
  <tr>
    <td>FlightDate</td>
    <td>Flight date</td>
    <td>02%20Feb%2024</td>
    <td>02 Feb 24</td>
  </tr>
  <tr>
    <td>Class</td>
    <td>Class</td>
    <td>Q</td>
    <td>Q</td>
  </tr>
  <tr>
    <td>Environment</td>
    <td>Environment</td>
    <td>Production</td>
    <td>Production</td>
  </tr>
</table>

If you ever travelled by plane, you may understand what an attacker could do right? Here is the answer.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_airindia_2.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Impact
So an attacker can connect to the check-in web page, but what does this mean?
- <b>Situation 1</b> - If the flight is in a few weeks or month, travellers are not allowed to check-in. In this situation, an attacker can't do anything except get the list of flights related to the PNR.
- <b>Situation 2</b> - If the traveller can check-in, well, an attacker could check-in.
- <b>Situation 3</b> - If the traveller already checked-in, an attacker would have access to a boarding pass that provides additional information such as a Passport number or a VISA number, and can still modify information on the boarding pass.

A curious thing is that the survey is meant to get get a feedback about reservation, check-in, in-flight meal, arrival and exit, but it seems to be sent immediately after tickets are bought <i>i.e.</i> before the flight date.

###### Responsible disclosure
15/04/2023 - Issue reported

<a href="#real-world-examples">Back to the table ⬆️</a>

***

##### Texts from my ex
A funny name right? The concept is also funny : this is a SaaS app where users upload a part of their conversation (WhatsApp, Messenger and so on) with their crush or ex, and an AI evaluates how compatible the two persons are. Unfortunately, I got rid of the message, so I can't tell how it looks like, but I have a URL example :
- <code>https://textsfrommyex.com/analysis/65bdREDACTED3655</code>

###### Impact
The impact entirely relies on the messages content. In the following example, the only sensitive infomation are first and last names. However, since this 'report' quotes the conversation, any sensitive information in that conversation could be present.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_textsfrommyex_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Responsible disclosure
Not reported.

<a href="#real-world-examples">Back to the table ⬆️</a>

***

##### Stickermule
Stickermule is a website where sticker can be bought. The message received looks like this.
- <code>Items from Sticker Mule order R236623233 shipped! http://stickermule.com/orders/R236623233?token=REDACTED</code>

###### Impact
When navigating to the URL, an attacker can access shipment information <i>i.e.</i> first name, last name and physical address and the list of stickers bought. Thus, the user account is not compromised.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_stickermule_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Responsible disclosure
01/05/2023 - Issue reported

<a href="#real-world-examples">Back to the table ⬆️</a>

***

##### Abinitio solutions
In the context of the received message, Abinitio solutions is a subcontractor of the Spanish Naturgy Iberia energy provider. In the following message, the user is required to pay their bills.
- <code>ABINITIO INFORMA REDACTED Pulse https://abinitio-solutions.com/n.php?p=RUIzNlREDACTED para el abono de sus facturas con NATURGY IBERIA</code>

When navigating to the URL we have a bill detail with the customer personal information <i>i.e.</i> first name, last name and address.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_abinitio_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

When navigating to the provided URL, we are redirected to a payment page.
<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_abinitio_2.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

<div class="col-sm mt-3 mt-md-0">
    {% include blog/figure.html path="assets/img/publicsms_results_abinitio_3.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

###### Impact
PII disclosure.

###### Responsible disclosure
TODO

<a href="#real-world-examples">Back to the table ⬆️</a>

***


### 3.2.2. Missed opportunities