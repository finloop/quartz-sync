---
title: Podman on debian How to fix podman no services after restart
publish: true
tags:
  - podman
  - systemd
  - sysadmin
description: I notices that with static build of podman, services didn't start after I restarted my server, even though they had Restart Always policy.
date: 2025-04-12
---
 
I notices that with static build of podman, services didn't start after I restarted my server, even though they had `Restart=Always` policy.

This is an issue that can popup when installing podman from a static build [[install-latest-podman-on-debian-ubuntu|installation on debian]]. 

Turns out that this issue is due to missing `podman-restart.service` that starts all the services after boot. 

This file was originally from [github repo](https://github.com/containers/podman/tree/main/contrib/systemd/system), but with static podman build it is not present.

Create directory for 

```sh
mkdir -p ~/.config/systemd/user/
```

Create file `podman-restart.service`:

```ini
[Unit]
Description=Podman Start All Containers With Restart Policy Set To Always
Documentation=man:podman-start(1)
StartLimitIntervalSec=0
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
RemainAfterExit=true
Environment=LOGGING="--log-level=info"
ExecStart=/usr/local/bin/podman $LOGGING start --all --filter restart-policy=always
ExecStop=/usr/local/bin/podman  $LOGGING stop  --all --filter restart-policy=always

[Install]
WantedBy=default.target
```

Reload units:

```sh
systemctl --user daemon-reload; journalctl -f
```

Enable service and timer:

```sh
systemctl --user enable podman-restart.service
```