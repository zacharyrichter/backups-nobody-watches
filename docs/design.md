# Design

## Purpose

The backup server (`pbs`) is where every Proxmox node backs up its guests. It keeps deduplicated backups in one datastore, and prunes, verifies and garbage-collects them on Sundays.

## Decisions

**Storage on the NAS, the server in a small VM.** The backups need tens of terabytes, which only the NAS array has. Running the backup server as a VM on the NAS and passing the share in over 9p puts the server next to its disks and off the Proxmox nodes it protects. If a node dies, the thing holding its backups is somewhere else entirely.

**Its own network.** The backup server is the only host on its own VLAN. The Proxmox nodes and the management host reach its API through the router, and nothing else can. It has no host firewall; the router is the control, as everywhere else in the lab.

| From | To | Why |
|---|---|---|
| The Proxmox nodes, the management host | pbs, port 8007 | Backups, and monitoring |
| The bastion, the management host | pbs, port 22 | SSH |
| pbs | the notification server, port 443 | Notifications |
| pbs | the Wazuh server, port 1514 | The Wazuh agent |
| pbs | the internet | Package updates, Let's Encrypt and the DNS provider for the certificate |

**One backup account, limited to backing up.** Every node connects with the same API token. Its role on the datastore (`DatastoreBackup`) lets it create backups and restore the ones it owns, and nothing else: it can't change the datastore, the jobs or anyone else's backups. A second, read-only token exists for monitoring.

**Retention belongs to the server.** Each node's storage entry says `keep-all=1`, meaning the node never prunes. The server's prune job decides what is kept, in one place:

| Prune setting | Value |
|---|---|
| Keep last | 3 |
| Keep daily | 7 |
| Keep weekly | 4 |
| Keep monthly | 3 |

A node that is misconfigured or rebuilt can't shorten retention, because it isn't the one doing it.

**A real certificate, checked by hostname.** The backup server has a Let's Encrypt certificate, requested by DNS challenge and renewed automatically. The nodes connect to it by name, with no pinned fingerprint, so the certificate is checked against the public authorities. The alternative is a pinned fingerprint, which changes at every renewal and silently breaks every backup the day after. That is the failure the audit exists to catch: see [verification.md](verification.md).

**No root account.** `root@pam` on the backup server has no permissions and is disabled. An administrator account holds the `Admin` role.

**Updated by hand.** Like the Proxmox nodes, the backup server never runs unattended-upgrades. An update that breaks backups should happen with someone watching, not at night.

**Standalone nodes, not a cluster.** Each node has its own `/etc/pve`. One node failing or being rebuilt never affects the others, and no quorum is needed. Settings they share come from Ansible, not the cluster filesystem.

**Clean shutdowns on a power cut.** The nodes are clients of the network UPS monitor, so a power cut stops the guests properly instead of cutting them off. This is checked by the same audit as the backup path.

## The datastore mount

The datastore is a 9p share from the NAS, mounted on the backup server:

```
pbs  /mnt/datastore/pbs  9p  trans=virtio,version=9p2000.L,rw,_netdev,nofail,cache=loose  0  0
```

`nofail` lets the server boot if the share isn't there, so a problem with the NAS leaves the server reachable for diagnosis instead of stuck in emergency mode. The audit requires the mount to exist and to be a 9p filesystem, because without the mount the datastore path is just a local directory on the server's small system disk.

The VM also has a few settings worth knowing when rebuilding it:

- **Clock:** the VM's clock is `localtime`, and the server reads its hardware clock as local time to match (`timedatectl set-local-rtc 1`), so it boots with the right time and chrony corrects the rest.
- **Guest agent:** `qemu-guest-agent` is installed so the NAS can shut the VM down cleanly.
- **Watchdog:** an `itco` watchdog resets the VM if it hangs.
- **Autostart:** it starts with the NAS array.

## Notifications

The backup server and every node use a **webhook** notification target to the notification server, with the token stored as a secret on the target and the body formatted as `[hostname] title: message`. The server's default matcher sends everything **except successful prune jobs** there, because a successful prune every week is noise. A mail target exists and is disabled.

The sender side is described in [no-mail-server](https://github.com/zacharyrichter/no-mail-server/blob/main/docs/consumers.md#proxmox-and-the-backup-server).
