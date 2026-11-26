# backups-nobody-watches

**Backups run at 1 a.m. on a weekend, and nobody looks at them. So the system is built so that a broken backup can't stay quiet.**

Four Proxmox nodes run every VM and container in my lab, and all of them back up to one Proxmox Backup Server. This repo is how that system is built and why: where the backups live, the weekly schedule and the reason each guest uses the mode it does, how a guest or a whole server is restored, and the checks that prove the path works rather than assuming it does.

```
pve1  pve2  pve3  pve4          four standalone nodes, one weekly job set each
  │     │     │     │
  └─────┴──┬──┴─────┘           HTTPS to pbs.example.net:8007
           ▼                    one token, allowed to back up and nothing else
   ┌──────────────────┐
   │ pbs (small VM)   │         own VLAN, reachable only from MGMT
   │ prune · GC ·     │         Sunday 03:45 / 04:00 / 04:15
   │ verify           │
   └────────┬─────────┘
            │ 9p share
            ▼
   NAS array (the datastore)    the server is small, the disks are not
            │
            ▼
   notification server          everything except a successful prune
```

## What's here

| Path | What it is |
|---|---|
| [`docs/design.md`](docs/design.md) | Why the system is built this way: where the data lives, who can touch it, how retention and certificates work |
| [`docs/schedule-and-modes.md`](docs/schedule-and-modes.md) | The weekly schedule, every guest's backup mode, and why databases are stopped |
| [`docs/restore-and-rebuild.md`](docs/restore-and-rebuild.md) | Restoring a guest, adding a node, rebuilding the backup server or a node, and what to do when backups fail |
| [`docs/verification.md`](docs/verification.md) | What proves a backup works, and the commands that check it |

## The ideas that matter

1. **The storage is next to the disks.** The backups need tens of terabytes, which only the NAS array has. The backup server is a small VM on that NAS with the share passed in over 9p, so the server sits beside its disks and off the nodes it protects.
2. **The nodes can back up, and nothing more.** Every node uses the same API token, whose role on the datastore lets it create and restore its own backups. Retention is decided by the server, never by a node.
3. **A real certificate, checked by name.** The nodes connect by hostname and check a Let's Encrypt certificate against the public authorities, with no pinned fingerprint. A renewal can't break the backups.
4. **Databases are stopped, not dumped.** A VM with a database backs up in `stop` mode for a consistent copy, on a schedule outside its busy hours. Everything else uses `snapshot`.
5. **Nothing here is trusted until it has been proven.** The storage must be `active`, every job must write to it, and the server runs a verify job every week. [diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit) turns those checks into a playbook.

## Limits

- **Weekly.** Jobs run once a week, so a restore can lose up to a week of changes.
- **The backup server isn't backed up.** It isn't a Proxmox guest. Its configuration is small and written down ([restore-and-rebuild.md](docs/restore-and-rebuild.md#rebuild-the-backup-server)), and the datastore is the backup that matters.
- **`stop` mode costs downtime.** A database VM is down for the length of its backup, in a window chosen to be quiet.

## Related

- [diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit): the playbook that audits the backup path, including the storage, the job modes and the server's jobs.
- [no-mail-server](https://github.com/zacharyrichter/no-mail-server): how the Proxmox nodes and the backup server send their webhook notifications.
- [alerts-worth-a-look](https://github.com/zacharyrichter/alerts-worth-a-look): the Wazuh stack whose data disk is left out of the backup, and why.
- [graceful-power-loss](https://github.com/zacharyrichter/graceful-power-loss): how the nodes shut their VMs down cleanly when the power goes.
- [no-open-doors](https://github.com/zacharyrichter/no-open-doors): the postmortem of a log rotation that left a reverse proxy stopped after Docker restarts, stop-mode backups among them.

Everything here is sanitized: domains are `example.net`, addresses are `10.0.x.x`, and the human account is `admin`.

MIT licensed. See [`LICENSE`](LICENSE).
