# Recovering Key Git Repo Files

## 1. Lab Overview

A web deployment exposed internal Git files alongside a Flask application. The INE exercise follows that exposure from repository metadata to application source and host access. This guide uses its five flag prompts as investigation questions for a defender reviewing a preserved deployment and trusted repository copy. [INE lab](https://my.ine.com/labs/1ddd4301-e570-3783-ad2e-0eaff4f8bd8e)

## 2. Lab Environment

The lab provides a Kali desktop at `192.X.Y.2` and a target web server at `192.X.Y.3`. The manual reports TCP services on ports 5000 and 8000 and identifies Python files named `API.py` and `utils.py`. The gateway at `192.X.Y.1` is outside the exercise. The local Git command below is an analyst-side verification example for a trusted clone. Findings describe the manual's scenario.

## 3. Workflow Diagram

Deployment contents → repository history → credential exposure → Flask signing key → application and host evidence

## 4. Tools Used

| Tool or evidence | Purpose |
|---|---|
| Git | Group commits by author in a trusted local copy |
| Deployment artifact | Identify repository metadata shipped with the application |
| Python source review | Trace how the application handles secrets, filenames and expressions |
| Web and host logs | Correlate requests with application processes and file access during incident response |

## 5. Task-by-Task Walkthrough

### T1 — Establish what the web deployment exposed

**Goal.** Determine whether internal repository data was present in the served application.

1. Inspect the preserved deployment manifest or a local copy of the deployed directory. Identify the web root and check whether it contains Git metadata.
2. Compare the included files with the intended public release. The manual identifies a marker file inside the Git directory.
3. In retained web logs, distinguish requests for metadata from successful responses. Record the time, status and response size.

**Expected finding.** The lab demonstrates that a repository marker was readable. Record its location and the evidence of access. A public `.gitignore` file by itself establishes only that this file was served.

### T2 — Identify the most frequent commit author

**Goal.** Use commit history to answer the attribution question without publishing a personal address.

1. In a trusted local copy, select the branch and revision corresponding to the deployed snapshot.
2. Run the following read-only summary. Record the leading count and retain the address in the private case record if needed.

```bash
git shortlog -sne HEAD
```

**Command explained.** `-s` summarizes counts, `-n` sorts them numerically and `-e` includes the recorded email address. `HEAD` limits the input to history reachable from the checked-out revision.

**Expected finding.** The manual attributes the commits to one author identity. Git author fields are supplied with the commit. They do not authenticate the person who used an account or accessed the server.

### T3 — Assess the exposed account credential

**Goal.** Identify the affected account and bound the exposure window.

1. The manual identifies a credential in repository configuration. Record the configuration file and affected service in the incident record, keeping the value out of tickets and screenshots.
2. Determine when the artifact first became accessible. Review that service's authentication records from deployment through credential rotation.
3. Revoke or rotate the credential through its owner and verify dependent services use the replacement.

**Expected finding.** A readable credential establishes exposure. Authentication records are needed to establish use. Removing a value from the current source tree leaves previously deployed copies and historical versions to investigate.

### T4 — Assess the Flask signing key

**Goal.** Explain the impact of revealing the application's `SECRET_KEY`.

1. Review the trusted source copy for where the application loads its signing key. Record the file and revision and identify which deployments used that value.
2. Determine which features trust the key. With Flask's default signed-cookie sessions, knowledge of the key can undermine confidence in session contents.
3. Coordinate key rotation across application instances and invalidate affected sessions. Store the replacement outside source control.

**Expected finding.** The manual identifies the key in application source. Its exposure is separate from the account credential in T3 and needs its own remediation and verification.

### T5 — Assess the path from application exposure to host access

**Goal.** Determine what evidence supports the final host-access finding.

1. Review the file-serving and calculator code in the trusted source. Identify where user-controlled filenames or expressions enter these features and what the application permits them to reach.
2. For an incident, align relevant web requests with application errors, child-process records, network connections and file-access events. Record timestamps and the identity of the application process.
3. Restrict file access to the intended directory after canonical path resolution. Parse only supported arithmetic operations. Run the service with the minimum filesystem permissions it needs.
4. Verify the fixes with regression checks for out-of-directory requests and unsupported expressions. Confirm normal file retrieval and arithmetic still work.

**Expected finding.** The training manual demonstrates access to a marker on the target host. In a real case, vulnerable code establishes a possible route. Correlated host evidence establishes which actions actually occurred.

## 6. Key Concepts (Beginner-Friendly)

A deployed application needs its runtime files. A Git repository can also contain history and local configuration. Publishing both gives visitors access to information the application never needed to display. A password authenticates an account, while a Flask signing key lets the application verify data it previously signed.

## 7. SOC / Real-World Relevance

Build a timeline with separate entries for deployment, repository access, credential use and host activity. Each has a different evidence source and owner. Preserve the affected release before replacing it with a clean build artifact.

## 8. Quick Command Cheat Sheet

| Command | What It Does |
|---|---|
| `git shortlog -sne HEAD` | Counts commits by recorded author and email for the checked-out history |
| `ip a` | Shows the interface addresses assigned to the Kali lab desktop |

## 9. Common Mistakes & Troubleshooting

- **Author counts differ.** Confirm the snapshot and branch. A shallow clone may omit older commits.
- **A request appears in an access log.** Inspect the response status, size and application behavior. A generic error page can also return HTTP 200.
- **A deployment fix seems sufficient.** Check cached artifacts, rotate exposed credentials and signing keys, and review activity during the exposure window.
- **Host telemetry is missing.** Record the gap and retain the application-side evidence. A request timestamp alone cannot establish shell execution.

## 10. Conclusion

Repository metadata can expose both development history and operational secrets.
Account credentials and signing keys require separate containment actions.
Web and host evidence help distinguish disclosure from subsequent compromise.
