# 2026-08-27 — Linux: Permissions and Processes

**Topic:** File permissions, ownership, and managing running processes
**Why:** These two come up constantly when deploying an app on a server

---

## 1. Reading `ls -l` output

```
-rw-r--r--  1 javeria devops  1240 Aug 27 10:15 app.py
drwxr-xr-x  2 javeria devops  4096 Aug 27 10:12 logs
```

The first character is the type: `-` file, `d` directory, `l` symlink.

The next nine characters are three groups of three:

```
rwx  r-x  r--
 |    |    |
user group other
```

| Letter | On a file | On a directory |
|---|---|---|
| `r` | read contents | list what's inside |
| `w` | modify contents | create/delete files inside |
| `x` | execute it | enter it with `cd` |

The `x` bit on a directory tripped me up — you need it just to `cd` in, even if you have `r`.

---

## 2. Numeric permissions

Each letter has a value: `r`=4, `w`=2, `x`=1. Add them per group.

| Number | Meaning |
|---|---|
| 7 | rwx |
| 6 | rw- |
| 5 | r-x |
| 4 | r-- |
| 0 | --- |

Common ones:

```bash
# Owner can read/write, everyone else read-only - normal file
chmod 644 app.py

# Owner full, others read+execute - scripts and directories
chmod 755 deploy.sh

# Owner read/write only - secrets, SSH private keys
chmod 600 .env
```

An SSH private key **must** be `600`. If it's more open than that, SSH refuses to use it and throws a permissions error.

---

## 3. Ownership

```bash
# Change the owner of a file
sudo chown javeria app.py

# Change owner and group together
sudo chown javeria:devops app.py

# Apply recursively to a directory
sudo chown -R javeria:devops /var/www/myapp
```

A very common deployment problem: Nginx runs as user `www-data`, so if your app files are owned by root with `700`, Nginx gets a 403 and cannot read them.

---

## 4. Looking at processes

```bash
# Snapshot of all processes
ps aux

# Filter for a specific one
ps aux | grep uvicorn

# Live, interactive view
top

# Nicer version if installed
htop

# Find the PID of a process by name
pgrep -f uvicorn

# See which process is holding a port
sudo lsof -i :8000
```

That last one is the fastest way to answer "why does it say port already in use."

---

## 5. Stopping processes

```bash
# Ask the process to shut down gracefully (SIGTERM)
kill 4821

# Force kill, no cleanup (SIGKILL) - last resort
kill -9 4821

# Kill by name
pkill -f uvicorn
```

Try plain `kill` first. `kill -9` gives the process no chance to close files or flush data, which can leave things in a broken state.

---

## 6. Background jobs

```bash
# Run in the background
python app.py &

# List background jobs in this shell
jobs

# Bring job 1 back to the foreground
fg %1

# Keep running after you log out
nohup python app.py &
```

Important: `&` alone is not enough for a real deployment. Closing the SSH session kills the process. For anything real, use `systemd` or PM2 instead.

---

## 7. Disk and memory quick checks

```bash
# Disk usage per filesystem, human readable
df -h

# Size of a directory
du -sh /var/log

# Memory usage
free -h
```

`df -h` is usually the first thing to run when a server starts behaving strangely — a full disk causes failures that look like completely unrelated bugs.

---

## 8. For tomorrow

- [ ] Write a `systemd` service file for a Python app
- [ ] Read up on `journalctl` for reading service logs
- [ ] Understand the difference between `su` and `sudo -i`
