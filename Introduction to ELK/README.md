# Introduction to ELK

## 1. Lab Overview

Collect Linux files and Windows event channels in Elasticsearch, then use Kibana to follow a test event from each endpoint. Finish with a chart that compares reporting activity across the two machines.

## 2. Lab Environment

The SOC machine runs Elasticsearch on port 9200 and Kibana on 5601. The Ubuntu desktop contains the Filebeat Debian package. The Windows desktop contains the Winlogbeat archive. Use the SOC machine's reachable address for <ELK_IP>.

## 3. Workflow Diagram

Ubuntu → Filebeat ↘ Elasticsearch → Kibana Discover → Lens

Windows → Winlogbeat ↗

## 4. Tools Used

| Tool | Purpose |
|---|---|
| rsyslog / auditd | Write Ubuntu system and audit records |
| Filebeat / Winlogbeat | Ship Linux files and Windows events |
| Elasticsearch | Index events for search |
| Kibana Discover / Lens | Investigate records and chart activity |

## 5. Task-by-Task Walkthrough

### T1 — Verify the SOC machine

**Goal.** Check the two central services.

1. Inspect both service states on the SOC machine.
2. Open Kibana at http://localhost:5601 on that machine. Identify the network address reachable by both endpoints.

```bash
systemctl status elasticsearch
systemctl status kibana
ip addr
```

**Command explained.** systemctl status shows service state. ip addr lists interface addresses.

**Expected finding.** Both services should be active before testing endpoint delivery.

### T2 — Enable Ubuntu logging

**Goal.** Confirm the endpoint has events to ship.

1. Check rsyslog. Start and enable it if necessary.
2. Start auditd and verify the expected files exist.

```bash
systemctl status rsyslog
systemctl start rsyslog
systemctl enable rsyslog
systemctl start auditd
systemctl enable auditd
systemctl status auditd
ls -la /var/log/syslog
ls -la /var/log/auth.log
ls -la /var/log/kern.log
ls -la /var/log/audit/audit.log
```

**Command explained.** start acts immediately and enable configures startup. ls -la shows file details.

**Expected finding.** The four paths supply general, authentication, kernel and audit events.

### T3 — Configure Filebeat

**Goal.** Ship four Ubuntu files to Elasticsearch.

1. Install the desktop package. Edit filebeat.yml and replace the corresponding input and destination sections using the configuration below.
2. Give each filestream input its own ID. Replace <ELK_IP> with the SOC address.
3. Test configuration and output before starting and enabling the service.

```bash
cd Desktop
dpkg -i filebeat-9.0.0-amd64.deb
nano /etc/filebeat/filebeat.yml
```

**Command explained.** dpkg -i installs a local Debian package. Each filestream input follows the file listed under paths.

```yaml
filebeat.inputs:
  - type: filestream
    id: syslog
    enabled: true
    paths:
      - /var/log/syslog
  - type: filestream
    id: authlog
    enabled: true
    paths:
      - /var/log/auth.log
  - type: filestream
    id: kernlog
    enabled: true
    paths:
      - /var/log/kern.log
  - type: filestream
    id: auditlog
    enabled: true
    paths:
      - /var/log/audit/audit.log
output.elasticsearch:
  hosts: ["<ELK_IP>:9200"]
setup.kibana:
  host: "<ELK_IP>:5601"
```

```bash
filebeat test config
filebeat test output
systemctl start filebeat
systemctl enable filebeat
systemctl status filebeat
```

**Expected finding.** A configuration pass checks syntax. The output test checks the connection to Elasticsearch.

### T4 — Generate Ubuntu events

**Goal.** Produce identifiable test records.

1. Attempt a local SSH sign-in with the supplied invalid username. Run the sudo command and write a distinctive test message.
2. Compare recent records with the actions you just performed.

```bash
ssh invaliduser@localhost
sudo ls /root
logger "ELK Test Event: This is a test log entry from Ubuntu Machine"
tail -5 /var/log/auth.log
tail -5 /var/log/syslog
```

**Command explained.** logger emits a message to local logging. tail -5 displays the last five lines.

**Expected finding.** Confirm the test message locally before searching Kibana.

### T5 — Search Ubuntu records

**Goal.** Locate the Ubuntu tests in Discover.

1. Create the filebeat-* index pattern and select @timestamp as the time field.
2. Choose that pattern in Discover. Replace the hostname below and run each search separately.

```kql
host.name: "<ubuntu-hostname>"
host.name: "<ubuntu-hostname>" and message: "invaliduser"
host.name: "<ubuntu-hostname>" and message: "ls /root"
host.name: "<ubuntu-hostname>" and message: "ELK Test Event"
```

**Command explained.** host.name scopes the endpoint and message narrows the matching text.

**Expected finding.** A matching event provides an indexed copy to compare with the local record.

### T6 — Enable Windows event logging

**Goal.** Prepare Windows security and operational channels.

1. Open Event Viewer and inspect Windows Logs. In Local Security Policy enable success and failure auditing for the lab's account logon, account management, logon, process tracking and system event categories.
2. In Group Policy Editor open Computer Configuration → Administrative Templates → Windows Components → Windows PowerShell and enable script block logging.

```cmd
eventvwr.msc
secpol.msc
gpedit.msc
```

