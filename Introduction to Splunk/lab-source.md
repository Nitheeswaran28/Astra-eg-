# Introduction to Splunk – SOC Lab

**Objective:** Configure Splunk to collect, monitor and analyze security logs from Ubuntu and Windows using Splunk Universal Forwarder.

**Lab environment:** SOC machine (Splunk server), Ubuntu endpoint, Windows endpoint. Port 8000 provides Splunk Web; port 9997 receives forwarded logs.

Ubuntu / Windows → Universal Forwarder → Splunk Server → Search & Analysis

## T1: Verify Splunk Server

→ Start all three lab machines.

→ On the SOC machine, open Firefox and visit `http://localhost:8000`.

→ Identify the server IP:

```bash
ip addr
```

**Explanation:** Displays network interfaces and IP addresses. The endpoints need the server IP to forward logs. Example: `10.0.21.229`.

**Finding:** Verify that Splunk Web is accessible and identify the server IP.

## T2: Enable Logging on Ubuntu

→ Check rsyslog:

```bash
systemctl status rsyslog
```

→ If inactive:

```bash
systemctl start rsyslog
systemctl enable rsyslog
```

**Explanation:** rsyslog collects system logs. `start` launches the service; `enable` configures automatic startup.

→ Verify log files:

```bash
ls -la /var/log/syslog
ls -la /var/log/auth.log
ls -la /var/log/kern.log
```

- syslog: General system activity.
- auth.log: Authentication, SSH and sudo activity.
- kern.log: Kernel messages.
- `ls -la`: Displays file details and permissions.

→ Enable and verify audit logging:

```bash
systemctl start auditd
systemctl enable auditd
systemctl status auditd
ls -la /var/log/audit/audit.log
```

**Explanation:** auditd records security audit events according to configured audit rules.

**Finding:** Verify system, authentication, kernel and audit logging.

## T3: Install Splunk Universal Forwarder on Ubuntu

→ Install the provided package:

```bash
cd Desktop
dpkg -i splunkforwarder-linux-amd64.deb
/opt/splunkforwarder/bin/splunk start --accept-license
```

→ Create administrator credentials when prompted.

**Explanation:** `dpkg -i` installs a Debian package. Universal Forwarder is a lightweight agent that sends logs to Splunk.

→ Configure the receiving server (replace the placeholder with the actual IP):

```bash
/opt/splunkforwarder/bin/splunk add forward-server <SPLUNK_IP>:9997
```

→ Add log files for monitoring:

```bash
/opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -index main
/opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -index main
/opt/splunkforwarder/bin/splunk add monitor /var/log/kern.log -index main
/opt/splunkforwarder/bin/splunk add monitor /var/log/audit/audit.log -index main
```

**Explanation:** `add monitor` watches a file; `-index main` selects the destination index.

→ Edit the configuration:

```bash
nano /opt/splunkforwarder/etc/apps/search/local/inputs.conf
```

→ Under the syslog monitor section:

```ini
[monitor:///var/log/syslog]
disabled = false
index = main
crcSalt = <SOURCE>
```

**Explanation:** `crcSalt = <SOURCE>` includes the source path in the file identity calculation, preventing files with similar initial content from being mistaken for duplicates. Use carefully with rotating logs because renamed files may be ingested again.

→ Save with Ctrl+O, Enter, Ctrl+X. Restart:

```bash
/opt/splunkforwarder/bin/splunk restart
```

**Finding:** Ubuntu forwarder is configured to monitor four log files.

## T4: Configure Splunk to Receive Logs

→ SOC machine → Splunk Web → Settings → Forwarding and receiving → Configure receiving → New Receiving Port.

→ Enter `9997` and save.

**Explanation:** The server listens on this port for forwarded data.

**Finding:** Verify that Splunk is configured to receive logs.

## T5: Generate and Verify Ubuntu Test Events

→ Generate a failed SSH login using an incorrect password, a sudo event and a custom message:

```bash
ssh invaliduser@localhost
sudo ls /root
logger "Splunk Test Event: This is a test log entry from Ubuntu Machine"
tail -5 /var/log/auth.log
tail -5 /var/log/syslog
```

**Explanation:** SSH produces authentication events; sudo records privileged activity; logger writes a test message; tail displays the last five lines.

→ SOC machine → Splunk Web → Search & Reporting.

