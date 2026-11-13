# Detection Lab

A home SOC (security operations center) built on Proxmox. I attack my own intentionally vulnerable machines from Kali Linux, collect their logs in a Wazuh SIEM, and write detection rules mapped to MITRE ATT&CK.

The goal is to measure what I can actually detect: for each attack, was it detected, how long did it take, and how many false positives did it create.

## Architecture

Two virtual networks run on one Proxmox host:

- **LAN** (`vmbr0`, 192.168.1.0/24): normal network with internet access.
- **Lab** (`vmbr1`, 10.10.10.0/24): isolated network with no DHCP and no internet. The vulnerable machines live here.

A network diagram will be added in `docs/`.

## Components

| Host | Network | IP | Role |
|---|---|---|---|
| Proxmox VE | LAN | 192.168.1.50 | Hypervisor |
| Kali Linux (VM) | LAN + Lab | 10.10.10.10 (lab) | Attacker |
| Metasploitable (VM) | Lab | 10.10.10.20 | Vulnerable target |
| DVWA (LXC) | Lab | 10.10.10.30 | Vulnerable web app (Wazuh agent) |
| Wazuh (VM) | LAN + Lab | 10.10.10.5 (lab) | SIEM (manager, indexer, dashboard) |

## Detections

| Rule ID | Level | What it detects | MITRE ATT&CK |
|---|---|---|---|
| 100100 | 5 | One failed SSH password attempt | T1110.001 Password Guessing |
| 100101 | 10 | 6 failed SSH passwords in 60 seconds from the same IP | T1110 Brute Force |

Files:

- Decoder: `detections/wazuh/decoders/msf-syslog-sshd.xml`
- Rules: `detections/wazuh/rules/ssh-bruteforce.xml`

## Coverage

See [coverage/coverage.md](coverage/coverage.md).

## Status

- [x] Isolated lab network
- [x] Wazuh SIEM with an agent on DVWA and syslog forwarding from Metasploitable
- [x] Custom decoder and rules for SSH brute force (tested with synthetic logs)
- [ ] Real SSH brute force from Kali and time-to-detect measurement
- [ ] Sigma versions of the rules
- [ ] Web attacks against DVWA
- [ ] Pivoting through a second network

## Troubleshooting

### Metasploitable syslog events did not create alerts

**Symptom:** Wazuh received the logs (they appeared in `archives.log`), but the dashboard showed no alerts.

**Diagnosis:** A packet capture with `tcpdump` showed that the old `sysklogd` on Metasploitable forwards messages without a timestamp or hostname (for example, `sshd: Failed password ...`). Because of that, Wazuh could not extract the program name, so no decoder matched. `wazuh-logtest` confirmed it: the event only matched the generic rule 1002 (level 2), which is below the alert threshold.

**Fix:** A custom decoder that recognizes lines starting with `sshd: `, plus custom rules in the 100100+ range.