# WordPress on Lightsail: Swap and Service Watchdog

Add swap and a self-healing service watchdog to a WordPress site that runs on a single AWS Lightsail instance. Works for the Bitnami WordPress image and for a standard Linux install (Apache or Nginx, PHP-FPM, MariaDB or MySQL) managed by systemd.

**What you get**

- A swap file, so a memory spike makes the site slow instead of killing a process instantly.
- A watchdog that checks the database and the site every minute and restarts the web and PHP services after 3 consecutive failed checks, with cooldowns to prevent restart loops.

**What you do not get.** The watchdog restarts services silently. It does not send alerts, it does not help if the whole instance freezes, and it does not find the root cause of a failure. Pair it with external uptime monitoring.

The design and the swap steps were proven on a 2 GB Bitnami WordPress instance in Sydney (ap-southeast-2), including a real failure test. The generic script below is a refactor of the script used there, with a configuration block and a systemd branch added. It has passed a syntax check but has not itself been run on a server, so run the failure test (section 5) on every new instance before trusting it.

---

## Quick reference

```bash
# Is swap already there?
sudo swapon --show; free -m

# Create 2 GB swap
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
grep swapfile /etc/fstab || echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf && sudo sysctl --system

# Watchdog status and log
systemctl list-timers site-watchdog.timer --no-pager
sudo journalctl -t site-watchdog --since "7 days ago" --no-pager

# Pause and resume the watchdog
sudo systemctl stop site-watchdog.timer
sudo systemctl start site-watchdog.timer
```

---

## 0. Before you start

1. **Take a Lightsail snapshot.** Instance, Snapshots tab, Create instance snapshot, and wait until it is Available.
2. **Connect** with the browser SSH button on the Connect tab, or your own SSH client.
3. **Check for a memory problem first.** A watchdog and swap hide a memory leak or an undersized plan, they do not fix it.

```bash
free -m
df -h /
sudo journalctl -k --since "30 days ago" --no-pager | grep -i "out of memory"
sudo journalctl --list-boots --no-pager
```

- Out-of-memory lines mean the instance is running out of RAM. Investigate or resize before relying on a watchdog.
- If `--list-boots` shows only one boot, the journal is not persistent and the out-of-memory search only covers the time since the last reboot. Section 4 shows how to make it persistent.

---

## 1. Identify your stack

```bash
ls /opt/bitnami 2>/dev/null && echo "Bitnami stack"
systemctl list-units --type=service --no-pager | grep -E 'apache2|nginx|php.*fpm|mariadb|mysql'
```

| Stack | How services are controlled | Watchdog setting |
| --- | --- | --- |
| Bitnami WordPress image | `sudo /opt/bitnami/ctlscript.sh status`, and `restart apache`, `restart mariadb`, `restart php-fpm` | `STACK="bitnami"` |
| Standard Debian or Ubuntu install | `systemctl restart apache2` (or `nginx`), `php8.x-fpm`, `mariadb` (or `mysql`) | `STACK="systemd"` |

On Bitnami, `ctlscript.sh status` lists the services. If it includes `php-fpm`, PHP runs as its own service and the watchdog must restart it too, because restarting Apache alone does not fix a hung PHP pool.

Note: the Lightsail console shows older Bitnami WordPress images as no longer maintained by Bitnami. Plan patching or a migration separately.

---

## 2. Swap

### Check first

```bash
sudo swapon --show
free -m
ls -l /swapfile
grep swapfile /etc/fstab
```

Use `sudo` for `swapon` and `mkswap`. They live in `/usr/sbin`, which is not in the PATH of a normal user, so `swapon: command not found` without sudo is expected.

If `/swapfile` already exists and is listed by `swapon --show`, do not create it again. `fallocate` fails with "Text file busy" and `mkswap` with "is mounted; will not make swapspace" in that case. Skip to the fstab step.

### Size (rule of thumb)

| Instance RAM | Swap file |
| --- | --- |
| 512 MB to 1 GB | 2 GB |
| 2 GB | 2 GB |
| 4 GB | 2 to 4 GB |
| 8 GB or more | 2 GB, rarely used |

Keep at least 5 GB of disk free after creating the file. Swap is a safety margin, not extra RAM. If swap is in regular use, the instance is too small for its load.

### Create

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

