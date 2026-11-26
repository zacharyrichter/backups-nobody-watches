# Verification

A backup that has never been checked is a guess. The checks here are in three layers, from the cheapest to the most convincing.

## 1. The configuration is right

[diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit/blob/main/docs/proxmox-standards.md#the-backup-path) enforces this on every run, from the management host:

```
ansible-playbook playbooks/pve-standards.yml "${Z[@]}" --check --diff
```

On a node it checks that the backup storage connects by hostname as the dedicated token user with no pinned fingerprint, is enabled and `active` now, that every job writes to it and is enabled, that job modes match the database list, that the UPS client mode is `netclient`, and that a webhook notification target exists. On the backup server it checks that the datastore path is a mounted 9p share, that the datastore exists there with a garbage-collection schedule, that a prune job and a verify job exist for it, and that the webhook target exists.

A passing run says, per host, `backup storage and jobs, NUT and notifications OK` or `datastore, its jobs and notifications OK`, with `changed=0 failed=0`.

## 2. The path works now

The configuration can be correct and the path still broken, by a certificate, a login or a full disk. Two commands exercise the whole thing.

On each node:

```
sudo pvesm status --storage pbs-main        # active
```

`active` means the network, the login, the certificate and the datastore all worked just now. The audit runs the same command.

Then a real backup of a small guest, in snapshot mode, from one node:

```
sudo vzdump <vmid> --storage pbs-main --mode snapshot
```

It should finish, appear under the storage's Backups, and send a notification to the notification server. A test with the web interface's Test button proves the notification target exists; a real backup proves the matcher delivers what a real event produces.

## 3. The stored data is good

The backup server verifies its own contents. The verify job runs every Sunday and re-reads stored backups, skipping any verified in the last 30 days, so each backup is verified when it first appears and again once that window has passed. A verify failure is a notification, not a surprise at restore time.

On the backup server:

```
systemctl is-active proxmox-backup proxmox-backup-proxy qemu-guest-agent
findmnt -n /mnt/datastore/pbs                       # a 9p share
timedatectl | grep -E 'RTC in local|synchronized'   # yes / yes
sudo proxmox-backup-manager cert info | grep -E 'Subject|Issuer|Not After'
sudo cat /etc/proxmox-backup/prune.cfg /etc/proxmox-backup/verification.cfg
sudo cat /etc/proxmox-backup/acl.cfg
```

On each node, the jobs and the UPS client:

```
sudo grep -E '^vzdump|storage|mode' /etc/pve/jobs.cfg
upsc <ups-name>@10.0.99.1 ups.status        # OL
```

## What each layer catches

| Failure | Caught by |
|---|---|
| A pinned fingerprint, or the IP address in place of the hostname | Layer 1, before the next certificate renewal |
| A disabled job, or a job in the wrong mode | Layer 1 |
| The datastore share isn't mounted | Layer 1 |
| A broken certificate, login, network path or full datastore | Layer 2 |
| Backups that exist but can't be read | Layer 3, on Sunday |

The first row is the one worth the whole exercise. A pinned fingerprint works perfectly until the certificate renews, and then every backup fails at once, on a weekend, with nobody looking.
