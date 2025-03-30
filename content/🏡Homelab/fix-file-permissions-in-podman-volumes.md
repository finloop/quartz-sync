---
title: Fix file permissions in podman volumes
publish: true
tags:
  - podman
description: How to fix common permissions issues in podman volumes (both local and bind mounts).
date: 2025-03-30
---

How to fix common permissions issues in podman volumes (both local and bind mounts)? This post 

The obvious way to fix them is to `chown`, but as a user we cannot execute this command - users are not permitted to change ownership of files for security reasons. Buttt: podman has a command `podman unshare` (I know, this name is not intuitive...) that lets us pretend to be root the same way as podman rootless containers do (root inside container is a current user, more on that in [this article](https://www.tutorialworks.com/podman-rootless-volumes/)).

We can execute `podman unshare chown -R 0:0 ./volume/path` to change ownership of the volume to you user (that is `id -u` on HOST).

Or if you want to change it to the `user` that was set:
-  inside container, created by command eg. `podman run --user 1000` 
-  inside quadlet: `User=1000`

Run this command:

```sh
podman unshare chown -R 1000:1000 ./volume/path
```

## References
- https://www.tutorialworks.com/podman-rootless-volumes/
- https://docs.podman.io/en/latest/markdown/podman-unshare.1.html