If `fallocate` fails, use this instead of the first line:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress
```

### Make it survive a reboot

Only add the line if `grep swapfile /etc/fstab` printed nothing:

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Reduce eager swapping

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
cat /proc/sys/vm/swappiness
```

Expect `10`. A low value tells the kernel to use swap only when memory is genuinely tight.

### Verify, including the fstab line

```bash
sudo swapoff -a && sudo swapon -a
sudo swapon --show
free -m
```

Expect `/swapfile` at 2G and a Swap total of about 2047 MB. `swapon -a` reads `/etc/fstab`, so this proves the entry works at boot. A bad entry otherwise only shows itself at the next reboot.

---

## 3. Watchdog

### How it works

A shell script runs every minute from a systemd timer.

1. **Database check.** Ping the database. If it does not answer, restart it (5 minute cooldown) and stop.
2. **Web check.** Request the site through localhost with `curl --resolve`, which skips DNS, the CDN and the external network path. A 2xx or 3xx answer is healthy. A timeout, a 4xx or 5xx answer, or the WordPress "Error establishing a database connection" page is a failure.
3. **Count failures.** After 3 consecutive failures, restart PHP-FPM (if any) and then the web server (15 minute cooldown).
4. **Recover.** On the first healthy check after a failure, log a recovery line and clear the counter.

A healthy site writes nothing to the log.

### Choose the check URL (do this before installing)

The default check URL is `/wp-login.php`. It needs PHP and the database, so it exercises the whole stack, and it is not served from a page cache. Test it first from the server:

```bash
curl -sk -o /dev/null -w '%{http_code}\n' --resolve YOUR-DOMAIN:443:127.0.0.1 https://YOUR-DOMAIN/wp-login.php
```

It must print `200` or a `3xx` code. If it prints anything else, the watchdog would count failures on a healthy site and restart services in a loop. Common causes:

| Result | Cause | Fix |
| --- | --- | --- |
| `404` | A "hide login page" or security plugin has moved the login URL | Use another URL that returns 200 and runs PHP, for example the REST API root `/wp-json/` |
| `401` | HTTP basic authentication, usually on a staging site | Do not run the watchdog there, or check a URL that is not protected |
| `403` | A firewall or security plugin blocks the request | Allow localhost, or choose another URL |
| `000` | Nothing is listening on that port locally | Set `SCHEME` and `PORT` (see the configuration table) |
| `301` or `302` | Redirect to www or to HTTPS | Fine, 3xx counts as healthy |

### Configuration

All settings are in the first block of the script.

| Setting | Meaning |
| --- | --- |
| `STACK` | `bitnami` or `systemd` |
| `SITE_HOST` | Domain name exactly as the site answers it, for example `example.com.au` |
| `CHECK_PATH` | URL path to request, default `/wp-login.php` |
| `SCHEME` and `PORT` | `https` and `443`. Use `http` and `80` if the instance only serves HTTP behind a CDN or proxy |
| `DB_CHECK` | `yes` if the database runs on this instance. Set `no` for a managed database, such as Lightsail database or RDS, because a local ping would fail |
| `WEB_SERVICE` | systemd stack: `apache2` or `nginx` |
| `PHP_SERVICE` | systemd stack: for example `php8.3-fpm`, or empty if PHP runs inside Apache |
| `DB_SERVICE` | systemd stack: `mariadb` or `mysql` |
| `BITNAMI_PHP_FPM` | Bitnami stack: `yes` if `ctlscript.sh status` lists `php-fpm` |
| `MAX_FAILS` | Consecutive failures before a restart, default 3 |
| `WEB_COOLDOWN` and `DB_COOLDOWN` | Seconds between restarts, defaults 900 and 300 |
| `CURL_TIMEOUT` | Seconds before a web check counts as failed, default 20 |

### The script

Save this as `scripts/site-watchdog.sh` in your repository, and edit the configuration block for each site.

