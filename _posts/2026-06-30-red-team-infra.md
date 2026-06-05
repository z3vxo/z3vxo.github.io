---
layout: post
title: "Modern Red Teamind: infrastructure"
date: 2026-10-23
categories:
  - maldev
  - cybersec
---
<style> body { background: #1e1e1e; color: #e0e0e0; } </style>


# Intro
Infrastructure makes or breaks an engagement, you could have the most evasive loader, the best phishing pretext but none of that matters if your beacon calls out to a 2 day old domain hosted on an IP in a sketchy ASN and a single new firewall rules destorys the engagement before it even began 
in this blog post i will be going over the core concepts, why we design it the way we do, what each piece is and how to set it up


# The Primitive approach 
The most common mistake beginners make is standing up a team server on a publicly exposed VPS and calling it a day. Point your beacon at the IP, get a callback, job done right? Not quite.

The moment your team server is directly exposed, you’ve handed defenders everything they need. One analyst spots the beacon traffic, one firewall rule gets pushed, and the engagement is over. Your C2 is burned, and with it every beacon calling home to that IP.

There’s no separation, no fallback, no resilience. It’s a single point of failure sitting naked on the internet.

an example of this is below

![Primitive Architecture](/assets/images/IMG_2077.jpeg)



that single arrow between the compromised machine and team server can be severed and the campaign is over

so how do we ensure our infrastructure is reliable and a single block doesnt bring it all down?

# Redirectors
A redirector is a server that sits infront of the team server, all traffic to and from the c2 is proxied through this, this does 2 things

1. it ensures the target never directly interacts with the team server, they can block this server all they want but its created to be expendable, if it goes down we jusr spin up a new one, update dns ane continue with the engagement 

2. it allows us to control who can touch our team server, the redirector handles filtering bots, scanners and most importantly the defenders, if the request does not contain a specific header, URI or paramter  they get a 404, decoy website or a redirect. the blue team can probe all they want but they wont find anything

The team server should only ever allow direct connections from the redirector and hopefully should be inside a private network not publicly reachable

this can be done by having the the team server on prem with a ssh/vpn tunnel connecting them or both inside a private VPC with only the redirector being publically reachable  
now our architecture looks like so

![Upgraded Architecture](/assets/images/IMG_2078.jpeg)

but the redirector itself has a public IP, maybe its not a fully trusted IP or under a known abused ASN, it could still be blocked, mapped or used by the blue team to stop you, so how do yoi solve this?

# Abusing CDNs
<details>
<summary>NOTE</summary>
Not all engagements need to abuse a CDN, if your redirectors IP is clean or you dont have access to a CDN then its perfectly fine to not use this
</details>
Content delivery networks are services legitimately used by basically every modern website. Almost every large cloud provider offers one — Amazon has CloudFront, Cloudflare has their CDN, Google has Cloud CDN, Microsoft has Azure Front Door.

What makes these useful for us is that their domains carry incredibly high trust. You can’t easily block them — if an org tried, they’d take down half the internet for their own users in the process.

This gives us two things:

1. A trusted front for our redirector.
Instead of the beacon calling out directly to our redirector, it calls out to a CDN, which then forwards the request to the redirector. The defender sees traffic going to *.cloudfront.net or *.azurefd.net which is incredibly common, and not to anything we own.

2. Clean IPs.
The IPs the beacon is actually hitting belong to the CDN provider, sitting inside trusted cloud ranges. No sketchy ASN, no 2 day old VPS IP, just infrastructure that looks completely legitimate.

so now the architecture looks like this

![CDN Architecture](/assets/images/IMG_2079.jpeg)

# Short haul vs Long haul
In an engagement you should idealy have 2 separate c2 channels serving different purposes

short haul: this is where your day-to-day hands on keyboard activity happens, running commands, lateral movement, anything that requires semi-realtime interaction. It beacons frequantly and gets used heavily. Ideally HTTPs

long haul: this is your persistence layey. It stays quiet checking in in 30 minutes to a day, sometimes longer. Ideally a DNS beacon since it simply needs to send a heartbeat and allow us to regain access incase the short haul beacon gets burned. You should rarely touch it

The critical rule is: these should be completely seperate, you do not want your short haul infra to be burned and in turn have your long hual beacon lost, this means seperate domains, seperate redirectors and ideally seperate domain registrars and cloud providers, not a hard rule though

Assume your short haul beacon will be burned, have backup servers and domains on standby that can be spun up a moments notice, your long haul is your saftey net, treat it like one

The architecture now looks like
![Architecture](/assets/images/IMG_2084.jpeg)