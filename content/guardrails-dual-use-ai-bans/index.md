---
title: "Guardrails, Dual Use, and the Ethics of AI Bans"
date: 2026-08-10T09:03:04+03:00
tags: ["AI", "Guardrails", "Dual use", "Ethics"]
categories: ["AI", "Ethics"]
keywords: ["AI", "Dual use", "Ethics", "Guardrails"]
draft: false
---
> "You betrayed my trust. From the beginning, you intended to misuse this knowledge for evil purposes."[^1]

A few days ago, my OpenAI account got deactivated[^2].
Reason: Cyber abuse. "Embrace AI or you'll be left behind," they said.
Now I have been banished from the [technofeudalist](https://simple.wikipedia.org/wiki/Technofeudalism)
network state.

While it may be incredibly easy to be salty here, it does make sense to some
extent why corporate bans are, at times, a necessity.

<!--more-->

> **Scope note:** This post is independent of whatever opinions I have on the impact of AI environmentally, socially, economically, neurologically, and its effects on open source.

### Steelmanning the concept of bans
I do understand the desire to curb misuse and abuse of a corporation's product.
In the case of AI companies, I may perhaps even sympathize with it. For one,
I'd like to believe they seriously care about preventing harm. But even if they
don't, it would still be in their best interest to prevent this as much as 
possible, both from a revenue or even optics standpoint. No one wants to use an
"unsafe" product. Pursuing such actions helps give these companies legitimacy,
not only to lawmakers and governments but also the public at large.

### Dual use
"Policing knowledge" has always felt wrong. That's why most of us today look down
on those eras where book burnings were a thing. Almost every piece of knowledge
you can think of can be misused. Biotechnology: bioweapons. Nuclear energy:
nuclear weapons and proliferation. Aerospace engineering and propulsion: ballistic missiles.
Cybersecurity: cybercrime. There is no denying the above-mentioned disciplines
can be used for good as well, but how do you actually determine that?
It's incredibly hard to infer intent based on a prompt. Knowledge is always double-edged.

### Middle ground?
Some AI companies try to get around this. Anthropic has [CVP](https://support.claude.com/en/articles/14604842-real-time-cyber-safeguards-on-claude-opus-and-sonnet)
and OpenAI has [TAC](https://openai.com/index/trusted-access-for-cyber/). These programmes lift **some** of the guardrails (I know for cybersecurity),
just as long as you apply to get in. The problem with these sorts of programmes
is not everything is permitted even if you get in. So the AI may waste tokens
trying to infer your intent, and just block your request anyway. Now you end up
wasting most of your time trying to get around this instead of accomplishing your goal(which may be a legitimate task). 

So now the question becomes, "Are Cybercriminals actually hindered by this?". 
I don't know the answer. I, however, suspect it's no. What's stopping someone
who has been banned from re-registering with different creds? Security professionals
ostensibly have much to lose when this happens as compared to cybercriminals.
The literature on whether these protections and bans are effective in their overall goals is inconclusive.

Cybersecurity (offensive) to me has always felt like it's about who gets to the
finish line first or at the very least who gets closest to it first.

### King of the Hill
[TryHackMe](https://tryhackme.com/) has an interesting game called [king of the hill](https://tryhackme.com/resources/blog/guide-to-king-of-the-hill).
The idea of this game is simple; race other ~~9~~ 10 hackers to compromise a system.
Get in, patch so the others do not get in, and maintain persistence while looking
for other flags hidden inside. This, at the core, is what offensive security really 
is about in the real world. Same idea, same concept; a race between the "good guys"
and "bad guys" to unearth vulnerabilities. The group that gets there first matters.
It is in the best interest of everyone that the good guys win this race.
And I doubt things like [project glasswing](https://www.anthropic.com/glasswing)
fully help with that. Personally as a security professional, it feels the easiest
way to increase our odds of winning is open weights.

### Open weights as a potential solution?
**Open weights** is one of the few things that excites me about AI and LLMs.
It's the idea that a model's parameters (weights) are published for use and modification
to use-cases. This is exciting because you get to be in control of the tool you use.
You study it, tweak it to your own liking, and just go. Technology should always
serve the user. I believe this could have a very solid ROI[^3], and generally
good for cybersecurity. Of course, I could be all wrong, the sun explodes tomorrow and none of this would matter.

### Footnotes
[^1]: An ATLA reference. That's what [Wan Shi Tong](https://villains.fandom.com/wiki/Wan_Shi_Tong) says when he ascertains what the Avatar and his team intend to do with the knowledge they found in his library. He goes on to bury it underground in an effort to ban humans from ever accessing it. `¯\_(ツ)_/¯`.
[^2]: Haven't contemplated an appeal. Don't know if I will.
[^3]: [Open weights and American AI leadership](https://www.microsoft.com/en-us/corporate-responsibility/topics/open-weight/)
