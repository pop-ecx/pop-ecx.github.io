---
title: "Security Slips: The Day I Ran Unknown Code"
description: "I almost got got"
date: 2025-03-28T13:17:54+03:00
tags: ["deobfuscation", "security"]
categories: ["deobfuscation", "Cybersecurity"]
keywords: ["code deobfuscation"]
draft: false
---
On March 25th 2025, Troy Hunt, [haveibeenpwned's](https://haveibeenpwned.com) author, published in his [blog](https://www.troyhunt.com/a-sneaky-phish-just-grabbed-my-mailchimp-mailing-list/) about
how a sneaky phishing lure got hold of his mailing list. This interesting read made me 
reminisce about how, not so long ago, I almost fell for a similar thing, albeit mine was
not so sophisticated.

January 2nd 2025, 12 p.m EAT, I had just gotten home from a lengthy holiday. Being super
tired I booted up my Latitude to check on a few things. One of the first things I saw was a 
github link sent in one of the many cybersecurity pages I follow. It was allegedly a tool
to check on some IoT stuff. I quickly clicked on the link to the github page, git cloned
the code and ran it (wasn't even thinking of one of my many VMs). 

While running the program, It requested for some input (country code). That's where my 
paranoid personality woke up and screamed. I quickly ctrl + c'd. I had not even checked the
code I was running. I opened the cloned directory. There were only 2 python files,
the main file and like a module file to be imported. "Ohh, that's easy. Let me just open it",
I thought to myself. To my shock, the main file was just a single, lengthy line of code, and on
top of that it was obfuscated. Yeah, not a good start. "Maybe suspicious", I said to myself.
I spent the next couple of minutes cleaning it up and writing a deobfuscation script. I wasn't 
happy with myself. How could I be so dumb? I finally had the actual source code and understood pretty
well what it does. Well, not everything. I still did not understand the import. 

Luckily for me, the obfuscated structure of the import code was similar to the one in main. My deobfuscation
script sufficed pretty well and within no time, I had the import code right in front of me.

A quick review showed the code did what it was advertized to do, no hidden shenanigans (at least that I know of).
I also ran my endpoint protection software just to be sure (I normally turn it off since I have some sketchy stuff.
Also, I'm my own antivirus (o≧▽≦)o).

The obfuscation threw me off into thinking I actually ran malware but then again I fail to understand why benign code that
is publicly available on github should be obfuscated. Sometimes obfuscated code is benign code.

Let that be a lesson, the only person who cannot fall for a hack is he who does not use technology. Return to monke :D
