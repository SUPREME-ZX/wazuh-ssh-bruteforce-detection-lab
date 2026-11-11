# Incident Report: SSH Brute-Force Attack Detected with Wazuh

| | |
|---|---|
| **Report type** | Detection and analysis (home lab exercise) |
| **Incident dates** | October 20 and 21, 2026 |
| **Time zone** | IST (UTC+5:30) as shown in the Wazuh dashboard and Kali terminal. Raw victim logs are in UTC, 5 hours 30 minutes behind |
| **Severity** | High (level 12 custom alert) |
| **Outcome** | Attack detected and escalated. No successful login |
| **MITRE ATT&CK** | T1110 Brute Force, tactic Credential Access |

---

## 1. Executive summary

A Hydra SSH brute-force attack was launched from a Kali Linux machine (IP redacted) against the account `victor` on an Ubuntu Server 24.04 victim monitored by a Wazuh SIEM.

On October 20, Wazuh's built-in rules detected the attack, producing 53 authentication-failure alerts in under two minutes. The highest built-in severity was level 10, so nothing reached the level 12 band that the dashboard tracks as high priority.

On October 21, a custom rule (ID 100100, level 12) was in place. It fired three times, each time that repeated brute-force alerts came from the same source IP, and mapped the activity to MITRE T1110. Every Hydra run failed to find a valid password, and Wazuh recorded 0 authentication successes throughout, so the attack did not succeed.

## 2. Environment

| Component | Details |
|---|---|
| SIEM | Wazuh server (manager name `wazuh`, cluster node `node01`), VM on VMware Workstation, bridged network |
| Victim | Ubuntu Server 24.04.5 LTS, hostname `ubuntu-victim`, Wazuh agent name `victim-ubuntu` (ID 001, version 4.14.9, group `default`), status Active |
| Victim IP | 192.168.1.106 on Oct 20, 192.168.1.102 on Oct 21 (the address changed between sessions) |
| Attacker | Kali Linux (IP redacted) |
| Target service | OpenSSH 9.6p1 (Ubuntu 3ubuntu13.20) on port 22, the only open port found by Nmap |
| Target account | `victor` |
| Log source | `journald` on the victim, collected by the Wazuh agent |

## 3. Attack methodology

1. **Reconnaissance:** `nmap -sV` against the victim showed 999 closed ports and SSH open on 22/tcp.
2. **Credential attack:** Hydra was run against the single account `victor` with a six-entry password list, trying different parallel task counts (`-t 4`, `-t 5`, `-t 6`, `-t 10`):

```bash
hydra -l victor -P passwords.txt ssh://<victim-ip> -t <tasks>
```

Every run ended with "0 valid password found".

## 4. Timeline

### October 20, 2026

| Time (IST) | Event | Evidence |
|---|---|---|
| ~16:39 | Wazuh overview before the agent is registered ("no agents registered") | [01](../screenshots/01-wazuh-overview-no-agents.png) |
| 17:27 | Nmap service scan of 192.168.1.106: only SSH (OpenSSH 9.6p1) open | [02](../screenshots/02-nmap-recon-oct20.png) |
| 17:00–17:30 | Baseline window for agent 001: 1 sshd event, 0 authentication failures | [03](../screenshots/03-sshd-baseline-before-attack.png) |
| 17:38:31 | Hydra run 1 (`-t 5`) starts, finishes 17:38:38, no password found | [04](../screenshots/04-hydra-attack-oct20.png) |
| 17:38:35 | Rule 5551 fires (PAM: multiple failed logins in a small period of time), level 10 | [07](../screenshots/07-level10-alerts-rules-5551-5763.png) |
| 17:38:37 | Rule 5763 fires (sshd: brute force trying to get access to the system), level 10 | [07](../screenshots/07-level10-alerts-rules-5551-5763.png) |
| 17:39:37 | Hydra run 2 (`-t 10`) starts, finishes 17:39:42 | [04](../screenshots/04-hydra-attack-oct20.png) |
| 17:39:43 | Rule 5763 fires again | [08](../screenshots/08-alert-details-rule-5763-panel.png), [09](../screenshots/09-alert-details-rule-5763-full.png) |
| 17:39:51 | Hydra run 3 (`-t 4`) starts, finishes 17:39:59 | [04](../screenshots/04-hydra-attack-oct20.png) |
| 17:39:55 | Rule 5551 fires again | [07](../screenshots/07-level10-alerts-rules-5551-5763.png) |
| 17:30–18:00 | Window totals: 53 authentication failures (rule 5760, level 5), 4 level 10 alerts, 0 level 12 alerts, 0 successes | [05](../screenshots/05-dashboard-53-auth-failures.png), [06](../screenshots/06-events-rule-5760-auth-failed.png) |

### October 21, 2026

