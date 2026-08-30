# PAM Enforcement & Threat Detection

**Author:** Amr Ahmed Mohamed Abdeldayem
**Program:** CyberOps Associate Internship — Cisco NetAcad


---

## Overview

This repository documents a two-part SOC analyst lab built across a single monitored network segment:

| # | Scenario | What it proves |
|---|----------|-----------------|
| 1 | **PAM Enforcement Validation** | A [JumpServer](https://github.com/jumpserver/jumpserver) PAM bastion — deployed as an 8-container stack from scratch — actively blocks a defined set of dangerous commands in real time, with every stage from network isolation through live enforcement documented. Network monitoring is then used to characterize what it can and cannot see about that session. |
| 2 | **Offensive Simulation & Detection** | A real, minimal kill chain (recon → exploitation) against a deliberately vulnerable host is correctly detected, and the resulting alerts are triaged from raw payload, not signature name. |

Every packet on the segment — from both scenarios — was captured passively by **Security Onion** (Zeek + Snort) and triaged with **Squert** / **Kibana**. Every action inside the PAM bastion itself — deployment, onboarding, authorization, live enforcement — was captured at the application layer by **JumpServer's own audit system**. Together, the two evidence sets demonstrate the full picture a SOC analyst actually works with: host/application-layer audit trails *and* network-layer detection, and where each one alone falls short.

Full write-up with evidence: **[INCIDENT-REPORT.md](INCIDENT-REPORT.md)**
Infrastructure debugging log for the PAM deployment: **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)**

---

## Why This Matters

Enterprises don't let engineers SSH directly into production servers with shared credentials — that fails every compliance audit (SOC 2, PCI-DSS, ISO 27001) that requires access control, least-privilege enforcement, and session auditability. PAM platforms like JumpServer, CyberArk, and BeyondTrust solve this by sitting between the user and the target system: broker the connection, enforce who can reach what, log every keystroke, and block dangerous commands before they execute.

This lab builds that stack from scratch — network isolation, PAM deployment, access policy configuration, and live enforcement testing — then adds a second, adversarial scenario and a network detection layer on top, to demonstrate practical understanding of *why* these controls exist and how a SOC analyst validates and triages them, not just how to click through a UI.

---

## Lab Architecture

```
                         PAM-LAB SEGMENT — 10.0.0.0/24 (isolated VMware LAN)
   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌─────────────── ┐
   │ RH-LinuxAdmin │  │ secOps        │  │ Kali Linux    │  │ Metasploitable2│
   │ 10.0.0.17     │  │ 10.0.0.18     │  │ 10.0.0.20     │  │ 10.0.0.25      │
   │ JumpServer    │  │ Managed PAM   │  │ Attacker      │  │ Victim host    │
   │ (8 containers)│  │ client (SSH)  │  │               │  │                │
   └───────┬───────┘  └───────┬───────┘  └───────┬───────┘  └───────┬────────┘
           └──────────────────┴──────────────────┴──────────────────┘
                                      │
                            (passive tap — eth0)
                                      │
                          ┌───────────────────────┐
                          │     Security Onion    │
                          │  Zeek · Snort · Squert│
                          │    · Kibana · Sguil   │
                          └───────────────────────┘
```

- **RH-LinuxAdmin (10.0.0.17)** — hosts the full JumpServer stack (`core`, `celery`, `koko`, `lion`, `chen`, `web`, `postgresql`, `redis`) via Docker Compose, dual-homed: one NIC on the isolated lab segment, one NAT NIC used only for pulling container images.
- **secOps / Cybersecurity_Lab_VM (10.0.0.18)** — the managed PAM target. SSH-only, no direct access — all connections proxied through JumpServer's `koko` component.
- **Kali Linux (10.0.0.20)** / **Metasploitable2 (10.0.0.25)** — the offensive-simulation pair for Scenario 2.
- The network is fully air-gapped from the host machine and internet except where explicitly needed (image pulls), mirroring how a real segmented environment isolates privileged access paths.
- Security Onion's `eth0` passively taps the shared segment, so traffic from **both** scenarios lands in the same evidence set.

📸 [`screenshots/00-environment/00-vm-inventory.png`](screenshots/00-environment/00-vm-inventory.png) - VMware Workstation library confirming the VM inventory.

---

## Repository Structure

