# Introduction to Wazuh

## 1. Lab Overview

Connect Ubuntu and Windows agents to a central Wazuh manager. Use the dashboard to trace authentication alerts back to the reporting endpoint.

## 2. Lab Environment

An Ubuntu SOC machine hosts the Wazuh manager and dashboard at https://localhost. A second Ubuntu machine and a Windows machine serve as agents. The lab also includes a Kali traffic-generation machine. Use the manager's reachable interface address in the agent configurations.

## 3. Workflow Diagram

Agent enrollment → service check → active-agent inventory → authentication alert → event review

## 4. Tools Used

| Tool | Purpose |
|---|---|
| Wazuh agent and manager | Collect endpoint telemetry and evaluate detection rules |
| Wazuh Dashboard | Explore agents, alert timelines and rule details |
| PowerShell and Windows Installer | Install and start the Windows service |

## 5. Task-by-Task Walkthrough

### T1 — Configure the Linux agent and investigate SSH alerts

**Goal.** Register the Ubuntu agent and identify its authentication alerts.

1. On the SOC machine use ip addr to identify the manager address. Replace <WAZUH_MANAGER_IP> in the following configuration command before running it on the Ubuntu agent.
2. Verify the server address in ossec.conf, start the agent and check its status.
3. Refresh the dashboard and open the active Ubuntu agent. Choose More → MITRE ATT&CK → Framework. Use the timeline and the option to hide techniques without alerts.
4. Inspect the lab's SSH authentication activity under Brute Force. Expand the rule for multiple failures followed by a success. Compare the account, source and times in the underlying records.

```bash
ip addr
sed -i 's/MANAGER_IP/<WAZUH_MANAGER_IP>/g' /var/ossec/etc/ossec.conf
cat /var/ossec/etc/ossec.conf
/etc/init.d/wazuh-agent start
/etc/init.d/wazuh-agent status
```

**Command explained.** sed -i edits the file in place. The substitution changes the supplied MANAGER_IP token to the manager's reachable IP. The init script starts the agent and reports its local state.

**Expected finding.** The dashboard should list the Ubuntu endpoint as active. A sequence of failures followed by success deserves investigation of whether access was authorized.

### T2 — Configure the Windows agent and review RDP events

**Goal.** Enroll the Windows endpoint and isolate its authentication events.

1. From an elevated PowerShell session open Desktop and run the supplied MSI. Substitute the SOC address in both installer properties.
2. Start WazuhSvc and identify the Windows endpoint's own address using ipconfig.
3. Refresh the agent inventory. Select the Windows agent, open More → MITRE ATT&CK → Events and inspect records for the lab's RDP authentication activity. Match the host and event time.

```powershell
cd Desktop
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER=<WAZUH_MANAGER_IP> WAZUH_REGISTRATION_SERVER=<WAZUH_MANAGER_IP> WAZUH_AGENT_GROUP=default
Start-Service -Name WazuhSvc
ipconfig
```

**Command explained.** /i installs the MSI and /q suppresses its user interface. WAZUH_REGISTRATION_SERVER selects the enrollment destination, WAZUH_MANAGER selects the reporting destination and WAZUH_AGENT_GROUP assigns the configuration group.

**Expected finding.** After enrollment, the inventory should contain both agents. Selecting the Windows agent narrows the timeline to that endpoint.

## 6. Key Concepts (Beginner-Friendly)

An agent reports what its endpoint observes. A manager evaluates those records. A rule can combine several records into one alert, such as repeated failures followed by a successful login.

## 7. SOC / Real-World Relevance

Use the alert as the starting point. Check whether the failures and success involve the same source, account and endpoint before escalating a possible compromise.

## 8. Quick Command Cheat Sheet

| Command | What It Does |
|---|---|
| `ip addr` | Register the Ubuntu agent and identify its authentication alerts. |
| `cd Desktop` | Enroll the Windows endpoint and isolate its authentication events. |

## 9. Common Mistakes & Troubleshooting

- **Agent absent.** Verify the manager address, service state and dashboard refresh. An endpoint's localhost address refers to itself.
- **Active agent with no matching alerts.** Check the selected agent, time window and whether authentication events occurred.
- **Quiet MSI installation.** The installer shows little feedback. Check that WazuhSvc exists and the agent appears in inventory.

## 10. Conclusion

Enrollment connects each endpoint to the manager.
The agent view identifies where each alert originated.
The authentication sequence helps an analyst decide what to investigate next.