```bash
#!/bin/bash
# Site watchdog for a WordPress instance on Lightsail.
# Healthy site: no output and no log lines.
# Events are logged to the journal under the tag site-watchdog.

set -u

# ---- Configuration: edit this block only ----
STACK="bitnami"               # bitnami or systemd
SITE_HOST="example.com.au"
CHECK_PATH="/wp-login.php"
SCHEME="https"
PORT="443"
DB_CHECK="yes"                # no if the database is not on this instance
WEB_SERVICE="apache2"         # systemd stack: apache2 or nginx
PHP_SERVICE=""                # systemd stack: for example php8.3-fpm, or empty
DB_SERVICE="mariadb"          # systemd stack: mariadb or mysql
BITNAMI_PHP_FPM="yes"         # Bitnami stack: yes if ctlscript.sh lists php-fpm
MAX_FAILS=3
WEB_COOLDOWN=900
DB_COOLDOWN=300
CURL_TIMEOUT=20
# ---- End of configuration ----

TAG="site-watchdog"
STATE_DIR="/var/lib/site-watchdog"
CTL="/opt/bitnami/ctlscript.sh"

if [ "$STACK" = "bitnami" ]; then
  DB_ADMIN="/opt/bitnami/mariadb/bin/mariadb-admin"
  [ -x "$DB_ADMIN" ] || DB_ADMIN="/opt/bitnami/mariadb/bin/mysqladmin"
  DB_ARGS=(--socket=/opt/bitnami/mariadb/tmp/mysql.sock --connect-timeout=5)
else
  DB_ADMIN="$(command -v mariadb-admin || command -v mysqladmin || true)"
  DB_ARGS=(--connect-timeout=5)
fi

mkdir -p "$STATE_DIR"
now=$(date +%s)

log() { logger -t "$TAG" "$*"; }

# cooldown_ok FILE SECONDS: true if at least SECONDS have passed since the time stored in FILE
cooldown_ok() {
  local last=0
  [ -f "$1" ] && last=$(cat "$1" 2>/dev/null || echo 0)
  [ $((now - last)) -ge "$2" ]
}

restart_db() {
  if [ "$STACK" = "bitnami" ]; then
    "$CTL" restart mariadb
  else
    systemctl reset-failed "$DB_SERVICE" 2>/dev/null
    systemctl restart "$DB_SERVICE"
  fi
}

restart_web() {
  if [ "$STACK" = "bitnami" ]; then
    if [ "$BITNAMI_PHP_FPM" = "yes" ]; then
      "$CTL" restart php-fpm || log "php-fpm restart command returned an error"
    fi
    "$CTL" restart apache
  else
    if [ -n "$PHP_SERVICE" ]; then
      systemctl reset-failed "$PHP_SERVICE" 2>/dev/null
      systemctl restart "$PHP_SERVICE" || log "$PHP_SERVICE restart command returned an error"
    fi
    systemctl reset-failed "$WEB_SERVICE" 2>/dev/null
    systemctl restart "$WEB_SERVICE"
  fi
}

# ---- 1. Database check ----
# The ping exits 0 whenever the server answers, even if it refuses our login.
# It exits non-zero only if the server is down or unreachable.
if [ "$DB_CHECK" = "yes" ] && [ -n "$DB_ADMIN" ]; then
  if ! "$DB_ADMIN" "${DB_ARGS[@]}" ping >/dev/null 2>&1; then
    if cooldown_ok "$STATE_DIR/last_db_restart" "$DB_COOLDOWN"; then
      log "database did not answer, restarting it"
      echo "$now" > "$STATE_DIR/last_db_restart"
      restart_db >/dev/null 2>&1 || log "database restart command returned an error"
    else
      log "database is not answering and the cooldown is active, manual attention is needed"
    fi
    exit 0
  fi
fi

# ---- 2. Web check through localhost ----
# --resolve skips DNS, any CDN and the external network path. -k means an expired
# certificate does not cause a pointless restart.
body=$(mktemp)
code=$(curl -sk -o "$body" -w '%{http_code}' --max-time "$CURL_TIMEOUT" \
  --resolve "${SITE_HOST}:${PORT}:127.0.0.1" "${SCHEME}://${SITE_HOST}:${PORT}${CHECK_PATH}" 2>/dev/null)

healthy=0
case "$code" in
  2*|3*) healthy=1 ;;
esac
if grep -qi "Error establishing a database connection" "$body" 2>/dev/null; then
  healthy=0
fi
rm -f "$body"

if [ "$healthy" -eq 1 ]; then
  if [ -f "$STATE_DIR/fails" ]; then
    log "site recovered after $(cat "$STATE_DIR/fails") failed check(s)"
    rm -f "$STATE_DIR/fails"
  fi
  exit 0
fi

fails=$(( $(cat "$STATE_DIR/fails" 2>/dev/null || echo 0) + 1 ))
echo "$fails" > "$STATE_DIR/fails"
log "web check failed (http code ${code:-none}), consecutive failures: $fails"

if [ "$fails" -ge "$MAX_FAILS" ]; then
  if cooldown_ok "$STATE_DIR/last_web_restart" "$WEB_COOLDOWN"; then
    log "restarting PHP and web services after $fails consecutive failures"
    echo "$now" > "$STATE_DIR/last_web_restart"
    restart_web >/dev/null 2>&1 || log "web service restart command returned an error"
    echo 0 > "$STATE_DIR/fails"
  else
    log "restart skipped because the cooldown is active, manual attention is needed"
  fi
fi
```

