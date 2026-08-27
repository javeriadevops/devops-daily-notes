# 2026-08-29 — Linux: systemd Services

**Topic:** Running an application as a managed service that survives reboots
**Why:** Follow-up from the processes notes — `nohup &` is not a real deployment

---

## 1. Why `&` is not enough

Starting an app with `python app.py &` fails in production because:

- It dies when the server reboots
- It does not restart if it crashes
- Logs go nowhere useful
- There is no clean way to stop or check on it

systemd handles all four.

---

## 2. Where service files live

```
/etc/systemd/system/myapp.service
```

The filename becomes the service name. `myapp.service` is controlled as `systemctl start myapp`.

---

## 3. A service file

```ini
[Unit]
Description=Resume Chatbot FastAPI App
# Wait until the network is available before starting
After=network.target

[Service]
# Run as this user, not root
User=javeria
Group=javeria

# Directory the command runs from
WorkingDirectory=/home/javeria/resume-chatbot

# Full absolute path to the executable - systemd does not use your PATH
ExecStart=/home/javeria/resume-chatbot/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000

# Restart the service if it exits for any reason
Restart=always

# Wait this many seconds before restarting
RestartSec=5

# Environment variables for the process
Environment="ENV=production"

[Install]
# Which target this service attaches to when enabled
WantedBy=multi-user.target
```

---

## 4. The three sections

| Section | Purpose |
|---|---|
| `[Unit]` | Metadata and ordering — description, what must start first |
| `[Service]` | How to actually run it — command, user, restart policy |
| `[Install]` | What happens when you `enable` it — boot behaviour |

---

## 5. Absolute paths are mandatory

`ExecStart=uvicorn main:app` will fail. systemd does not inherit your shell's `PATH`, so it cannot find `uvicorn`.

Find the real path first:

```bash
which uvicorn
```

Then paste that full path in. For a virtualenv it's the `venv/bin/` path, not the system one.

---

## 6. Commands

```bash
# Reload systemd after creating or editing a service file
sudo systemctl daemon-reload

# Start the service now
sudo systemctl start myapp

# Start automatically on boot
sudo systemctl enable myapp

# Both at once
sudo systemctl enable --now myapp

# Check status - shows running state and recent log lines
sudo systemctl status myapp

# Stop it
sudo systemctl stop myapp

# Restart after a code change
sudo systemctl restart myapp

# Stop it starting on boot
sudo systemctl disable myapp
```

`daemon-reload` is easy to forget. Edit the file, forget the reload, and systemd keeps running the old version — which looks like your change did nothing.

---

## 7. `start` vs `enable`

- **`start`** — runs it right now, this boot only
- **`enable`** — makes it start on every boot, but does not start it now

Two separate things. You usually want both.

---

## 8. Logs with journalctl

systemd captures stdout and stderr automatically. No log file setup needed.

```bash
# All logs for this service
sudo journalctl -u myapp

# Follow live
sudo journalctl -u myapp -f

# Last 50 lines
sudo journalctl -u myapp -n 50

# Since a time
sudo journalctl -u myapp --since "10 minutes ago"
sudo journalctl -u myapp --since today

# Only errors
sudo journalctl -u myapp -p err
```

`journalctl -u myapp -f` is the systemd equivalent of `docker logs -f`.

---

## 9. Debugging a service that will not start

Order I would follow:

1. `sudo systemctl status myapp` — usually names the problem directly
2. `sudo journalctl -u myapp -n 50` — the actual application error
3. Check `ExecStart` path exists: `ls -l /path/to/binary`
4. Try running the exact `ExecStart` command by hand as that user
5. Check the `User=` has permission on `WorkingDirectory`

Step 4 catches most of it. If it fails by hand, the problem is the app, not systemd.

---

## 10. For tomorrow

- [ ] Put Nginx in front of this as a reverse proxy
- [ ] Look at `Restart=on-failure` vs `Restart=always`
- [ ] Read about resource limits — `MemoryMax`, `CPUQuota`
