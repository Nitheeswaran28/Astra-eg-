# DNS Basics

## 1. Lab Overview

Use two DNS servers to see how the answer to a lookup depends on the name, record type and server. Read individual records, compare resolver responses and interpret the evidence of an overly permissive zone transfer. The examples come from the [INE lab](https://my.ine.com/labs/94797a41-006a-3a6b-ad33-b008c7dd6af9).

## 2. Lab Environment

INE provides a Kali desktop at `192.X.Y.2` and two DNS servers at `192.X.Y.3` and `192.X.Y.4`. Read X and Y from the current lab interface. The gateway at `192.X.Y.1` is outside the exercise. Use the example domain names below only with the lab's resolver.

## 3. Workflow Diagram

Identify interface → check DNS service → identify default resolver → query named records → compare answers and transfer policy

## 4. Tools Used

| Tool | Purpose |
|---|---|
| ip | Identify the active lab interface |
| Nmap | Interpret the service check for the two assigned servers |
| nslookup | Request a particular record or select a different server |
| dig | Inspect status, answer sections, TTL and DNSSEC data |

## 5. Task-by-Task Walkthrough

### T1 — Identify the assigned subnet

1. Inspect Kali's interfaces.
2. Find the IPv4 address ending in .2 and use its subnet to identify the two assigned DNS servers.

```bash
ip a
```

**Check.** Read the `inet` line on the lab interface. The loopback address refers only to Kali.

### T2 — Check the DNS service

1. Locate the two server entries in the manual's Nmap results.
2. Read the port, transport and state together.

**Expected finding.** The manual reports port 53 open. A TCP service check establishes reachability on that transport. The next steps test whether the server can answer the intended DNS question.

### T3 — Identify the default resolver

1. Read the resolver configuration.
2. Record the nameserver address so you can explain which server answers queries that omit an explicit destination.

```bash
cat /etc/resolv.conf
```

**Check.** The `nameserver` entry identifies where this client sends the query. That server can resolve a zone on the client's behalf.

### T4 — Compare records with nslookup

1. Replace `<DNS_SERVER_IP>` with the first assigned server address. Use the reverse lookup to identify names associated with that address.
2. Query individual record types. Compare an IPv4 address, descriptive text, an alias, a mail destination and a DNSSEC signature.
3. In the MX example the final argument selects the answering server. Compare that with using the default resolver.

```bash
nslookup <DNS_SERVER_IP>
nslookup -type=A witrapper.com
nslookup -type=TXT witrap.com
nslookup -type=CNAME labs.witrap.com
nslookup -type=MX witrap.com ns3.witrapper.com
nslookup -type=DNSKEY witrapper.com.
nslookup -type=RRSIG witrapper.com.
nslookup -debug promo.witrap.com
```

**Command explained.** `-type` selects the question. DNSKEY supplies public key material and RRSIG supplies a signature over a record set. `-debug` adds details about the exchange.

**Expected finding.** In the manual, the alias points to `witrapper.com`. The RRSIG caption and command name different domains. The example above preserves the command's signed domain.

### T5 — Read dig responses and interpret zone exposure

1. Compare the full response with the compact answer. In the full response identify the question, status, answering server and TTL.
2. Check the selected records below. For MX, smaller preference numbers take precedence. For SOA, identify the primary server and zone serial. For SRV, read the service target and port.
3. Compare the same name on the second assigned DNS server. Replace `<SECOND_DNS_IP>` before running the query.
4. Review the manual's transfer result for `witrap.com`. It shows multiple types and names returned together. Compare the requesting client with the zone's list of approved secondary servers when assessing this exposure.

```bash
dig witrap.com
dig A witrap.com +short
dig AAAA witrap.com
dig MX witrapper.com
dig SOA promo.witrap.com
dig SOA witrapper.com
dig NS witrap.com
dig TXT promo.witrap.com
dig CAA witrapper.com
dig CNAME labs.witrap.com +short
dig +dnssec witrapper.com +short
dig +noall +answer +question witrap.com
nslookup -type=srv _ldap._tcp.witrapper.com
dig ldap.witrapper.com
dig @<SECOND_DNS_IP> witrap.com
```

**Command explained.** `@server` selects the DNS server. `+short` hides most context. `+noall +answer +question` shows the query and its answer together. `+dnssec` requests DNSSEC data. Signature validation is a separate operation.

**Expected finding.** An individual lookup describes one name and type. The manual's AXFR result disclosed a zone-wide inventory including aliases and service-related records.

## 6. Key Concepts (Beginner-Friendly)

A DNS zone is a set of names managed together. A resolver asks questions about those names. A secondary server uses zone transfers to keep a copy current. The TTL tells a resolver how long it can cache a record.

## 7. SOC / Real-World Relevance

For an unexpected answer, record the time, queried name, record type, resolver and response status. For a transfer alert, check which client requested the transfer, whether the server allowed it and whether that client is an approved secondary.

## 8. Quick Command Cheat Sheet

| Command | What It Does |
|---|---|
| `ip a` | Shows the address assigned to Kali |
| `dig NS witrap.com` | Requests the zone's name servers |
| `dig MX witrapper.com` | Shows mail exchangers and preference values |
| `dig +noall +answer +question witrap.com` | Keeps the question beside its answer |

## 9. Common Mistakes & Troubleshooting

- **Two similar names give different answers.** The manual uses both `witrap.com` and `witrapper.com`. Check the spelling before comparing results.
- **Blank short output.** Repeat the full lookup and read the status. An existing name can lack the requested record type.
- **An ANY query returns little data.** Servers can limit ANY responses. Ask for the specific type you need.
- **Conflicting timeout syntax.** The manual's timeout examples omit the leading hyphen from the port option. Its separate port example uses `-port=53`.

## 10. Conclusion

The record type determines what question DNS answers.
The server, status and TTL give that answer context.
The lab's transfer example shows why secondary-server permissions deserve a separate review.