### Install

**1. Preflight.** These must all succeed, otherwise stop.

```bash
command -v curl logger systemctl
sudo /opt/bitnami/ctlscript.sh status        # Bitnami only
systemctl status apache2 --no-pager | head   # systemd stack, use your service name
```

**2. Put the script on the server.** Pasting a long script into the browser terminal can insert a blank line after every line. A blank line after a `\` at the end of the `curl` line splits the command, drops the `--resolve` option, and the check silently tests the wrong thing. `bash -n` does not catch this. The safest route is to download the script from your repository:

```bash
sudo curl -fsSL -o /usr/local/bin/site-watchdog.sh \
  https://raw.githubusercontent.com/YOUR-USER/YOUR-REPO/main/scripts/site-watchdog.sh
```

If you paste instead, run `sudo nano /usr/local/bin/site-watchdog.sh`, paste, save with Ctrl+O then Enter, exit with Ctrl+X, and then run the checks in step 4.

**3. Edit the configuration block and set permissions.**

```bash
sudo nano /usr/local/bin/site-watchdog.sh
sudo chmod 755 /usr/local/bin/site-watchdog.sh
```

**4. Check the file.**

```bash
sudo bash -n /usr/local/bin/site-watchdog.sh && echo syntax ok
grep -n -A1 '\\$' /usr/local/bin/site-watchdog.sh
```

The line after the `curl -sk ...` line must be the `--resolve` line, not an empty line. If blank lines were inserted, remove them:

```bash
sudo sed -i '/^[[:space:]]*$/d' /usr/local/bin/site-watchdog.sh
```

**5. Create the systemd service and timer.**

```bash
sudo tee /etc/systemd/system/site-watchdog.service > /dev/null <<'EOF'
[Unit]
Description=Site watchdog (database and web health check)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/site-watchdog.sh
TimeoutStartSec=180
EOF

sudo tee /etc/systemd/system/site-watchdog.timer > /dev/null <<'EOF'
[Unit]
Description=Run the site watchdog every minute

[Timer]
OnBootSec=3min
OnUnitActiveSec=1min
AccuracySec=10s

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now site-watchdog.timer
```

### Verify the healthy path

```bash
sudo /usr/local/bin/site-watchdog.sh; echo exit=$?
systemctl list-timers site-watchdog.timer --no-pager
sudo journalctl -t site-watchdog --since '10 minutes ago' --no-pager
```

Expect `exit=0` with no other output, a timer line whose next run is within a minute, and `-- No entries --` in the journal.

A silent `exit=0` only means something if the web check itself returns a healthy code. Run the `curl` test from "Choose the check URL" again and confirm `200` or a `3xx` code.

Always add `--no-pager` to `systemctl` and `journalctl`, otherwise the output opens in a pager that traps the terminal until you press `q`. The `NEXT` column shows a dash while a check is running, which is normal.

---

## 5. Failure test (outside business hours)

This is the only proof that the restart path works. It takes the site down for about 3 to 4 minutes, so run it when nobody is using the site, and do not repeat it within 15 minutes because the cooldown is still active.

**Session 1: watch the log and leave it open.**

```bash
sudo journalctl -t site-watchdog -f
```

**Session 2: stop the web server.**

```bash
sudo /opt/bitnami/ctlscript.sh stop apache      # Bitnami
sudo systemctl stop apache2                     # systemd stack, use your service name
```

**Expected log, about a minute apart:**

```
web check failed (http code 000), consecutive failures: 1
web check failed (http code 000), consecutive failures: 2
web check failed (http code 000), consecutive failures: 3
restarting PHP and web services after 3 consecutive failures
site recovered after 0 failed check(s)
```

The "0" in the recovery line is cosmetic, because the script resets its counter when it restarts the services. Confirm the site loads in a browser afterwards.

If the site is not back after 5 minutes, start the service yourself (`sudo /opt/bitnami/ctlscript.sh start apache` or `sudo systemctl start apache2`) and read the full journal for the reason.

The database restart path is not covered by this test. Stopping the database on a live site is riskier than the information is worth, so leave it untested unless you have a staging copy.

---

## 6. Operations

### Weekly check

```bash
sudo journalctl -t site-watchdog --since "7 days ago" --no-pager
free -m
```

| Signal | Meaning | Action |
| --- | --- | --- |
| No log lines | Every check passed | None |
| One short failure then a recovery | A transient problem was handled | Note the time and look for a cause, such as a heavy plugin call or a backup job |
| Repeated restarts | A capacity problem or a faulty plugin, not a one-off crash | Investigate before changing any threshold |
| "manual attention is needed" | A restart was blocked by the cooldown, so the site may still be down | Check the site and the services now |
| Swap regularly in use above a few hundred MB | The instance is too small or something is leaking memory | Review memory use, consider a larger plan |

### Keep the journal across reboots (optional)

Without this, the log and the kernel out-of-memory history disappear at each reboot.

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
sudo journalctl --list-boots --no-pager
```

