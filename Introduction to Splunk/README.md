# Introduction to Splunk

## 1. Lab Overview

Collect authentication and system events from two operating systems in a single Splunk index. Generate recognizable test events and trace them from the endpoint to a search result. Findings below are checks to perform in the lab.

## 2. Lab Environment

The SOC machine runs Splunk Enterprise. Ubuntu and Windows each have a Universal Forwarder installer on the desktop. Splunk Web uses port 8000 and the receiver uses TCP 9997. Replace IP and hostname placeholders with values from your machines.

## 3. Workflow Diagram

Ubuntu files / Windows channels → Universal Forwarder → Splunk receiver → main index → SPL search

## 4. Tools Used

| Tool | Purpose |
|---|---|
| rsyslog and auditd | Write system and audit activity on Ubuntu |
| Event Viewer and policy editors | Inspect Windows channels and enable auditing |
| Universal Forwarder | Send selected events to the receiver |
| Splunk Web and SPL | Search and visualize indexed events |

## 5. Task-by-Task Walkthrough

### T1: Verify Splunk Server

**Goal.** Verify the server and find its reachable address.

1. Start all three lab machines.

2. On the SOC machine, open Firefox and visit `http://localhost:8000`.

3. Identify the server IP:

```bash
ip addr
```

**Command explained.** Displays network interfaces and IP addresses. The endpoints need the server IP to forward logs.

**Expected finding.** Verify that Splunk Web is accessible and identify the server IP.

### T2: Enable Logging on Ubuntu

**Goal.** Check that Ubuntu produces logs before configuring collection.

1. Check rsyslog:

```bash
systemctl status rsyslog
```

2. If inactive:

```bash
systemctl start rsyslog
systemctl enable rsyslog
```

**Command explained.** rsyslog collects system logs. `start` launches the service; `enable` configures automatic startup.

3. Verify log files:

```bash
ls -la /var/log/syslog
ls -la /var/log/auth.log
ls -la /var/log/kern.log
```

- syslog: General system activity.
- auth.log: Authentication, SSH and sudo activity.
- kern.log: Kernel messages.
- `ls -la`: Displays file details and permissions.

4. Enable and verify audit logging:

```bash
systemctl start auditd
systemctl enable auditd
systemctl status auditd
ls -la /var/log/audit/audit.log
```

**Command explained.** auditd records security audit events according to configured audit rules.

**Expected finding.** Verify system, authentication, kernel and audit logging.

### T3: Install Splunk Universal Forwarder on Ubuntu

**Goal.** Configure Ubuntu to send its local log files.

1. Install the provided package:

```bash
cd Desktop
dpkg -i splunkforwarder-linux-amd64.deb
/opt/splunkforwarder/bin/splunk start --accept-license
```

2. Create administrator credentials when prompted.

**Command explained.** `dpkg -i` installs a Debian package. Universal Forwarder is a lightweight agent that sends logs to Splunk.

3. Configure the receiving server (replace the placeholder with the actual IP):

```bash
/opt/splunkforwarder/bin/splunk add forward-server <SPLUNK_IP>:9997
```

4. Add log files for monitoring:

```bash
/opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -index main
/opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -index main
/opt/splunkforwarder/bin/splunk add monitor /var/log/kern.log -index main
/opt/splunkforwarder/bin/splunk add monitor /var/log/audit/audit.log -index main
```

**Command explained.** `add monitor` watches a file; `-index main` selects the destination index.

5. Edit the configuration:

```bash
nano /opt/splunkforwarder/etc/apps/search/local/inputs.conf
```

6. Under the syslog monitor section:

```ini
[monitor:///var/log/syslog]
disabled = false
index = main
crcSalt = <SOURCE>
```

**Command explained.** `crcSalt = <SOURCE>` includes the source path in the file identity calculation, preventing files with similar initial content from being mistaken for duplicates. Use carefully with rotating logs because renamed files may be ingested again.

7. Save with Ctrl+O, Enter, Ctrl+X. Restart:

```bash
/opt/splunkforwarder/bin/splunk restart
```

**Expected finding.** Ubuntu forwarder is configured to monitor four log files.

### T4: Configure Splunk to Receive Logs

**Goal.** Open the receiving port on the SOC machine.

1. SOC machine → Splunk Web → Settings → Forwarding and receiving → Configure receiving → New Receiving Port.

2. Enter `9997` and save.

**Command explained.** The server listens on this port for forwarded data.

**Expected finding.** Verify that Splunk is configured to receive logs.

### T5: Generate and Verify Ubuntu Test Events

**Goal.** Trace a known Ubuntu event from disk to Splunk.

1. Generate a failed SSH login using an incorrect password, a sudo event and a custom message:

```bash
ssh invaliduser@localhost
sudo ls /root
logger "Splunk Test Event: This is a test log entry from Ubuntu Machine"
tail -5 /var/log/auth.log
tail -5 /var/log/syslog
```

**Command explained.** SSH produces authentication events; sudo records privileged activity; logger writes a test message; tail displays the last five lines.

2. SOC machine → Splunk Web → Search & Reporting.

```spl
index=main host=<ubuntu-hostname>
```

3. Search specific events (replace the example hostname):