```
SOC-analysis-lab/
├── README.md                   
├── INCIDENT-REPORT.md            
├── TROUBLESHOOTING.md            
├── LICENSE                     
├── .gitignore
└── screenshots/
    ├── 00-environment/
    │   └── 00-vm-inventory.png                                      
    │
    ├── 01-scenario1-pam-enforcement/
    │   ├── 01-network-foundation/
    │   │   ├── 01a-vmware-lan-segment-rhadmin.png                         
    │   │   ├── 01b-vmware-lan-segment-cyberlab.png                    
    │   │   ├── 01c-secops-manual-ip-config.png                         
    │   │   ├── 01d-secops-ping-success.png                              
    │   │   └── 01e-rhadmin-ping-success.png                            
    │   ├── 02-jumpserver-deployment/
    │   │   └── 02-jumpserver-containers-healthy.png                      
    │   ├── 03-asset-account-authorization/
    │   │   ├── 03a-jumpserver-signin-page.png                            
    │   │   ├── 03b-first-login-console-pam-audits-workbench.png            
    │   │   ├── 03c-console-dashboard.png                                   
    │   │   ├── 03d-asset-protocol-config.png                         
    │   │   ├── 03e-authorization-details-actions.png                  
    │   │   ├── 03f-users-list.png                                       
    │   │   ├── 03g-user-authorization-rules-tab.png                      
    │   │   ├── 03h-asset-details-basic.png                                 
    │   │   ├── 03i-asset-details-hardware.png                              
    │   │   └── 03j-authorization-full-config.png                        
    │   ├── 04-connectivity-verification/
    │   │   └── 04-connectivity-test-passing.png                            
    │   ├── 05-live-proxied-session/
    │   │   ├── 05a-workbench-access-assets-list.png                       
    │   │   ├── 05b-connect-dialog-analyst.png                            
    │   │   └── 05c-session-terminal-whoami-hostname.png                    
    │   ├── 06-session-recording-audit/
    │   │   ├── 06a-session-playback-player.png                             
    │   │   ├── 06b-session-playback-download.png                           
    │   │   ├── 06c-session-commands-log.png                                
    │   │   ├── 06d-pam-accounts-dashboard.png                             
    │   │   ├── 06e-audits-dashboard-command-stats.png                      
    │   │   └── 06f-audits-historical-sessions-with-actions.png             
    │   ├── 07-command-filtering-setup/
    │   │   ├── 07a-command-group-dangerous-commands.png                   
    │   │   └── 07b-command-filter-acl-config.png                         
    │   ├── 08-active-enforcement/
    │   │   ├── 08a-commands-blocked-live.png                              
    │   │   ├── 08b-blocked-commands-alt-view.png                           
    │   │   └── 08c-asset-sessions-replayable-playback-columns.png          
    │   └── 09-network-detection-evidence/
    │       ├── 09a-capture-import-pam-lab-incident.png                     
    │       ├── 09b-squert-false-positive-alert-list.png                   
    │       └── 09c-squert-false-positive-raw-payload.png                 
    │
    └── 02-scenario2-offensive-simulation/
        ├── 01-nmap-scan-vsftpd-identified.png                         
        ├── 02-capture-import-metasploitable-incident.png                 
        ├── 03-kibana-overview-dashboard.png                             
        ├── 04-kibana-alert-summary-table.png                           
        ├── 05-ftp-backdoor-trigger-terminal.png                          
        ├── 06-root-shell-confirmed-nc-6200.png                           
        └── 07-squert-true-positive-root-compromise.png                    
```

> Folders `01` through `08` under Scenario 1 mirror the original PAM-lab build order exactly (network → deployment → onboarding → connectivity → live session → audit → filter policy → enforcement). Folder `09` holds the network-detection layer (Security Onion / Squert) added on top of that original lab for this SOC project.

---

## Toolchain

`JumpServer v4.10.19-ce` · `Docker` · `VMware Workstation` · `RHEL 10` · `Ubuntu 22.04` · `NetworkManager` · `Ansible` (JumpServer's connectivity-test executor) · `Security Onion` · `Zeek` · `Snort` · `Squert` · `Kibana` · `Kali Linux` · `Metasploitable2`

---

## Key Findings (summary)

| Evidence | Verdict | Why |
|----------|---------|-----|
| 6/6 filtered commands (`rm`, `rm -rf`, `sudo su`, `sudo -i`, `dd if=`, `mkfs`) | 🟢 **Enforced** | JumpServer's Command Filter ACL rejected all six live, each with `Command 'X' is forbidden`. |
| `GPL WEB_SERVER DELETE attempt` (10.0.0.18 → 10.0.0.17) | 🟡 **False Positive** | Generic Snort signature matched a legitimate JumpServer REST API call — valid session cookie + CSRF token present in payload. |
| `GPL ATTACK_RESPONSE id check returned root` (10.0.0.25 → 10.0.0.20) | 🔴 **True Positive** | Literal command output `uid=0(root) gid=0(root).` captured in response traffic from the vsftpd 2.3.4 backdoor (port 6200) — confirmed root-level compromise. |

Full evidence, payloads, and reasoning: **[INCIDENT-REPORT.md](INCIDENT-REPORT.md)**

---

## Roadmap / Not Yet Implemented

- [ ] Tiered access: second low-privilege account with a separate, narrower authorization
- [ ] Time-boxed / expiring access grants (just-in-time access pattern)
- [ ] MFA enabled on the admin account
- [ ] Windows target VM — RDP proxying via `lion`, LOLBAS detection pairing
- [x] Security Onion integration — network-level visibility into JumpServer-proxied sessions *(this repo)*

---

## Author's Note

Built as a self-directed project while completing a CyberOps Associate internship and studying for Cloud/Network/Ops/Security roles. The goal wasn't just to get JumpServer running or to pop a known-vulnerable box, it was to understand *why* each control exists (network isolation, credential vaulting, session recording, command filtering) and to practice the kind of methodical, layer-by-layer work, both building controls and triaging alerts against them, that real SOC and infrastructure work actually requires.

---

## Ethical Use Notice

This lab was performed entirely inside an isolated, non-production VM segment (`10.0.0.0/24`) against intentionally vulnerable software (Metasploitable2) for educational purposes. No systems outside this lab were accessed or tested.
