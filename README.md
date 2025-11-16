# systemd Units & Timers — Examples

This repository contains several `systemd` services (and one timer) that demonstrate common patterns:

1) Reading variables from a `.env` and appending to a log  
2) Backing up an old log, writing a new line, then “emailing” (simulated)  
3) Running only if a flag file exists (two ways)  
4) Waiting for network to be fully online  
5) Weekly timer (Thu 18:00) + service that appends to a weekly log  
6) A failing service that retries 3 times, then stops

> All examples target Linux systems that use `systemd` (Ubuntu, Debian, Fedora, CentOS, etc.)

---

## Prerequisites

- `systemd` is PID 1 (`systemctl --version` should work).
- Root privileges for installing units to `/etc/systemd/system/`.
- Bash available at `/usr/bin/env bash` (default on most distros).

---

## Files & Layout

You can keep unit files in a local `units/` folder, then copy them to `/etc/systemd/system/`.

```
units/
  env-greeting.service
  backup-append-email.service
  3a-conditional.service
  3b-conditional-execcond.service
  4-after-network-online.service
  5-weeklyCheckIn.service
  5-weeklyCheckIn.timer
  6-retry-failing.service
```

---

## Installing or Updating Units

After copying any unit(s) to `/etc/systemd/system/`, always run:

```bash
sudo systemctl daemon-reload
```

Then enable and/or start the specific unit or timer as shown below.

---

## 1) Read .env and append greeting

**Goal:** On start, read `GREETING` and `NAME` from an env file and append to `/tmp/envlog.txt` like:
```
Assalomu alaykum, <Ismingiz>!
```

**Env file (example):** save as `/home/variables.env`
```dotenv
GREETING="Assalomu alaykum"
NAME="Ismingiz"
```

**Unit:** `env-greeting.service`
```ini
[Unit]
Description=Read .env and append greeting to /tmp/envlog.txt

[Service]
Type=oneshot
EnvironmentFile=/home/variables.env
ExecStart=/bin/bash -c 'printf "%%s, %%s!\n" "$GREETING" "$NAME" >> /tmp/envlog.txt'
Restart=no

[Install]
WantedBy=multi-user.target
```

**Install & test:**
```bash
sudo cp units/env-greeting.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now env-greeting.service
# Re-run anytime:
sudo systemctl start env-greeting.service
cat /tmp/envlog.txt
```

---

## 2) Backup old log, write new line, then “email” (simulated)

**Unit:** `backup-append-email.service`
```ini
[Unit]
Description=Back up old /tmp/my.log, append new text, then simulate email

[Service]
Type=oneshot
ExecStartPre=-/usr/bin/mv /tmp/my.log /tmp/my.log.bak
ExecStart=/usr/bin/env bash -lc 'echo "Ish boshladi" >> /tmp/my.log'
ExecStartPost=/usr/bin/echo "Email yuborildi"
Restart=no

[Install]
WantedBy=multi-user.target
```

**Install & test:**
```bash
sudo cp units/backup-append-email.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now backup-append-email.service
cat /tmp/my.log
ls -l /tmp/my.log*
```

---

## 3) Run only if `/etc/maintenance.flag` exists

### A) Using `ConditionPathExists=`
**Unit:** `3a-conditional.service`
```ini
[Unit]
Description=Run only if /etc/maintenance.flag exists
ConditionPathExists=/etc/maintenance.flag

[Service]
Type=oneshot
ExecStart=/usr/bin/echo "The file exists"
Restart=no

[Install]
WantedBy=multi-user.target
```

### B) Using `ExecCondition=` (systemd ≥ 240)
**Unit:** `3b-conditional-execcond.service`
```ini
[Unit]
Description=Run only if /etc/maintenance.flag exists (ExecCondition)

[Service]
Type=oneshot
ExecCondition=/usr/bin/test -e /etc/maintenance.flag
ExecStart=/usr/bin/echo "The file exists"
Restart=no

[Install]
WantedBy=multi-user.target
```

**Install & test:**
```bash
sudo cp units/3a-conditional.service units/3b-conditional-execcond.service /etc/systemd/system/
sudo systemctl daemon-reload
# Create the flag to allow run:
echo | sudo tee /etc/maintenance.flag >/dev/null

sudo systemctl start 3a-conditional.service
sudo systemctl start 3b-conditional-execcond.service

# Remove the flag and see them skip:
sudo rm -f /etc/maintenance.flag
sudo systemctl start 3a-conditional.service
sudo systemctl start 3b-conditional-execcond.service
```

