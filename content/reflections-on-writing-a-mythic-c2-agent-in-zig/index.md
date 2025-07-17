---
title: "Reflections on Writing a Mythic C2 Agent in Zig"
date: 2025-07-14T09:38:04+03:00
tags: ["zig", "Offensive tooling", "C2 development"]
categories: ["zig", "Offensive tooling", "C2 development"]
keywords: ["zig", "Offensive tooling", "C2 development", "Mythic C2"]
draft: false
---

![avatar](./rango-2.png)


Over the past couple of months, I've been writing a mythic C2 implant in zig.
"Why?", you ask. Well, it's all part of my learning process. I also 
wanted to understand how C2 frameworks work and how zig can be used for offensive tooling. 

There are lots of C2 frameworks out there and each has its own quirks and uniqueness.
I happened to settle on Mythic. Mythic supports agents written in any language. This helps me
improve my zig skills. Plus, the documentation is great.

The implant is simple and minimal. It supports Mythic's HTTP profile. The agent itself
does not support encryption at the moment. Zig does not have AES-CBC as far as I can tell.
It does have other AES modes though like GCM and OCB. There are also endless 'CBC bad' warnings I came across.
Though it is possible to use external dependencies for AES-CBC, I chose not to because reasons.
I might just write my own implementation of the CBC mode. If I do, I'll probably talk about it here.

The agent is for educational purposes and is not opsec sound yet (that's why I have no qualms putting it in the public domain).


You can find [my zig agent](https://github.com/pop-ecx/rango.git) here.


Below are some super useful resources

- [Mythic docs](https://docs.mythic-c2.net/)
- [C-based Mythic agent tutorial](https://red-team-sncf.github.io/how-to-create-your-own-mythic-agent-in-c.html)