| Time (IST) | Event | Evidence |
|---|---|---|
| 16:54:17–16:57:02 | Four Hydra runs against 192.168.1.102 (`-t 4` three times, `-t 6` once), all with 0 valid passwords | [11](../screenshots/11-hydra-attack-oct21-first-runs.png) |
| 16:56:37 | **Custom rule 100100 fires (first time)**, level 12 | [12](../screenshots/12-custom-rule-100100-fired-twice.png) |
| 16:57:03 | **Custom rule 100100 fires (second time)**, `firedtimes` = 2 | [13](../screenshots/13-rule-100100-details-source-log.png), [14](../screenshots/14-rule-100100-details-mitre.png) |
| 16:30–17:00 | Window totals: 74 alerts, 2 at level 12 or above, 60 authentication failures, 0 successes | [15](../screenshots/15-dashboard-74-alerts-2-level12.png) |
| 17:22:20 and 17:24:11 | Two further Hydra runs (`-t 6`), 0 valid passwords | [16](../screenshots/16-hydra-attack-oct21-second-runs.png) |
| 17:24:15 | **Custom rule 100100 fires (third time)**, `firedtimes` = 3, alongside rules 5760, 5763 and 5503 | [18](../screenshots/18-events-24-hits-rule-100100.png), [19](../screenshots/19-rule-100100-third-fire.png), [20](../screenshots/20-rule-100100-third-details-source-log.png), [21](../screenshots/21-rule-100100-third-details-mitre.png) |
| ~17:25 | Endpoints page: agent 001 `victim-ubuntu` still Active (Ubuntu 24.04.5 LTS, 192.168.1.102, v4.14.9) | [10](../screenshots/10-endpoints-agent-active.png) |
| 17:00–17:30 | Window totals: 24 alerts, 1 at level 12 or above, 23 authentication failures, 0 successes | [17](../screenshots/17-dashboard-24-alerts-1-level12.png) |
| 17:43 | Nmap re-scan of 192.168.1.102: still only SSH open | [22](../screenshots/22-nmap-recon-oct21.png) |

## 5. Detection analysis

### Built-in Wazuh rules observed

| Rule ID | Level | Description | Role in this incident |
|---|---|---|---|
| 5760 | 5 | sshd: authentication failed | One alert per failed attempt. The raw volume signal |
| 5503 | 5 | PAM: User login failed | PAM-side record of each failed login |
| 5763 | 10 | sshd: brute force trying to get access to the system | Fires when failures from one source pile up |
| 5551 | 10 | PAM: multiple failed logins in a small period of time | Corroborates the pattern through PAM |

### Custom rule 100100

| Field | Value |
|---|---|
| Rule ID | 100100 |
| Description | Persistent SSH brute force: repeated brute-force alerts from the same source IP |
| Level | 12 |
| Frequency | 2 |
| Groups | `local`, `sshd` |
| Email alert | Enabled |
| MITRE | T1110 Brute Force (Credential Access) |

**Why it was needed:** on October 20 the built-in rules produced 53 failure alerts, but none above level 10. A single noisy attempt and a sustained attack looked similar in severity. Rule 100100 escalates when brute-force alerts repeat from the same source IP, which separates a persistent attacker from a one-off typo and lifts it to level 12 so it stands out in the dashboard and triggers email.

**Result:** it fired on October 21 at 16:56:37, 16:57:03 and 17:24:15. Each firing lines up with a Hydra run that finished a few seconds earlier.

### Key fields from the alert (rule 100100)

| Field | Value |
|---|---|
| `data.srcip` | redacted (attacker IP) |
| `data.dstuser` | victor |
| `agent.name` / `agent.ip` | victim-ubuntu / 192.168.1.102 |
| `decoder.name` | sshd |
| `location` | journald |
| `rule.mitre.id` | T1110 |

The `previous_output` field of the alert lists the underlying "Failed password for victor from <attacker IP> ... ssh2" log lines, which makes it quick to prove the source of the attack.

## 6. Impact assessment

- **Confidentiality:** no impact. No password was found and no session was opened (0 authentication successes in every window).
- **Integrity and availability:** no impact observed.
- **Exposure:** SSH on port 22 was reachable with password authentication enabled, which is what made the attack possible.

## 7. Response and recommendations

In a production environment, the response to this alert would be:

1. **Contain:** block the attacker IP at the firewall, or with an active response such as Wazuh active-response or fail2ban.
2. **Investigate:** search the victim's authentication logs for any "Accepted password" from the same IP to confirm there was no successful login.
3. **Harden SSH:** disable password authentication and use key-based login only, disable direct root login, and restrict SSH to known source addresses.
4. **Reduce guessability:** enforce strong passwords and account lockout, and limit login attempts per connection (`MaxAuthTries`).
5. **Tune detection:** keep rule 100100 and review its frequency and timeframe to balance false positives against detection speed.

## 8. Lessons learned

- Default rules detect brute force well, but severity capped at level 10, so a custom correlation rule was useful for prioritisation.
- Timestamps need care: the dashboard shows IST while the victim's raw logs are in UTC, so they differ by 5 hours 30 minutes.
- The victim's IP changed between sessions, so a static IP or DHCP reservation would keep lab evidence consistent.
- Collecting alert details (`previous_output`, `srcip`, `dstuser`) directly from the dashboard gave enough evidence to rebuild the whole attack without touching the victim.

## 9. Evidence index

All screenshots are in the [`screenshots`](../screenshots/) folder, numbered in timeline order (01 to 22). The attacker IP address and the MAC addresses shown by Nmap were redacted. The remaining IP addresses are the victim machine's private lab addresses.
