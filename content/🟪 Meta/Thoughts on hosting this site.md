---
title: Thoughts on hosting this site
publish: true
tags:
  - "#meta"
description: All about hosting this site.
aliases:
  - hosting this site
  - garden.piotrk.it
date: 2024-12-31
---

This page was built using [Quartz](https://quartz.jzhao.xyz/) and so far I'm loving it. I've set it up such that it publishes automatically using [Cloudflare Pages](https://pages.cloudflare.com/) because I wanted to get something running fast.  One advantage of Cloudflare is that it gives me analytics for free. My end goal is to move out of Cloudflare and self-host.

Updating this site is super easy,:

```sh
npx quartz sync --no-pull
```

and the site is up!

To make this process less painful (I'm lazy and don't want to open a terminal) I've set up [Obsidian Shellcommands](https://github.com/Taitava/obsidian-shellcommands) plugin. You can read abut this process here [[Setting obsidian-shellcommands plugin on flatpak|setting obsidian-shellcommands plugin on flatpak]]. 

Now running `Ctrl + P`  and typing  `Update page` updates the page directly from obsidian.
