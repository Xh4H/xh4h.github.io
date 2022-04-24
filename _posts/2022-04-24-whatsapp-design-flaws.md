---
title: Various WhatsApp design flaws
description: WhatsApp design flaws
categories: web whatsapp
author: Xh4H
tags: web whatsapp design security flaws ip phishing
---

# Introduction
Welcome to this post. Long ago I discovered some design flaws in WhatsApp. Expecting these to be fixed with the time, I never gave it much importance.

But, the other day, I showed my discoveries to a close friend of mine, [S4vitar](https://twitter.com/s4vitar), and he suggested I should write about this, so here we are!

Initially, I want to declare this investigation has been done with educational purposes, and discoveries showcased here were sent to WhatsApp's team's attention (with no response whatsoever).

## Findings
### 1. IP Disclosure
#### Problem
Everything started when I was casually checking the access log of my website.

In these logs, there were some lines that included an interesting UserAgent: `WhatsApp/X.X.X.X`, where the X's specify a version. I was curious where they were coming, was WhatsApp scanning my website? No, not really.

I found out the following: every time you typed a URL while forming your message, WhatsApp would perform GET requests to that URL as to display, if available, thumbnails, titles, descriptions...

Have a look at the following image to understand what I am talking about.

<div style="text-align:center"><img src="/assets/images/whatsapp1.png"/></div>

Usual behaviour in messaging applications. The problem is that the implementation is wrong. Every time WhatsApp performs these requests, they are done using the actual client, exposing the IP of the user:

<div style="text-align:center"><img src="/assets/images/whatsapp2.png"/></div>

#### Suggested solution
The fix is rather simple, these requests should be performed from WhatsApp's servers, they should act as a proxy to request and deliver these kind of requests. Having a look at other messaging apps, like Telegram and Discord, they indeed follow this approach.

### 2. Phishing is a piece of cake
#### Problem
After investigating how WhatsApp Web sent messages, I found out we could modify the message object that was being used by this application before it was sent to the server.
WhatsApp Web is constantly being updated so the place to find the following information might be different in the future.

I used Chrome Devtools to search for `sendmessage`, and found the following:

<div style="text-align:center"><img src="/assets/images/whatsapp3.png"/></div>

After beautifying some of these files and reading through the code, I found that in the line 996 of one of the bootstrap_main js files, there was a call to `sendMsgRecord(z)` where `z` is the message object.

I decided to set a breakpoint there and start debugging. It turns out that the message object contains so many properties, such as sender details, recipient details, message content, and a very long list of other properties.

The bad thing comes here, if you are sending a valid URL, the message object will contain the following:

<div style="text-align:center"><img src="/assets/images/whatsapp4.png"/></div>
<div style="text-align:center"><img src="/assets/images/whatsapp5.png"/></div>

Just by changing manually those properties to another domain, the message object will still be accepted and both the sender and recipient will see a modified preview:

These are the previews prior to and post modification:
<div style="text-align:center"><img src="/assets/images/whatsapp6.png"/></div>

If somebody were to perform a phishing attack, they would be able to modify the message object to send a link with a fake website preview. The recipient could just check it and click on the URL without having to read the actual message.

In prior versions of WhatsApp, it was also possible to modify the image of the preview.
<div style="text-align:center"><img src="/assets/images/whatsapp7.png"/></div>

#### Suggested solution
The client of the recipient should always fetch the preview of the URL received and not trusting what the sender has sent. Simple as that.

## Conclusion
WhatsApp, being the most popular messaging app, it still has various architectural misdesigns, from which I have only talked about two of them. I reported these issues to WhatsApp's team 2 years ago and no response was received from their side. I hope this shines some light on this.

## Thank you
Thanks to [@S4vitar](https://twitter.com/S4vitar) for allowing me to test with him.


Thanks for reading :)

<script src="https://www.hackthebox.eu/badge/21439"></script>
