# Wazuh SIEM Home Lab: Detecting an SSH Brute-Force Attack

A hands-on SOC lab where I built a Wazuh SIEM, attacked a monitored Ubuntu server with Hydra from Kali Linux, and wrote a custom detection rule that escalates persistent SSH brute-force activity to a high-severity alert mapped to MITRE ATT&CK T1110.

**Full write-up:** [report/incident-report.md](report/incident-report.md)

## What this lab demonstrates

- Deploying a Wazuh server and enrolling an Ubuntu agent
- Simulating a real attack (reconnaissance with Nmap, brute force with Hydra)
- Reading and triaging SIEM alerts in the Wazuh dashboard (Threat Hunting, Events, alert details)
- Writing a custom correlation rule (ID 100100, level 12) and confirming it fires
- Mapping detections to MITRE ATT&CK and building an evidence-based timeline
- Documenting an incident the way a SOC analyst would

## Architecture

| Role | Machine | Details |
|---|---|---|
| SIEM | Wazuh server VM | VMware Workstation, bridged network |
| Victim | Ubuntu Server 24.04 VM | Wazuh agent 001 (`victim-ubuntu`), SSH enabled |
| Attacker | Kali Linux | IP redacted, Nmap and Hydra |

```
 Kali (attacker)  --- SSH brute force --->  Ubuntu victim (Wazuh agent)
                                                    |
                                           logs via journald
                                                    v
                                              Wazuh server
                                       (rules, alerts, dashboard)
```

## Attack summary

1. `nmap -sV` found only SSH (OpenSSH 9.6p1) open on the victim.
2. `hydra -l victor -P passwords.txt ssh://<victim-ip> -t <tasks>` was run several times against one account.
3. All attempts failed, with 0 valid passwords found.

## Detection results

| Date | Result | Evidence |
|---|---|---|
| Oct 20, 2026 | Built-in rules caught the attack: 53 failed-login alerts (rule 5760, level 5), plus rules 5763 and 5551 at level 10. No level 12 alerts | [Screenshot 05](screenshots/05-dashboard-53-auth-failures.png), [07](screenshots/07-level10-alerts-rules-5551-5763.png) |
| Oct 21, 2026 | Custom rule **100100** (level 12, T1110) fired 3 times, at 16:56:37, 16:57:03 and 17:24:15 | [Screenshot 12](screenshots/12-custom-rule-100100-fired-twice.png), [19](screenshots/19-rule-100100-third-fire.png) |

Wazuh recorded **0 authentication successes** throughout, so the attack failed.

### Custom rule 100100

- **Description:** Persistent SSH brute force: repeated brute-force alerts from the same source IP
- **Level:** 12, with email alerting enabled
- **MITRE ATT&CK:** T1110 Brute Force (Credential Access)

Details are in [rules/rule-100100.md](rules/rule-100100.md).

## Repository structure

```
.
├── README.md
├── report/
│   └── incident-report.md      Full incident report with timeline
├── rules/
│   └── rule-100100.md          Custom rule details
└── screenshots/                22 numbered screenshots in timeline order
```

## Key takeaways

- Built-in rules detect brute force well, but a custom rule helps prioritise a persistent attacker over a one-off failed login.
- Correlating alerts by source IP turns many low-level events into one actionable alert.
- Time zones matter in investigations: the dashboard shows IST while the victim's raw logs are in UTC.

## Disclaimer

All attacks were performed in my own home lab against machines I own, for learning purposes only. The attacker IP and MAC addresses are redacted, and the remaining addresses are private lab addresses.
