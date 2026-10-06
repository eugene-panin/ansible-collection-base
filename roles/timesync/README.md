# timesync

systemd-timesyncd with NTP servers that answer, by default `pool.ntp.org`,
and a wait until the clock is synchronized.

Some providers drop the replies of the default servers of the distribution
(`ntp.ubuntu.com`): systemd-timesyncd then runs and never synchronizes, and
the clock drifts. A clock behind by seconds already breaks TLS certificates
issued elsewhere, and tokens with a lifetime.

```yaml
- hosts: servers
  become: true
  roles:
    - role: eugene_panin.base.timesync
```

Where chrony, ntp, ntpsec or openntpd keeps the time, as chrony does on
Google Cloud, the role leaves it alone and only waits for the
synchronization: installing systemd-timesyncd would remove it. Either way it
fails, with what to check, when the clock is not synchronized in
`timesync_wait_seconds`.
