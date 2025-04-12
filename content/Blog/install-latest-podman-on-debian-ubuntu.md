---
title: How to install latest version of podman on debian/ubuntu
publish: true
tags:
  - podman
  - debian
  - ubuntu
description: Download static build of podman to get the latest version of podman on debian.
date: 2025-04-12
---


I like stability of debian. I use it as my host distro for almost every server I have. But debian repositories (stable) have a very old version of podman. The same goes for debian testing, there's simply no podman 5>= avaliable. Debian backports don't work either - there's no backport of podman.

That left me with one other option - [static build of podman](https://github.com/mgoltzsche/podman-static).

## Installation on amd64 host

> Copied from https://github.com/mgoltzsche/podman-static?tab=readme-ov-file#binary-installation-on-a-host

Download the statically linked binaries of podman and its dependencies:

```sh
curl -fsSL -o podman-linux-amd64.tar.gz https://github.com/mgoltzsche/podman-static/releases/latest/download/podman-linux-amd64.tar.gz
```

Verify signature:

```sh
curl -fsSL -o podman-linux-amd64.tar.gz.asc https://github.com/mgoltzsche/podman-static/releases/latest/download/podman-linux-amd64.tar.gz.asc
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys 0CCF102C4F95D89E583FF1D4F8B5AF50344BB503
gpg --batch --verify podman-linux-amd64.tar.gz.asc podman-linux-amd64.tar.gz
```

Install binaries on the host:

```sh
tar -xzf podman-linux-amd64.tar.gz
sudo cp -r podman-linux-amd64/usr podman-linux-amd64/etc /
```

Install additional binaries:

```
apt install uidmap nsenter iptables slirp4netns
```

### Add UID/GID mapping

Check `/etc/subuid` and `/etc/subgid` for mappings:

```sh
cat /etc/subuid
cat /etc/subgid
```

If theres nothing in here add mapping:

```sh
sudo sh -c "echo $(id -un):100000:200000 >> /etc/subuid"
sudo sh -c "echo $(id -gn):100000:200000 >> /etc/subgid"
```

### Persist containers after logout

Enable linger:

```sh
loginctl enable-linger <USERNAME>
```

### Enable podman socket

```sh
systemctl enable --now --user podman.socket
```

### Enable automatic restart of services

```sh
systemctl enable podman-restart.service
```

Source: https://github.com/containers/podman/blob/main/contrib/systemd/system/podman-restart.service.in

## Install podman compose

```sh
apt install pipx
pipx install podman-compose
```

## Check if everything is working

```sh
$ podman run quay.io/podman/hello

!... Hello Podman World ...!

         .--"--.
       / -     - \
      / (O)   (O) \
   ~~~| -=(,Y,)=- |
    .---. /`  \   |~~
 ~/  o  o \~~~~.----. ~~
  | =(X)= |~  / (O (O) \
   ~~~~~~~  ~| =(Y_)=-  |
  ~~~~    ~~~|   U      |~~

Project:   https://github.com/containers/podman
Website:   https://podman.io
Desktop:   https://podman-desktop.io
Documents: https://docs.podman.io
YouTube:   https://youtube.com/@Podman
X/Twitter: @Podman_io
Mastodon:  @Podman_io@fosstodon.org
```