---

## 4) Run only after network is fully online

**Notes:**  
- `After=network-online.target` controls order only.  
- `Wants=` or `Requires=` pull in the dependency.  
  - `Wants=` is soft (preferred here).  
  - `Requires=` is hard (service will fail if the dependency fails).

**Unit:** `4-after-network-online.service`
```ini
[Unit]
Description=Run only after the network is online
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/echo "Network is online; running job now."
Restart=no

[Install]
WantedBy=multi-user.target
```

**Install & test:**
```bash
sudo cp units/4-after-network-online.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now 4-after-network-online.service
# On a booted system already online, it starts immediately when enabled.
```

> If you need `network-online.target` to be reliable, also enable your distro’s *network-online* helper, e.g. `systemd-networkd-wait-online.service` or `NetworkManager-wait-online.service`.

---

## 5) Weekly timer — Thursday 18:00

**Service:** `5-weeklyCheckIn.service`  
(Appends a line to `/tmp/weekly.log`.)
```ini
[Unit]
Description=Weekly check-in

[Service]
Type=oneshot
ExecStart=/usr/bin/env bash -lc 'echo "Weekly check-in" >> /tmp/weekly.log'
Restart=no

[Install]
WantedBy=multi-user.target
```

**Timer:** `5-weeklyCheckIn.timer`
```ini
[Unit]
Description=Runs at 18:00 every Thursday

[Timer]
OnCalendar=*-*-4 18:00:00
Persistent=true
Unit=5-weeklyCheckIn.service

[Install]
WantedBy=timers.target
```

**Install & test:**
```bash
sudo cp units/5-weeklyCheckIn.service units/5-weeklyCheckIn.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now 5-weeklyCheckIn.timer

# Check timers:
systemctl list-timers --all | grep weeklyCheckIn

# Manual run (for testing now):
sudo systemctl start 5-weeklyCheckIn.service
cat /tmp/weekly.log
```

---

## 6) Failing service with 3 retries

**Goal:** Intentionally fail (exit code 1) and let `systemd` retry 3 times, then stop.

**Unit:** `6-retry-failing.service`
```ini
[Unit]
Description=Always fails via divide-by-zero; retry 3 times then stop
StartLimitIntervalSec=30
StartLimitBurst=4

[Service]
Type=oneshot
Environment=Var=1
ExecStart=/usr/bin/env bash -lc 'echo $((Var/0))'
Restart=on-failure
RestartSec=1

[Install]
WantedBy=multi-user.target
```

**Why `StartLimitBurst=4`?**  
With `Restart=on-failure`, systemd attempts a new start after a failure. One initial start + 3 automatic retries = 4 starts within `StartLimitIntervalSec`. After the 4th failure, systemd stops trying.

**Install & test:**
```bash
sudo cp units/6-retry-failing.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl start 6-retry-failing.service
sudo systemctl status 6-retry-failing.service
journalctl -u 6-retry-failing.service -b --no-pager
```

---

## Troubleshooting

- **Unit doesn’t start / syntax error:**  
  Check `systemd-analyze verify /etc/systemd/system/<unit>.service`

- **Logs:**  
  `journalctl -u <unit>.service -b --no-pager`

- **Timer not firing:**  
  `systemctl list-timers --all` to verify schedule and last/next run.

- **Network-online not honored:**  
  Ensure your distribution’s *wait-online* service is enabled (e.g. `NetworkManager-wait-online.service`).

- **Permissions / paths:**  
  Make sure files referenced in `EnvironmentFile=`, `ExecStartPre=`, etc. exist and are readable.

---

## Tezkor qisqa ko‘rsatmalar (Uzbek)

- **Unit o‘rnatish:** faylni `/etc/systemd/system/`ga ko‘chirib, `sudo systemctl daemon-reload` qiling.  
- **Enable/start:** `sudo systemctl enable --now <unit>.service`  
- **Timer:** `.timer`ni `enable --now`; xizmat qo‘lda sinash uchun tegishli `.service`ni `start` qiling.  
- **Shartli xizmat:** `/etc/maintenance.flag` mavjud bo‘lsa ishlaydi.  
- **Network online:** `Wants=` + `After=` bilan tarmoq to‘liq tayyor bo‘lgandan keyin ishga tushadi.  
- **3 marta urinish:** `Restart=on-failure`, `RestartSec=1`, `StartLimitBurst=4`.


