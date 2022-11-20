---
title: "Help An Incident Responder Out - Enable logging"
date: 2022-10-29T17:12:12+02:00
draft: true
toc: false
images:
tags:
  - DFIR
  - forensics
  - sysmon
---

One of the issues that I, as an incident responder, often see and have to deal with is the lack of evidence or logs when investigating an incident.

We often see recommendation arguing that to fix this issue, organisations needs to implement log collection systems or SIEM systems, so they can start alerting on issues, and query across multiple PCs. That is a really good recommendation and I still recommend this today, but this often takes time, money and a lot of tuning to get a working system, and also needs to have people actually looking at the alerts it creates. Things that many system administrators does not have. So how can we better the situation, and possibly give the administrators and organisation some help to get started and give responders more information when ever an incident occure? I good place to start is Sysmon.

Sysmon is a tool developed by Sysinternals and enables logging on a much more detailed level than what comes out of the box on a Windows system. And in the newest version, it also allows to block execution of programs, which could help block the most common attack (ie. the default Mimikatz executable).
