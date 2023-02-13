+++
title = "Security Telemetry Linux"
date = "2022-09-03T21:30:01+03:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
+++

<!--more-->
{{< image src="/img/forest/logo.png" alt="" position="center" style="border-radius: 8px;" >}}

## Introduction

In this blog post, I'm looking at a topic that I've been curious about for some time now. This is security telemetry on endpoints. Security telemetry here refers to security related data you can collect on endpoints to aid in security monitoring. This data would be mainly from logs generated. Endpoints here include workstations and servers. I am focusing on linux for this post.

I want to look at configurations on linux that enable for generation of different logs to help in security monitoring. I hope at the end of this post we will have a fairly good idea of how we can increase visibility on activities on a linux endpoint.

## Authentication

In security monitoring, authentication logs is important. They tell you who logged on to the system, at what time and how they logged on. These information is also useful for identifying credential attacks such as bruteforce attacks so as to respond to them.

For system administration on linux servers, we typically connect via the ssh protocol remotely. 
