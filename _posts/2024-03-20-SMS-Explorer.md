---
layout: post
title: SMS Explorer - An misknown blah
date: 2025-03-21 10:00:00
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
- [1 - Introduction](#introduction)
- [2 - Automation](#automation)
- [3 - Results](#results)

***

## Introduction
### What are Public SMS Services?
While browsing the Internet, you probably searched for a temporary email address. For example, you created a user account on some web app, and you didn’t want to provide real information. Alternatively, you may need a temporary phone number if the app requires it: in this case, you probably used a SMS Service.

It may be simple to understand, but let's give a proper definition to what I mean by SMS Service : "An online service that provides virtual phone numbers to their customers in order to receive SMSs.". Of course, such service providers may propose additional services such as Voice Over IP (VoIP), but this is out of our scope.

There are plenty of such services out there. Using any simple online research such as <b>receive sms online</b> using your favorite search engine will lead you to multiple websites. By taking a look at the results, we can quickly notice something : some services need you to register or pay, others directly provide phone numbers.
- <b>Paid services</b> – These are services that allows customers to rent a phone number.
- <b>Registered services </b> – These are services where users are not forced to pay to access the service, but that have limited features.
- <b>Free services</b> – These are the services that are fully accessible without any form of authentication. It is interesting to note that this kind of service has a tremendous amount of trackers and ads that can even lead the web browser to freeze.


As you may think, paid and registered services provide access control, so that received SMSs can’t be seen by other users: we will call these services Private SMS Service. Similarly, since free service allow any user to read any message of any phone number, we will call these services Public SMS Services (PSS), and this is what I will discuss. Among all these PSSs, I tried to focus on [https://receive-smss.com/](https://receive-smss.com/), for multiple reasons that will be detailed in the automation section.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_0.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>


### Give it a try!
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

### A few insights
In order to get some information about the interest in PSSs, we can use any trend analysis tool. Here are my results for keywords free sms online on [https://semrush.com](https://semrush.com). We can see in the bottom right corner that these keywords belong to a cluster where users seem to look for test or verification phone number. This already give insight about data we will find.

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_intro_4.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

Additionally, we can take a specific domain to analyze its traffic and see how it is used, which may be useful in order to select a high traffic PSSs for analysis.

### About the data
Legitimately, you may wonder why would anyone spend their time reading strangers SMS. Well, let me give an example: while refreshing a PSS “received messages” web page to see the message I was waiting for, I looked at already received messages, and one of them was something like the following: “Hi <username>, your new access code is <code>. You can now connect on <website>”.

Since I couldn’t believe it, I quickly configured an anonymization pipeline and accessed the account. The website was a cryptocurrency application that allowed users to access their different wallets. I had access to personal information and to a significative amount of 65 Bitcoins.

Besides the message received represented itself a flaw, I had three hypotheses in mind:
-	<i>This is a human mistake</i> – The person owning the account used an Open SMS service ignoring what the consequences could be.
-	<i>This is a development environment</i> – In this case, the account is not a real one, or at least this is a test account, with mocking data.
-	<i>This is some kind of aggressive honeypot</i> – Since the message is received on an Open SMS service, and that it leaks the three key information needed to access the account (website, username and password), it seemed too obvious not to be a honeypot.
In the end, the domain was created three months ago, so it probably was a development environment.

Before digging more into the data that can be found, I’ll quickly discuss the automation process.


### State of the art?
As far as I know, there isn’t any research about this topic. The only source I could find is from the Canadian ZATAZ that [quickly said a word about PSSPs](https://www.zataz.com/osint-comment-lire-les-sms-dinconnus/).


## Automation
### Why and how?
Before explaining anything, you may wonder why automation is needed? Because of the amount of SMSs received, our primary need is simply to collect data to duplicate the database, and develop a searching feature that would allow to identify interesting patterns. Additionally, we want our database to grow as we gather more and more SMSs, which is usually not done by PSSP as only the last 40 SMSs are displayed, at least for https://receive-smss.com/.

#### Version 0.1
I developed a first version of the program which was very basic. It parses https://receive-smss.com/ and stores any unique SMS identified. After a few hours of automated SMS collection, I was excited to analyze the data, and a quickly figured out that there were two interesting cases.
-	The first one is when messages directly contain sensitive data, like credentials or PII. However, this is a very rare case, and it is complex to automate without false positive, and I didn’t want my program to generate false positive. 
-	The second one is when messages contain URLs. This is a much more frequent case, and it represents 99% of interesting messages at this point. Why? Because anything can hide behind a URL, a scam or an account takeover.
I abandoned the first case for now, and I focused on messages that contain URLs


#### Version 0.2
To make the analysis a bit simpler, I developed a web interface where I can look for messages collected, and easily navigate to URLs. Thanks to this tiny improvement, I built a list of all URLs and I manually navigated to each one of them, and for some of them simply gave me access to user accounts. At this point, I decided to develop the last core feature: the aggressive mode. This can be turned on to navigate to URLs known to hide interesting data, then it will add this data to the database!

However, I faced a challenge that is still not resolved: race conditions. Some URLs, such as password reset URLs, have either an expiration date, or can be accessed only once to prevent someone else to reset a password a second time. I decided not to focus on that problem for now, because there are a lot of URLs to explore.

#### Version 1.0
Now the architecture is relatively stable, I can describe it. By the way, this is entirely developed in Python and containerized in Docker. There are multiple containers
-	Flask – This is the web server.
-	RedisQueue – This is a queue, were two jobs are enqueued
o	The fetcher job which will parse https://receive-smss.com/
o	The data job which will look for URLs in the database and handle them to retrieve data.
-	Workers – Since we have two jobs that never end, we need two workers to execute them, one per job.
-	PostgreSQL Database – Well, this is a database.
Once Flask is up, all workers are enqueued and executed. The user interface is made to do basic stuff such as cleaning the database, searching data, and so on.


<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/publicsms_automation_1.png" class="img-fluid rounded z-depth-1 imgc" style="display: block;margin: auto;" %}
</div>

### Improvements


## Results