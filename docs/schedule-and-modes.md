# Schedule and modes

Every guest is in a weekly backup job to the backup server, written down per node. Jobs add the guest's name as the backup note and use the notification system.

## The week

| When | Where | What |
|---|---|---|
| Saturday 01:00 | pve2 | The two containers, `snapshot` |
| Sunday 01:00 | pve2 | The web apps host and the home automation VM, `stop` |
| Sunday 01:30 | pve3 | The internet-facing VM, `snapshot` |
| Sunday 02:00 | pve1 | Every guest on the node, `stop` |
| Sunday 02:00 | pve4 | Every guest on the node, `snapshot` |
| Sunday 03:45 | pbs | Prune |
| Sunday 04:00 | pbs | Garbage collection |
| Sunday 04:15 | pbs | Verify |

The server's prune, garbage collection and verify all start after the week's backups.

## Guests and modes

| Node | Guest | Type | Mode | Why |
|---|---|---|---|---|
| pve1 | management host | VM | `stop` | Holds a database |
| pve1 | bastion | VM | `stop` | In the node's one job, which is `stop` because of the management host |
| pve2 | web apps host | VM | `stop` | Holds several databases |
| pve2 | home automation | VM | `stop` | Holds a database |
| pve2 | notification server | LXC | `snapshot` | No database; its own backup mustn't interrupt it |
| pve2 | print server | LXC | `snapshot` | No database |
| pve3 | internet-facing host | VM | `snapshot` | No database |
| pve4 | Wazuh server | VM | `snapshot`, system disk only | Its alert history, indexes and vulnerability feed sit on a separate data disk that the backup skips. All of it is rebuilt or downloaded again, so backing it up would cost storage for nothing. Agent registrations, certificates and config are on the system disk, which is backed up |

The notification server is the thing every other backup reports through, so it is backed up in `snapshot` mode and is never stopped by its own backup.

## Why databases are stopped

A snapshot of a running database is a crash-consistent copy: the data files as they were at an instant, with whatever the database had not yet flushed. It usually restores, but it relies on the database's crash recovery. A `stop` backup shuts the VM down cleanly, copies a state with nothing in flight, and starts it again.

I considered two alternatives and chose neither:

- **A dump hook** that exports each database before the backup. It would also give a consistent copy, but it means a script per database engine, kept correct as stacks change.
- **Suspend mode**, which pauses the VM instead of stopping it.

The existing `stop` jobs are simpler than either, and they cost one short outage in the early hours of Sunday morning. The jobs are scheduled outside the databases' normal write times.

The audit holds the line in both directions. A job in `stop` mode must be listed as a database job in the node's host variables, and a job that is listed must actually be in `stop` mode. Anything else fails the audit with the job's name. A new database VM can't be added to a `snapshot` job without someone noticing.

## When `stop` mode backfired

The internet-facing VM used to be backed up in `stop` mode. After one of those backups its reverse proxy was left stopped, and it stayed down until it was redeployed. The job was changed to `snapshot`, which is correct for a host with no database, and the real cause turned up days later in the log rotation rule that had marked the container as stopped by hand. The account is in [the postmortem in no-open-doors](https://github.com/zacharyrichter/no-open-doors/blob/main/docs/postmortem-the-logrotate-that-stopped-traefik.md).

## Adding a guest

Add it to its node's backup job, with the backup server's storage. If the guest holds a database, give it a `stop` job and list that job in the node's host variables, or the audit fails.
