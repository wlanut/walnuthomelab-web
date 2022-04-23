---
title: "Time Capsule"
date: 2022-04-11 05:58:27
# weight: 1
# aliases: ["/first"]
tags: ["homelab"]
categories: ["homelab"]
author: "Omar El-Sherif"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
#description: ""
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: false
ShowBreadCrumbs: true
ShowPostNavLinks: true
cover:
    image: "/img/2022/April/002-timecapsule/timecapsule_cover.png" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
editPost:
    URL: "https://github.com/wlanut/walnuthomelab-web/tree/master/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---



We're taking a short trip back to approximately 1 year ago when I made my first and only "post" documenting the state of my homelab at any given point in time. I'm leaving the entirety of what follows unmodified for posterity. Since then, I've relocated my homelab to a new residence, made a large number of changes to both the physical and virtual infrastructure, and learned many lessons along the way. That, however, will be for a future post.

![img1](/img/2022/April/002-timecapsule/homelabfirst.jpg)


## An evolving system
I receive the internet through my ISP via fiber from the ONT, which goes to my ISP’s gateway. From there it goes to an unmanaged Netgear switch, and is distributed to several ethernet wall receptacles all over the house.

From the receptacle in my office, a Cisco 24 port switch (C2960G) handles the ethernet connections to all the other devices in the room (2 consoles, 2 computers, a laptop, etc).

Connected to the switch are my home server and my gaming rig. The rig is a Windows 10 Pro machine with an i7-6700k, 32gb RAM, and a GTX 1070. The home server is a 4U Rosewill chassis and houses a Xeon E3-1270V2, 16gb RAM, one 4TB Seagate Ironwolf, and one 480GB PNY SSD for the hypervisor.

I’m running Hyper-V Server 2019, which is the free headless version. I manage it through my gaming rig using a combination of Powershell sessions, the Hyper-V Manager, and Windows Admin Center, all very effective tools. I use Powershell when I’ve got the time and want to learn how to do my next task through the terminal, and WAC when I just need to make a quick filesystem change.

I’m currently running a few VMs, an Ubuntu one for all my media services, Ubuntu Server for Nextcloud, and a Windows Server 2019 VM set up as a domain controller with two Windows 10 VMs joined to it.

## Changes I’d like to make
I would eventually like to replace my ISP gateway with a Dell Optiplex SFF PFSense box, it would be capable of handling my available bandwidth and would allow me greater access/freedom on my own network. The ISP gateway would pretty much just be used for 802.x authentication.

I’m currently in the process of trying to combine all my services onto one Docker stack. I’d like to add the ability to self host a WordPress website, but the first step for that is to secure everything more, so I’m working on setting up SSL certs using Traefik 2 and LetsEncrypt. My current barriers are a less than optimal understanding of my current setup since I followed a tutorial initially, so I’ll need to put some work into creating a test environment replicating my current setup in case I mess something up.

Some hardware upgrades may also be in order soon. I’ll definitely need to double the RAM at some point, as well as add more storage. I’ll need a raid controller so I can set my storage up properly, and I’ll need to figure out a way to store my existing media for a small amount of time while I configure everything.