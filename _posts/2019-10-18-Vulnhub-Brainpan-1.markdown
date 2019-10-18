---
title:  "Vulnhub - Brainpan: 1"
date:   2019-10-18 12:15:00
categories: [Writeup]
tags: [Vulnhub, Brainpan, Buffer Overflow, OSCP]
---

**intro**
I am starting off this blog with a quick writeup on [Brainpan: 1][brainpan-link], a boot2root virtual machine that is an excellent introduction for anyone interested
in going down the challenging yet rewarding rabbithole that is exploit development.
It is also good practice for solidifying concepts taught in the Penetration Testing With Kali Linux course.

**Initial Discovery**
After spinning up the VM, we are greeted with the login screen for the box.
![](/images/brainpan-1/1.png)

The first step will be to find the IP address of the VM on our network. So we'll do a quick nmap -sn (No port scan) to find it.
(IMAGE_2)

Looks like we have an IP of (IP). Now lets move on to port scanning.

After another nmap scan we are presented with the following ports.

[brainpan-link]: https://www.vulnhub.com/entry/brainpan-1,51/