**Command explained.** These consoles open the event records, audit policy and group policy settings respectively.

**Expected finding.** The endpoint can now record the selected activity for Winlogbeat.

### T7 — Configure Winlogbeat

**Goal.** Send the four Windows channels to Elasticsearch.

1. Extract the desktop ZIP under C:\Program Files\Winlogbeat. Open elevated PowerShell in the extracted folder.
2. Edit the channel and destination sections, substitute the SOC IP and test both settings.
3. Install the supplied service and check its state.

```powershell
cd "C:\Program Files\Winlogbeat\winlogbeat-9.0.0-windows-x86_64"
notepad winlogbeat.yml
```

**Command explained.** The event_logs list chooses the precise channel names. Elasticsearch receives records while the Kibana setting identifies the UI endpoint.

```yaml
winlogbeat.event_logs:
  - name: Application
  - name: Security
  - name: System
  - name: Microsoft-Windows-PowerShell/Operational
output.elasticsearch:
  hosts: ["<ELK_IP>:9200"]
setup.kibana:
  host: "<ELK_IP>:5601"
```

```powershell
.\winlogbeat.exe test config
.\winlogbeat.exe test output
.\install-service-winlogbeat.ps1
Start-Service winlogbeat
Get-Service winlogbeat
```

**Expected finding.** Successful tests and a running service establish readiness. Search for a specific event to verify ingestion.

### T8 — Generate Windows events

**Goal.** Produce three recognizable Windows test signals.

1. In the lab VM use Command Prompt for the invalid account attempt and an elevated prompt for the Spooler stop/start.
2. Use Windows PowerShell to write the Application event shown below.

```cmd
runas /user:fakeuser cmd
net stop spooler
net start spooler
```

**Command explained.** runas triggers authentication. Restarting the service supplies System activity. The following command writes a labeled Application event using a registered source.

```powershell
Write-EventLog -LogName Application -Source "Application" -EventId 9999 -EntryType Information -Message "ELK Test Event: This is a test log entry from Windows Machine"
```

**Expected finding.** Look for failed logon 4625, Spooler activity and the custom event 9999 in their respective channels.

### T9 — Verify Windows records

**Goal.** Identify the Windows host and its test events.

1. Create the winlogbeat-* pattern with @timestamp.
2. Select the pattern in Discover. Replace the hostname and run the queries separately.

```kql
host.name: "<windows-hostname>"
host.name: "<windows-hostname>" and message: "ELK Test Event"
host.name: "<windows-hostname>" and message: "spooler"
host.name: "<windows-hostname>" and winlog.event_id: "4625" and message: "fakeuser"
```

**Command explained.** The event ID narrows the last search to failed logons. The username selects the specific test account.

**Expected finding.** Compare timestamps and channel names with the local events.

### T10 — Explore both endpoints

**Goal.** Compare event volume and investigate authentication across hosts.

1. Create the combined *beat-* pattern with @timestamp.
2. In Visualize Library → Create visualization → Lens, build a bar chart with @timestamp on the horizontal axis, record count on the vertical axis and host.name as the breakdown. Save the chart.
3. In Discover select the same pattern and apply the query.

```kql
log.file.path: "/var/log/auth.log" or winlog.channel: "Security"
```

**Command explained.** The or operator matches either the Linux authentication file or the Windows Security channel.

**Expected finding.** The chart separates volume by host. The query brings both evidence sources into one view for investigation.

## 6. Key Concepts (Beginner-Friendly)

A Beat ships records. An index stores them. A Kibana index pattern selects which stored records Discover can search. The @timestamp field aligns records on a timeline.

## 7. SOC / Real-World Relevance

Compare operating systems during authentication triage. Verify the time window and reporting hosts before interpreting a quiet period as an absence of activity.

## 8. Quick Command Cheat Sheet

| Command | What It Does |
|---|---|
| `systemctl status elasticsearch` | Check the two central services. |
| `systemctl status rsyslog` | Confirm the endpoint has events to ship. |
| `cd Desktop` | Ship four Ubuntu files to Elasticsearch. |
| `ssh invaliduser@localhost` | Produce identifiable test records. |
| `host.name: "<ubuntu-hostname>"` | Locate the Ubuntu tests in Discover. |
| `eventvwr.msc` | Prepare Windows security and operational channels. |
| `cd "C:\Program Files\Winlogbeat\winlogbeat-9.0.0-windows-x86_64"` | Send the four Windows channels to Elasticsearch. |
| `runas /user:fakeuser cmd` | Produce three recognizable Windows test signals. |
| `host.name: "<windows-hostname>"` | Identify the Windows host and its test events. |
| `log.file.path: "/var/log/auth.log" or winlog.channel: "Security"` | Compare event volume and investigate authentication across hosts. |

## 9. Common Mistakes & Troubleshooting

- **Configuration test fails.** Check YAML indentation and avoid duplicate top-level sections.
- **Output test fails.** Verify the SOC address and Elasticsearch port 9200 from the endpoint.
- **Discover looks empty.** Select the appropriate pattern, widen the time picker and compare with a locally confirmed test event.

## 10. Conclusion

Local tests establish what each endpoint recorded.
Beat checks validate the shipping configuration.
Discover and Lens make the combined evidence searchable and comparable.
