# ntpclock

![Shell: Bash](https://img.shields.io/badge/shell-bash-4EAA25?style=plastic&logo=gnubash&logoColor=white)
![License: MIT](https://img.shields.io/badge/license-MIT-blue?style=plastic)
![Platform: Linux](https://img.shields.io/badge/platform-linux-FCC624?style=plastic&logo=linux&logoColor=black)
![Maintained: Yes](https://img.shields.io/badge/maintained-yes-brightgreen?style=plastic)

A minimal Bash script that syncs the system clock from an NTP server and writes
the result to the hardware (RTC) clock. Originally written by Paul Swonger
(mawst) on January 19, 2003. Preserved, modernized, and re-documented in 2026.

---

## Why Does This Still Exist?

Modern Linux systems typically run `chronyd` or `ntpd` as a persistent daemon —
meaning time is kept in sync automatically and you never need to think about it.

**ntpclock still has legitimate uses when you don't want or can't run a daemon:**

- **Headless/embedded systems** with no persistent NTP daemon
- **Containers** (Docker, LXC) that inherit a bad clock from the host and need a one-shot fix
- **VMs** that drift and need a periodic hard correction (especially after sleep/resume)
- **Air-gapped or intermittently-connected machines** that sync on demand
- **Cron-based environments** where a lightweight one-shot sync is preferred over a running daemon
- **Legacy or resource-constrained hardware** where you want zero daemon overhead
- **Learning / understanding** how `ntpdate` and `hwclock` interact at a fundamental level

If you're running a modern always-on server with `chronyd` already active, you
probably don't need this. But if you do need it — it's here.

---

## Requirements

- Linux (any modern distro)
- Must be run as **root** or via `sudo`
- One of the following NTP tools installed:
  - `ntpdate` *(legacy but still widely packaged)*
  - `chronyd` *(preferred on systemd-based distros)*
  - `sntp` *(from the `ntp` package on some distros)*
- `hwclock` *(part of `util-linux`, present on virtually all Linux systems)*

### Installing Dependencies

**Debian / Ubuntu:**
```bash
sudo apt install ntpdate        # legacy, simple
# or
sudo apt install chrony         # modern, preferred
```

**RHEL / Fedora / CentOS:**
```bash
sudo dnf install ntpdate
# or
sudo dnf install chrony
```

**Alpine:**
```bash
apk add ntpdate
# or
apk add chrony
```

---

## Installation

```bash
# Clone the repo
git clone https://github.com/smokeluce/ntpclock.git
cd ntpclock

# Install the script
sudo install -m 755 bin/ntpclock /usr/local/bin/ntpclock
```

---

## Usage

```bash
sudo ntpclock [NTP_SERVER]
```

`NTP_SERVER` is optional and defaults to `pool.ntp.org`.

### Examples

```bash
# Sync using the default NTP pool
sudo ntpclock

# Sync using a specific server
sudo ntpclock time.cloudflare.com

# Sync using a local NTP server
sudo ntpclock 192.168.1.1
```

### Example Output

```
[ntpclock] Using sync tool: ntpdate
[ntpclock] NTP server: pool.ntp.org
 8 Mar 22:14:05 ntpdate[12345]: adjust time server 162.159.200.1 offset +0.003842 sec
[ntpclock] System clock synced from NTP.
[ntpclock] Hardware clock updated from system clock.
[ntpclock] Done. Current time: Fri Mar  6 22:14:05 CST 2026
```

---

## Scheduling with Cron

### Run hourly (as root)

Edit root's crontab with `sudo crontab -e` and add:

```cron
# Sync system clock from NTP every hour, log output
0 * * * * /usr/local/bin/ntpclock >> /var/log/ntpclock.log 2>&1
```

### Run at boot + every 6 hours

```cron
@reboot     /usr/local/bin/ntpclock >> /var/log/ntpclock.log 2>&1
0 */6 * * * /usr/local/bin/ntpclock >> /var/log/ntpclock.log 2>&1
```

### Run daily at 3am

```cron
0 3 * * * /usr/local/bin/ntpclock >> /var/log/ntpclock.log 2>&1
```

> **Note:** The cron daemon itself does not run as root by default. Always use
> `sudo crontab -e` (not your user's crontab) to schedule this, or prefix with
> `sudo` if your cron config supports it.

### Logrotate (optional)

To prevent `/var/log/ntpclock.log` from growing unbounded, add a logrotate config:

```
/var/log/ntpclock.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}
```

Save to `/etc/logrotate.d/ntpclock`.

---

## Scheduling with systemd (modern alternative to cron)

See [`systemd/`](systemd/) for drop-in unit files that run ntpclock as a
systemd timer — a cleaner, more observable alternative to cron on modern distros.

```bash
sudo cp systemd/ntpclock.service /etc/systemd/system/
sudo cp systemd/ntpclock.timer   /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ntpclock.timer
sudo systemctl list-timers ntpclock.timer
```

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0`  | Success |
| `1`  | Not running as root |
| `2`  | No supported NTP tool found |
| `3`  | NTP sync failed |
| `4`  | Hardware clock sync failed |

---

## Project Structure

```
ntpclock/
├── bin/
│   └── ntpclock          # The script
├── systemd/
│   ├── ntpclock.service  # systemd one-shot service unit
│   └── ntpclock.timer    # systemd timer unit (replaces cron)
├── cron-examples/
│   └── ntpclock.cron     # Ready-to-paste crontab examples
├── docs/
│   └── BACKGROUND.md     # History, ntpdate deprecation, alternatives
├── LICENSE               # MIT
└── README.md
```

---

## Background & History

`ntpdate` was deprecated in favor of `ntpd -q` and later `chronyd -q`, but it
remains in active use and is still packaged by most major distributions. This
script originally hardcoded `ntp0.mcs.anl.gov` (Argonne National Laboratory's
NTP server) — replaced here with `pool.ntp.org`, which load-balances across
thousands of volunteer servers worldwide and is the modern community standard.

See [`docs/BACKGROUND.md`](docs/BACKGROUND.md) for more context.

---

## License

MIT License. See [LICENSE](LICENSE).

Copyright © 2003–2026 Paul Swonger (mawst) — https://github.com/smokeluce/ntpclock
