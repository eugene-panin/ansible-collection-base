# backup

restic backups on a systemd timer, to a repository off the host. The
repository is any that restic supports, given by its address. An `sftp:` one
needs nothing but a second machine you can log in to over SSH: another VPS, a
NAS at home, a storage box.

```yaml
- name: Backup
  hosts: all
  become: true
  roles:
    - role: eugene_panin.base.backup
      vars:
        backup_repository: sftp:backup@storage.example.com:/srv/backups/web
        backup_password: "{{ vault_backup_password }}"
        backup_ssh_private_key: "{{ vault_backup_ssh_key }}"
        backup_ssh_known_hosts: storage.example.com ssh-ed25519 AAAAC3...
        backup_paths: [/opt/nomad/data/host_volumes]
        backup_hooks:
          - name: vault
            command: vault operator raft snapshot save "$BACKUP_HOOK_OUTPUT/vault.snap"
        backup_hook_environment:
          VAULT_TOKEN: "{{ vault_backup_vault_token }}"
```

For an S3 bucket, set `backup_repository: s3:https://s3.example.com/bucket`
and the keys in `backup_environment`, `AWS_ACCESS_KEY_ID` and
`AWS_SECRET_ACCESS_KEY`.

## What happens

- A pinned restic release is downloaded, checked against its SHA-256, and
  unpacked to `/usr/local/lib/restic/restic-<version>`.
  `/usr/local/bin/restic` points at it.
- The repository is created if it does not exist. restic compresses,
  deduplicates and encrypts what it stores; `backup_compression` is `auto`
  by default, `max` for less space at more CPU.
- Every day, `restic-backup.timer` does three things:
  1. runs the hooks in order; each writes what it dumps under
     `$BACKUP_HOOK_OUTPUT`;
  2. backs up `backup_paths` with that directory;
  3. forgets and prunes the snapshots `backup_keep` no longer covers, by
     default 7 daily, 4 weekly and 6 monthly.

  A failing hook fails the whole run, so no backup goes out without its dump.
- Every week, `restic-check.timer` checks the repository and reads back
  `backup_check_subset` of its data.
- The password, the environment of restic and of the hooks, and the SSH key
  live in `/etc/restic`, readable by root only.
- The host key of an `sftp:` server is checked against
  `backup_ssh_known_hosts`, never trusted on first use.

Keep the repository password somewhere off the host: the backups cannot be
read without it.

## Restore

`backup-restic` is restic with the repository and its credentials already
set:

```bash
backup-restic snapshots
backup-restic restore latest --target /tmp/restore
backup-restic restore latest --target / --include /opt/nomad/data/host_volumes/mail-data
```

## Tested

The scenario runs a host and a repository server that only has SSH. On the
host it checks that:

- the secrets in `/etc/restic` are root's only, and both timers are enabled;
- the backup the timer runs goes to the server over `sftp:`;
- a restore of the latest snapshot brings back the data and the hook's
  output, without the file `backup_exclude` leaves out;
- after a second backup the same day, the retention keeps the latest
  snapshot only, and it restores the changed data;
- the repository check passes;
- restic refuses the server once its known host key is replaced by another;
- a hook no longer declared, and its environment, are removed.

Each scenario runs converge, idempotence and `--check` on a fresh and on a
converged host.