### Pause for maintenance

Pause the watchdog before planned work that stops the web server or database for more than a few minutes, and start it again afterwards.

```bash
sudo systemctl stop site-watchdog.timer
sudo systemctl start site-watchdog.timer
```

### Remove the watchdog

```bash
sudo systemctl disable --now site-watchdog.timer
sudo rm /etc/systemd/system/site-watchdog.service /etc/systemd/system/site-watchdog.timer /usr/local/bin/site-watchdog.sh
sudo systemctl daemon-reload
sudo rm -r /var/lib/site-watchdog
```

### Remove the swap file

```bash
sudo swapoff /swapfile
sudo rm /swapfile
sudo nano /etc/fstab                            # delete the line: /swapfile none swap sw 0 0
sudo rm /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

---

## 7. Pitfalls

| Symptom | Cause | Fix |
| --- | --- | --- |
| `fallocate: Text file busy` or `mkswap: ... is mounted` | `/swapfile` already exists and is active | Check with `sudo swapon --show`, do not recreate it |
| Swap gone after reboot | No `/etc/fstab` entry | Add the line, then test with `sudo swapoff -a && sudo swapon -a` |
| `swapon: command not found` | `/usr/sbin` is not in a normal user PATH | Run it with `sudo` |
| Pasted script has blank lines between all lines | Browser terminal paste inserts extra newlines | Download the script instead, or strip blank lines with `sed` and recheck the `curl` continuation |
| Watchdog restarts a healthy site repeatedly | The check URL returns 404, 401 or 403 locally | Test the URL with `curl --resolve` and choose one that returns 200 or 3xx |
| Watchdog restarts the database repeatedly | The database is not on this instance | Set `DB_CHECK="no"` |
| Restart fires but the site stays down | The cause is not a hung service, for example a full disk or a broken plugin | Read the journal, `df -h /`, and the web server error log |
| `systemctl` or `journalctl` output traps the terminal | The output opened in a pager | Press `q`, and use `--no-pager` |
| Timer `NEXT` shows a dash | A run is in progress, the timer re-arms when it finishes | Wait a few seconds and run the command again |
| Instance shows swap constantly in use | Instance too small, or a leak | Resize or fix the cause, the watchdog only hides it |

---

## 8. Limitations

- **No alerts.** Restarts are silent. Without external monitoring that alerts a person, a recurring fault can run for weeks unnoticed.
- **Whole-instance freezes.** A local script cannot act if the server hangs. Only an external reboot, for example a Lambda function calling `lightsail:RebootInstance`, covers that.
- **Root cause unknown.** A watchdog that fires often signals a capacity or traffic problem.
- **Swap is a margin, not a fix.** It turns an instant process kill into a slow site, which the web check timeout can then detect.
- **One site per run.** The script checks one hostname. For several sites on one instance, copy the script with a different `SITE_HOST`, state directory and tag, or extend the web check into a loop.
- **Unmaintained base images.** Older Bitnami WordPress images no longer receive updates from Bitnami, so operating system and stack patching needs its own plan.
