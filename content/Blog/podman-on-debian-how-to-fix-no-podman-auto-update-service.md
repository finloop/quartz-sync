---
title: Podman on debian How to fix no podman-auto-update.service
publish: true
tags:
  - systemd
  - podman
  - debian
description: This is an issue that can popup when installing podman from a static build of podman.
date: 2025-04-12
---

This is an issue that can popup when installing podman from a static build [[install-latest-podman-on-debian-ubuntu|Installation on Debian]].

To fix it we have to create a update service and timer. These were originally from [github repo](https://github.com/containers/podman/tree/main/contrib/systemd/system), but with static podman build they aren't.

Create directory for 

```sh
mkdir -p ~/.config/systemd/user/
```

Create file `podman-auto-update.service`:

```ini
[Unit]
Description=Podman auto-update service
Documentation=man:podman-auto-update(1)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/podman auto-update
ExecStartPost=/usr/local/bin/podman image prune -f

[Install]
WantedBy=default.target
```

Create file `podman-auto-update.timer`:

```ini
[Unit]
Description=Podman auto-update timer

[Timer]
OnCalendar=daily
RandomizedDelaySec=900
Persistent=true

[Install]
WantedBy=timers.target
```

Reload units:

```sh
systemctl --user daemon-reload; journalctl -f
```

Enable service and timer:

```sh
systemctl --user enable podman-auto-update.service
systemctl --user enable podman-auto-update.timer
```