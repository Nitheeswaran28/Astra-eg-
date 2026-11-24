# SOC-1 Linux Essentials

## 1. Lab Overview

Investigate an Ubuntu host using its account records, processes, logs and startup files. Follow each lead back to the user, service or file involved.

## 2. Lab Environment

One Ubuntu-based Linux desktop with Bash, systemd, SSH configuration and Apache log files. Administrative access may be needed to inspect protected logs.

## 3. Workflow Diagram

Accounts → process ownership → authentication → HTTP evidence → file triage → startup and SSH checks

## 4. Tools Used

| Tool | Purpose |
|---|---|
| Bash, cat, grep | Read settings and filter records |
| ps, readlink, systemctl | Connect running processes to executables and services |
| journalctl, tail | Review recent activity |
| find | Select files by permissions, ownership or modification time |

## 5. Task-by-Task Walkthrough

### T1 — Accounts and policies

**Goal.** Review the local identity and password policy files.

1. Read the account and group lists. Compare each account's ID, home and shell.
2. Inspect login defaults and the PAM password policy.

```bash
cat /etc/passwd
cat /etc/group
cat /etc/login.defs
cat /etc/pam.d/common-password
```

**Command explained.** The passwd file describes accounts, group describes memberships and PAM controls password checks.

**Expected finding.** A service account and an interactive user may have different shells. Interpret the shell with the account's purpose.

### T2 — Processes and services

**Goal.** Trace a process to its executable and service.

1. Find a process of interest and set the shell variable PID to its observed numeric ID.
2. Review its parent, owner and executable. Inspect running and enabled services.

```bash
ps aux
ps -p $PID -o pid,ppid,user,cmd
readlink -f /proc/$PID/exe
systemctl list-units --type=service --state=running
systemctl status <service_name>
systemctl list-unit-files --type=service | grep enabled
```

**Command explained.** The -p selector limits ps to one PID. The -o list chooses columns. readlink -f resolves the executable link. Replace the service placeholder before running.

**Expected finding.** Running describes the current state. Enabled describes startup configuration.

### T3 — System and authentication logs

**Goal.** Review recent authentication activity.

1. Inspect the last 100 authentication records. Filter failures and sudo activity.
2. Compare SSH journal entries with system and kernel messages.

```bash
sudo tail -n 100 /var/log/auth.log
sudo grep -i "Failed" /var/log/auth.log | tail -n 50
sudo grep -i "sudo" /var/log/auth.log | tail -n 50
sudo journalctl -u ssh
cat /var/log/syslog
cat /var/log/kern.log
```

**Command explained.** tail -n limits the number of lines, grep -i ignores case and journalctl -u selects a systemd unit.

**Expected finding.** Compare the timestamp, source and account before linking a failure to later activity.

### T4 — Apache logs

**Goal.** Review recent HTTP requests and errors.

1. List the log directory and read the supplied backup access log.
2. Inspect error messages and filter the current access file for 404 responses.

```bash
ls -la /var/log/apache2/
sudo tail -n 50 /var/log/apache2/access.log.bak
sudo tail -n 20 /var/log/apache2/error.log
sudo grep " 404 " /var/log/apache2/access.log | tail -n 20
```

**Command explained.** The quoted spaces help match the status field. The backup and current file cover different periods.

**Expected finding.** Repeated missing paths can indicate enumeration. A 200 response alone does not show that a command embedded in a URL ran.

### T5 — Suspicious files

**Goal.** Identify file candidates for further review.

1. Inspect executable files in temporary directories.
2. Review recently modified binaries and files. Compare ownership, permission bits, location and time with expected activity.

```bash
sudo find /tmp /var/tmp /dev/shm -type f -perm /111 -ls 2>/dev/null
sudo find /bin /sbin /usr/bin /usr/sbin /usr/local/bin -type f -mtime -15 -ls 2>/dev/null | head -n 50
sudo find / -xdev -perm -4000 -type f -ls 2>/dev/null | head -n 100
sudo find /etc /var /tmp /dev/shm -type f -mtime -3 -ls 2>/dev/null | head -n 100
```

**Command explained.** -type f selects regular files, -perm /111 matches any execute bit, -4000 checks set-user-ID and -xdev stays on one filesystem. head bounds displayed matches.

**Expected finding.** An executable in a temporary directory is a review lead. A package update can explain recent system binaries.

### T6 — Startup inspection

**Goal.** Compare user and machine startup settings.

1. Replace <user> with the account under review. Check shell startup files and hidden files.
2. Review system profile fragments and enabled services, then compare user-level units.

```bash
ls -la /home/<user>/.bashrc /home/<user>/.bash_history /home/<user>/.profile
cat /home/<user>/.bashrc
find /home/<user> -type f -name ".*"
cat /etc/profile
ls -la /etc/profile.d/
systemctl list-unit-files --type=service | grep enabled
systemctl --user list-unit-files
```

**Command explained.** The quoted wildcard lets find match hidden filenames. The --user option queries the current user's service manager.

**Expected finding.** A shell startup file, user service and system service act at different scopes. Review the referenced path before drawing a conclusion.

### T7 — SSH access configuration

**Goal.** Review authorized public keys and SSH server settings.

1. Inspect the selected account's authorized_keys entries locally. Compare them with approved public keys.
2. Read the daemon configuration and review which authentication methods it permits.

```bash
sudo cat /home/<user>/.ssh/authorized_keys 2>/dev/null
sudo cat /etc/ssh/sshd_config
```

**Command explained.** authorized_keys contains public keys accepted for an account. The daemon configuration governs the server.

**Expected finding.** An unfamiliar key warrants ownership verification. Correlate with authentication logs for evidence of use.

## 6. Key Concepts (Beginner-Friendly)

A process is a running program. A service defines how a program is managed. A log records a past event, so its timestamp matters when comparing it with current state.

## 7. SOC / Real-World Relevance

Connect an authentication event to the account, related process and service. Use web and system logs to test whether a suspicious request produced host activity.

## 8. Quick Command Cheat Sheet

| Command | What It Does |
|---|---|
| `cat /etc/passwd` | Review the local identity and password policy files. |
| `ps aux` | Trace a process to its executable and service. |
| `sudo tail -n 100 /var/log/auth.log` | Review recent authentication activity. |
| `ls -la /var/log/apache2/` | Review recent HTTP requests and errors. |
| `sudo find /tmp /var/tmp /dev/shm -type f -perm /111 -ls 2>/dev/null` | Identify file candidates for further review. |
| `ls -la /home/<user>/.bashrc /home/<user>/.bash_history /home/<user>/.profile` | Compare user and machine startup settings. |
| `sudo cat /home/<user>/.ssh/authorized_keys 2>/dev/null` | Review authorized public keys and SSH server settings. |

## 9. Common Mistakes & Troubleshooting

- **Blank output.** Several lab searches redirect errors to /dev/null. Inspect permissions and paths before interpreting an empty result.
- **Wrong time window.** Check both active and rotated logs. A piped head or tail shows only part of the matching data.

## 10. Conclusion

Account and service records provide host context.
Logs help establish a sequence of events.
File and startup checks identify evidence that deserves closer review.
