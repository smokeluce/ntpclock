# Background: ntpclock History & Context

## Origin

ntpclock was written by Paul Swonger (mawst) on January 19, 2003. At the time,
`ntpdate` was the standard tool for one-shot NTP synchronization on Linux, and
the script's entire logic was two lines:

```bash
ntpdate ntp0.mcs.anl.gov
hwclock --systohc
```

The original hardcoded server (`ntp0.mcs.anl.gov`) belongs to Argonne National
Laboratory's Math and Computer Science division, which has operated public NTP
servers for decades. It still responds today, but `pool.ntp.org` is the modern
community standard and is used as the default in this modernized version.

---

## The `ntpdate` Deprecation Story

`ntpdate` has been officially deprecated since around 2011 by the NTP Project,
who recommended replacing it with `ntpd -q` (run ntpd in one-shot mode). The
reasoning was that `ntpdate` used a simplified protocol implementation and could
make large, abrupt clock jumps rather than slewing gradually.

Despite this, `ntpdate` continues to be packaged by Debian, Ubuntu, Alpine, and
others because:

1. It's simple and does exactly one thing
2. Many init scripts and legacy tools depend on it
3. For machines not running a persistent daemon, abrupt correction is often fine

In practice, `ntpdate` is not going away anytime soon.

---

## Modern Alternatives

### chronyd (recommended for most use cases)
`chronyd` is the default NTP daemon on RHEL 8+, Fedora, and many modern distros.
It supports a one-shot query mode: `chronyd -q "server pool.ntp.org iburst"`.
This is what ntpclock uses when `ntpdate` is not found.

### systemd-timesyncd
Built into systemd, `systemd-timesyncd` runs as a lightweight background service
and handles NTP sync automatically. On systems where it's active, you may not
need ntpclock at all. Check with: `timedatectl status`.

### When ntpclock is still the right tool
- You want a simple, auditable, cron-scheduled sync with no daemon
- You're on a container or VM that doesn't have systemd-timesyncd configured
- You want a quick one-liner that logs to a file and exits
- You're on a resource-constrained or embedded system

---

## The `hwclock --systohc` Step

The two-step process (NTP → system clock, then system clock → hardware clock)
is intentional:

1. **NTP sets the system clock** (kernel time, kept in RAM)
2. **`hwclock --systohc` writes it to the RTC** (battery-backed hardware chip)

Without step 2, the hardware clock drifts back to its old value after reboot.
This is especially important for machines that don't run a persistent NTP daemon —
if the hardware clock is wrong, the system clock will be wrong at next boot.

---

## Pool.ntp.org

`pool.ntp.org` is a large virtual cluster of timeservers maintained by volunteers.
DNS round-robin load-balances queries across thousands of servers globally.
Subpools exist for regions: `us.pool.ntp.org`, `europe.pool.ntp.org`, etc.
Using the global `pool.ntp.org` is fine for most purposes.