```spl
index=main host=<ubuntu-hostname>
```

→ Search specific events (replace the example hostname):

```spl
index=main host=ip-10-0-28-202 "invaliduser"
index=main host=ip-10-0-28-202 "ls /root"
index=main host=ip-10-0-28-202 "Splunk Test Event"
```

**Explanation:** SPL (Search Processing Language) searches and filters collected events.

**Finding:** Verify that Ubuntu test events appear in Splunk.

## T6: Enable Security Logging on Windows

→ Win+R → `eventvwr.msc` → Windows Logs.

→ Verify Application, Security, System and Setup logs.

→ Win+R → `secpol.msc` → Security Settings → Local Policies → Audit Policy.

→ Enable Success and Failure for account logon events, account management, logon events, process tracking and system events, as specified by the lab.

**Explanation:** Audit policies determine which successful and failed security activities Windows records.

→ Win+R → `gpedit.msc` → Computer Configuration → Administrative Templates → Windows Components → Windows PowerShell → Turn on PowerShell Script Block Logging → Enabled → OK.

**Explanation:** Script Block Logging records PowerShell script content for investigation.

**Finding:** Verify Windows auditing and PowerShell logging.

## T7: Install Splunk Universal Forwarder on Windows

→ Open the provided MSI installer on Desktop → Accept license → Create credentials → Leave Deployment Server blank.

→ Receiving Indexer: enter actual Splunk IP and port `9997` → Install → Finish.

→ Open Command Prompt as Administrator:

```cmd
notepad "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"
```

→ Add:

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

**Explanation:** WinEventLog identifies the channel; index selects storage; disabled=false enables collection; renderXml=false uses rendered event text.

→ Save and verify connection, then restart:

```cmd
cd "C:\Program Files\SplunkUniversalForwarder\bin"
splunk list forward-server
splunk restart
```

**Finding:** Windows forwarder is configured to collect Security, System, Application and PowerShell logs.

## T8: Generate Windows Security Test Events

→ Generate a failed login (enter a password when prompted):

```cmd
runas /user:fakeuser cmd
```

**Explanation:** An invalid account authentication attempt can generate Security Event ID 4625 (failed logon).

→ Generate service events in Administrator Command Prompt:

```cmd
net stop spooler
net start spooler
```

**Explanation:** Stops and restarts Print Spooler, generating service-related events.

→ Generate a custom event in PowerShell:

```powershell
Write-EventLog -LogName Application -Source "Application" -EventId 9999 -EntryType Information -Message "Splunk Test Event: This is a test log entry from Windows Machine"
```

**Explanation:** Writes a test Application event if the event source is registered.

→ SOC machine → Search & Reporting. Search each query separately:

```spl
index=main host="ATTACKDEFENSE"
index=main "Splunk Test Event" host="ATTACKDEFENSE"
index=main host="ATTACKDEFENSE" *spooler*
index=main EventCode=4625 host="ATTACKDEFENSE" "fakeuser"
```

**Finding:** Verify Windows test events in Splunk, confirming log ingestion.

## T9: Explore the Splunk Dashboard

→ Search & Reporting → View all events:

```spl
index=main
```

→ Click host in the Fields sidebar to identify monitored machines; click source to identify log types.

→ Visualize event counts over time:

```spl
index=main | timechart count by host
```

→ Click Visualization. This helps identify spikes in activity.

→ Identify top log sources:

```spl
index=main | top source
```

→ Search authentication-related events across both machines:

```spl
index=main (source="/var/log/auth.log" OR source="WinEventLog:Security")
```

**Finding:** Splunk can search, filter and visualize logs from both endpoints.

## Conclusion

This lab demonstrates centralized security monitoring: enabling Ubuntu and Windows logging, configuring Universal Forwarders, receiving logs on port 9997, generating test events and analyzing them using SPL.

**SOC relevance:** Collect endpoint logs, investigate failed logins, monitor system activity and identify potential incidents.

**Tools used:** Splunk Enterprise, Splunk Universal Forwarder, Ubuntu, Windows Event Viewer, rsyslog, auditd, PowerShell, Command Prompt and Firefox.

**Note:** IP addresses and hostnames are examples. Use actual lab values. Findings are expected results to verify in the running lab. Linux commands may require elevated privileges.
