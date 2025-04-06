---
title: ZFS Snapshots using sanoid and syncoid
publish: true
tags:
  - zfs
  - snapshots
description: Use sanoid and syncoid to craete snapshots of your ZFS datasets and send them to a remote machine.
date: 2025-04-06
---
 
Proxmox by default backups vms as tar archives/system images, which is not suitable for me, as I want to use zfs snapshots to ensure that I don't use many system resources. Then I want to replicate those snapshots to a remote system for an offsite backup.

## Installation

Follow the instructions from: https://github.com/jimsalterjrs/sanoid/blob/master/INSTALL.md

After you're done see if the `sanoid.timer` works. This timer will execute a policy from `/etc/sanoid/sanoid.conf` (empty by default).

## Configuration

1. Find all the things that you want to snapshot by running:

```sh
$ zfs list

NAME                       USED  AVAIL  REFER  MOUNTPOINT
rpool                     45.1G   412G   104K  /rpool
rpool/ROOT                2.34G   412G    96K  /rpool/ROOT
rpool/ROOT/pve-1          2.34G   412G  2.34G  /
rpool/data                28.6G   412G    96K  /rpool/data
rpool/data/vm-100-disk-0   128K   412G    64K  -
rpool/data/vm-100-disk-1  6.08G   412G  5.96G  -
rpool/data/vm-100-disk-2  19.7G   412G  18.7G  -
rpool/data/vm-101-disk-0  2.73G   412G  2.70G  -
rpool/var-lib-vz          14.1G   412G  14.1G  /var/lib/vz
```

2. Create file `/etc/sanoid/sanoid.conf`:

```txt
[rpool/data]
        use_template = production
        recursive = yes
        process_children_only = yes

#############################
# templates below this line #
#############################

[template_production]
        frequently = 0
        hourly = 36
        daily = 30
        monthly = 3
        yearly = 0
        autosnap = yes
        autoprune = yes
```

This template will backup all datasets in `rpool/data` but NOT the `rpool/data` itself. In this case I want to create snapshots of all the vm disks.

## Remote replication

Our setup will consist of two machines:
- production machine
- remote machine, a repote machine that will store our snapshots

### Remote machine configuration

Create dataset into which we'll replicate the data:

```sh
zfs create data/production-machine-xxx
```

Remote replication uses ssh as its transport layer and we don't want to give the remote machine root access. 

We need to create a new user: `adduser remote-backup`, and give him  privileges to write to it:

```sh
zfs allow -u remote-backup compression,mountpoint,create,mount,receive,rollback,destroy data/production-machine-xxx
```

### Production machine configuration

#### SSH Config

Edit `~/.shh/config`:

```txt
Host remote-machine:
  Hostname <ip addr>
  User remote-backup
```

Copy ssh key to remote machine:

```sh
ssh-copy-id remote-machine
```

#### Timer

Create `/etc/systemd/system/zfs-sync.timer`:

```txt
[Unit]
Description=Run syncoid daily

[Timer]
OnBootSec=30min
OnCalendar=*-*-* 21:00:00 Europe/Warsaw
Persistent=true

[Install]
WantedBy=timers.target
```

Example values that can be used in OnCalendar (credit: [Rockibul Islam](https://medium.com/@rockibul.islam20/configure-and-implement-of-systemd-timers-8c640cc6d667)):

```
Explaination                          Systemd timer
Every Minute                          *-*-* *:*:00
Every 5 minutes                       *-*-* *:*/5:00
Every 15 minutes                      *-*-* *:*/15:00
Every 30 minutes                      *-*-* *:*/30:00
Every 1 hour                          *-*-* *:00:00
Every other hour                      *-*-* */2:00:00
Every 12 hour                         *-*-* */12:00:00
Between certain hours                 *-*-* 9-17:00:00
Daily                                 *-*-* 00:00:00
Every Night                           *-*-* 01:00:00
Every Night at 2am                    *-*-* 02:00:00
Every morning                         *-*-* 07:00:00
Every midnight                        *-*-* 00:00:00
Every sunday                          Sun *-*-* 00:00:00
Every friday at midnight              Fri *-*-* 00:00:00
Every weekday                         Mon...Fri *-*-* 00:00:00
Every weekend                         Sat,Sun *-*-* 00:00:00
Every 7 days                          * *-*-* 00:00:00
monthly                               * *-*-01 00:00:00
Every quarter                         * *-01,04,07,10-01 00:00:00
Every 6 months                        * *-01,07-01 00:00:00
Every year                            * *-01-01 00:00:00
```

#### Oneshot service

Create file `/etc/systemd/system/zfs-sync.service`:

```txt
[Unit]
Description=Send zfs snapshots to remote machine
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/syncoid --no-privilege-elevation -r --skip-parent rpool/data remote-machine:data/public-services
[Install]
WantedBy=default.target
```

Enable the timer and service:

```
systemctl daemon-reload; journalctl -f
systemctl enable zfs-sync.timer
systemctl enable zfs-sync.service
```

Start `zfs-sync.service` manually to sync the first snapshot:

```sh
systemctl start zfs-sync.service
```

You can list snapshots on the remote machine by running:

```sh
zfs list -t snapshot
```