---
title: Documentation, Automation and Ansible
date: "2022-01-16T19:00:00.000Z"
description: "Using Ansible playbooks as a knowledge base"
---

I've been religiously documenting every step I do when learning something new for a long time. I tend to forget things easily, especially when not really practicing them every day, so my personal defense has been to write down anything that might help me when coming back to the topic at hand many months (and sometimes years) later. A bit like a [Zetellkasten](https://en.wikipedia.org/wiki/Zettelkasten).

Lately, I've moved from simply taking notes for myself to actually automating whatever it is I'm working on. I've found that it makes me actually dig a bit deeper, remember and learn more effectively and be 100% thorough in my documentation.

Like many, I've started with Bash, since it's readily available and simple. However, I find that Bash codebases tend to grow out of control quite easily, becoming a mess of tangled, unreadable code. Since some months ago, I've started to play around with [Ansible](https://www.ansible.com) and have completely fallen in love with it. It's easy to begin with and grows with you as you need it to, helps keep the automation well structured and, if you follow the [best practices](https://docs.ansible.com/ansible/2.8/user_guide/playbooks_best_practices.html), will make all your automation scripts immutable by default (mandatory shoutout to [Jeff Geerling](https://www.jeffgeerling.com) and [it's book](https://www.ansiblefordevops.com) and [YouTube series](https://youtu.be/goclfp6a2IQ) on Ansible, from whom I've learned most of what I know).

[Here's an example](https://github.com/aveiga/k3s-knative-arm) of some of my Ansible playbooks, in this case deploying K3s and Knative on ARM.