```spl
index=main host=<ubuntu-hostname> "invaliduser"
index=main host=<ubuntu-hostname> "ls /root"
index=main host=<ubuntu-hostname> "Splunk Test Event"
```

**Command explained.** SPL (Search Processing Language) searches and filters collected events.

**Expected finding.** Verify that Ubuntu test events appear in Splunk.

### T6: Enable Security Logging on Windows

**Goal.** Enable the Windows channels needed for investigation.

1. Win+R → `eventvwr.msc` → Windows Logs.

2. Verify Application, Security, System and Setup logs.

3. Win+R → `secpol.msc` → Security Settings → Local Policies → Audit Policy.

4. Enable Success and Failure for account logon events, account management, logon events, process tracking and system events, as specified by the lab.

**Command explained.** Audit policies determine which successful and failed security activities Windows records.

5. Win+R → `gpedit.msc` → Computer Configuration → Administrative Templates → Windows Components → Windows PowerShell → Turn on PowerShell Script Block Logging → Enabled → OK.

**Command explained.** Script Block Logging records PowerShell script content for investigation.

**Expected finding.** Verify Windows auditing and PowerShell logging.

### T7: Install Splunk Universal Forwarder on Windows

**Goal.** Connect the Windows endpoint and select its channels.

1. Open the provided MSI installer on Desktop → Accept license → Create credentials → Leave Deployment Server blank.

2. Receiving Indexer: enter actual Splunk IP and port `9997` → Install → Finish.

3. Open Command Prompt as Administrator:

```cmd
notepad "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"
```

4. Add:

```ini
[WinEventLog://Security]
index = main
disabled = false
renderXml = false

[WinEventLog://System]
index = main
disabled = false
renderXml = false

[WinEventLog://Application]
index = main
disabled = false
renderXml = false

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
index = main
disabled = false
renderXml = false
```

**Command explained.** WinEventLog identifies the channel; index selects storage; disabled=false enables collection; renderXml=false uses rendered event text.

5. Save and verify connection, then restart:

```cmd
cd "C:\Program Files\SplunkUniversalForwarder\bin"
splunk list forward-server
splunk restart
```

**Expected finding.** Windows forwarder is configured to collect Security, System, Application and PowerShell logs.

### T8: Generate Windows Security Test Events

**Goal.** Match Windows test activity to indexed events.

1. Generate a failed login (enter a password when prompted):

```cmd
runas /user:fakeuser cmd
```

**Command explained.** An invalid account authentication attempt can generate Security Event ID 4625 (failed logon).

2. Generate service events in Administrator Command Prompt:

```cmd
net stop spooler
net start spooler
```

**Command explained.** Stops and restarts Print Spooler, generating service-related events.

3. Generate a custom event in PowerShell:

```powershell
Write-EventLog -LogName Application -Source "Application" -EventId 9999 -EntryType Information -Message "Splunk Test Event: This is a test log entry from Windows Machine"
```

**Command explained.** Writes a test Application event if the event source is registered.

4. SOC machine → Search & Reporting. Search each query separately:

```spl
index=main host="<windows-hostname>"
index=main "Splunk Test Event" host="<windows-hostname>"
index=main host="<windows-hostname>" *spooler*
index=main EventCode=4625 host="<windows-hostname>" "fakeuser"
```

**Expected finding.** Verify Windows test events in Splunk, confirming log ingestion.

### T9: Explore the Splunk Dashboard

**Goal.** Compare event volume and sources across hosts.

1. Search & Reporting → View all events:

```spl
index=main
```

2. Click host in the Fields sidebar to identify monitored machines; click source to identify log types.

3. Visualize event counts over time:

```spl
index=main | timechart count by host
```

4. Click Visualization. This helps identify spikes in activity.

5. Identify top log sources:

```spl
index=main | top source
```

6. Search authentication-related events across both machines:

```spl
index=main (source="/var/log/auth.log" OR source="WinEventLog:Security")
```

**Expected finding.** Splunk can search, filter and visualize logs from both endpoints.


## 6. Key Concepts (Beginner-Friendly)

An index is where Splunk stores searchable events. The host field identifies the originating machine, while source tells you which file or channel supplied an event. One host can have several sources.

## 7. SOC / Real-World Relevance

Use a known event to verify the whole collection path. Once both hosts are reporting, a single search window can show failed logons from Windows alongside authentication activity from Ubuntu.

## 8. Quick Command Cheat Sheet

| Command | What It Does |
|---|---|
| `ip addr` | Shows the SOC machine's network addresses |
| `splunk list forward-server` | Displays configured forwarding destinations and their state |
| `index=main` | Searches events in the lab index |
| `index=main &#124; top source` | Ranks sources by event count |

## 9. Common Mistakes & Troubleshooting

- **Splunk Web opens but events are missing.** Verify the receiver on 9997 and the endpoint forwarding destination. Port 8000 serves the browser interface.
- **Search returns no test event.** Check its local log first, widen the search time range and use the hostname displayed in Splunk.
- **Configuration edit has no effect.** Check the stanza, file location and forwarder restart. Run example searches individually.

## 10. Conclusion

Local logging supplies the records that each forwarder collects.
A known test message verifies delivery and indexing.
Host and source fields make cross-platform investigations easier to reproduce.
