---
title: Setting obsidian-shellcommands plugin on flatpak
publish: true
tags:
  - meta
  - obsidian
---
Running Obsidian from flatpak comes with its own set of problems. One of them is that Obsidian is ran in sandbox environment - which makes sense when we consider security, but to be frank - it's annoying when I have to do something non-standard.

I wanted to run this command from obsidian (it updates my git repo):

```sh
npx quartz sync --no-pull
```

but after adding it to there:

![[shellcomands.png]]

I was welcomed with an error that it couldn't find `npx` command, which makes sense - I'm using [NVM](https://github.com/nvm-sh/nvm) to manage node. 

> [!tip]
> To make it work I had to make sure that `npx` was added to Obsidians `$PATH`  variable and obsidian has permissions to this directory. 

I used [Flatseal](https://flathub.org/apps/com.github.tchx84.Flatseal) to:
1. Give Obsidian permissions to `/home/$USER` directory - that's where `bin/` of NVM resides 
2.  Add NVM to `$PATH`: ![[obsidian-env.png]]


