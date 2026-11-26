# Restore and rebuild

## Restore a guest

On the Proxmox node: the guest's **Backup** tab, or the backup server's storage → **Backups** → pick one → **Restore**. Restoring over an existing guest needs it stopped first.

## Add a node

1. On the node: Datacenter → Storage → Add → Proxmox Backup Server. Storage ID and datastore as the other nodes have them, the server by **hostname**, the shared backup user and token (the secret from the password manager), **no fingerprint**, and `keep-all=1`.
2. Add the node's address to the router's alias of Proxmox nodes, so it can reach port 8007.
3. Add its guests to a backup job, with the mode rules in [schedule-and-modes.md](schedule-and-modes.md).

## Rebuild the backup server

The backups are on the NAS share and survive the loss of the server. The server is rebuilt around them.

1. Create the VM on the NAS: the 9p share for the datastore, the backups VLAN, autostart, UEFI. Install Proxmox Backup Server from the ISO with the address from the notes.
2. Bring it to the fleet standard: hardening, the SSH host certificate, the CIS baseline and the base and Proxmox standards, from the new-host procedure.
3. Add the `/etc/fstab` line and mount the share. Add the datastore **with its existing path**: the server reuses the chunks already there, and the old backups appear.
4. Recreate the prune job, the garbage-collection schedule and the verify job, the notification target and matcher, the users, tokens and ACLs, and the ACME account and DNS plugin, then order the certificate. A new token means a new secret, so every node's storage entry and the monitoring token need updating.
5. Set the hardware clock to local time (`timedatectl set-local-rtc 1`), and install the guest agent and the monitoring agents.
6. Run every check in [verification.md](verification.md), then a test backup from one node.

The server's settings are listed in [design.md](design.md) and in the tables here, because the server isn't a Proxmox guest and so isn't in any backup. Its configuration is small, and the datastore is what has to be safe.

## Rebuild a node

1. Set the BIOS to the standard settings.
2. Install Proxmox VE with the node's address and bridge layout, and turn Energy-Efficient Ethernet off on the NIC.
3. Bring it to the fleet standard: the SSH certificate, the CIS baseline, the base standard, then the Proxmox standard for that node.
4. Order its ACME certificate with the DNS plugin.
5. Re-add the local data storage where the node has one, and the backup server's storage as in *Add a node*, with no fingerprint.
6. Recreate the notification target and default matcher, disable the mail target, and configure the UPS client.
7. Restore the node's guests from the backup server, and recreate its backup jobs.
8. Install the monitoring agents, then run every check in [verification.md](verification.md).

## When something fails

| Symptom | Cause and fix |
|---|---|
| Backups fail with a certificate error | A node still pins a fingerprint or uses the server's IP address. Its storage entry must say `server <hostname>` with no `fingerprint` line |
| The datastore isn't mounted after a boot | `nofail` let the server boot without it. Check the share on the NAS, then `sudo mount /mnt/datastore/pbs` and restart `proxmox-backup-proxy` |
| The server boots with the wrong time | Check that `RTC in local TZ` is `yes` and the VM's clock is still `localtime`; chrony corrects the rest |
| An update breaks the server | Boot the previous kernel from the boot menu on the NAS's console. The datastore is untouched |
| The backup server is lost | Rebuild as above. The backups are on the NAS share and survive |
| A node won't boot after an update | Pick the previous kernel in the boot menu, then pin it with `proxmox-boot-tool kernel pin` until the problem is sorted |
