---
title: My experience running Truenas SCALE on unsupported setup
publish: true
tags:
  - homelab
  - truenas
description: Description used for preview
aliases:
  - other names for this note
date: 2023-06-26
---
TL;DR Some apps work flawlessly and some (those with postgres) take a lot of time to initialize. 2024 update: don't use USB drives, not worth it.

I don't write this post to rant about my bad experience, it was completely my fault. But to issue a warning for a newbies, to save them from my pain (and hours of debugging).

> Btw: the Truecharts team is awesome, they provide a ton of apps and their support is top notch. Please consider supporting them on Patreon.

## My (unsupported) setup

My story begins with my friend who's got into self-hosting, he introduced me to this concept and showed me how to start a journey of my own (thanks @pgronkievitz). So I began searching for a computer. I found an old HP with 6th gen core i7, 16GB of RAM for around $270. But there was a catch - this computer only had one SATA port (it was a mini PC), so to add more storage I had to use USB hard drives.

For software part I considered: Truenas Core/Scale, Proxmox or Yunohost. I choose Truenas Scale for their versitality. I could run VMs, Apps on k3s (Truecharts) or use build-in services. And with that I began setting up my homelab server. 

What's unsupported in this setup? USB hard drives. Type Truenas and and USB hard drives into search and you'll find hundreds of forum threads saying it's a bad idea. But they cannot stop me from doing it. 

![[c17830a585460b56cdbcccd05b300393_MD5.png]]

First app that I tried was Nexcloud from iXsystems. The app started just fine, though it was quite slow even though Iit was in the same (local) network with direct (no-vpn) connection. I thought of ways to improve it's speed and found alternative Nextcloud app from Truecharts. I installed it and it worked way faster, although was caught my attention was quite slow startup - the app sometimes took a few minutes to load. At first I thought that it had something to do with my HDD drives, but my friend, running similar setup but with debian + docker didn't have such issue running  same containers. For now I decided to ignore it, I mean how often would I restart the app? Well...

I wanted to sync my obsidian vault to Nextcloud, so I installed the Nextcloud desktop app. And the app stopped working after a minute with 404 error. I checked apps and saw that Nextcloud app was restarting... oh god, I had to fix this issue after all... or had I?

I don't have much experience with k3s, I checked the logs and didn't find anything interesting - so fixing the issue wasn't on the table. Maybe I didn't have to fix it - I jest had to make sure the Nextcloud app wouldn't restart, that meant I mustn't use Nextcloud Desktop. So I began looking for some other alternative apps that could sync directories between my devices. Then I could use Nextcloud's External storage to make them visible from Nextcloud.

Syncthing seemed to be great fit and it run just fine. I added my computer and server, the folders synced flawlessly.

Finally, my Nexcloud setup was done.

I had similar problems running Paperless-ngx, which also includes. But after inspecting the logs I found that it was hanging at mapping user to UID and GUID (ticket here: https://discord.com/channels/830763548678291466/1056109264927739954/1056109267461087295), which was probably caused by my HDDs.

## 2024 update

I'm passed all that. I quit after one of my drives started showing signs of wear. I took my old PC and bought refurbished hard drives. And so far my experience is much better. Night and day. So don't bother with USB drives.