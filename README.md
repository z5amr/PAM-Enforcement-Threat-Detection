# PAM-lab — Privileged Access Management with JumpServer

A hands-on lab demonstrating Privileged Access Management (PAM) concepts using [JumpServer](https://github.com/jumpserver/jumpserver), an open-source PAM platform, deployed in a fully isolated virtual network. Built as part of my transition into Cloud / Network / Ops / Security roles.

## Why this matters

Enterprises don't let engineers SSH directly into production servers with shared credentials — that fails every compliance audit (SOC 2, PCI-DSS, ISO 27001) that requires access control, least-privilege enforcement, and session auditability. PAM platforms like JumpServer, CyberArk, and BeyondTrust solve this by sitting between the user and the target system: broker the connection, enforce who can reach what, log every keystroke, and block dangerous commands before they execute.

This lab builds that stack from scratch — network isolation, PAM deployment, access policy configuration, and live enforcement testing — to demonstrate practical understanding of *why* these controls exist, not just how to click through a UI.

## Architecture

```
                    ┌─────────────────────────┐
                    │   pam-lab (isolated     │
                    │   VMware LAN segment)   │
                    │   10.0.0.0/24           │
                    └─────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ RH-LinuxAdmin │   │ Cybersecurity_Lab│   │  (planned)       │
│ 10.0.0.17     │   │ VM ("secOps")    │   │  Windows target  │
│               │   │ 10.0.0.18        │   │  RDP + LOLBAS    │
│ JumpServer    │   │ SSH target       │   └──────────────────┘
│ (8 containers)│   │ (analyst account)│
└───────────────┘   └──────────────────┘
        │
   NAT adapter (2nd NIC)
   for image pulls only
```

- **RH-LinuxAdmin** — hosts the full JumpServer stack (core, celery, koko, lion, chen, web, postgresql, redis) via Docker Compose, dual-homed: one NIC on the isolated lab segment, one NAT NIC used only for pulling container images.
- **Cybersecurity_Lab_VM ("secOps")** — the managed target. SSH-only, no direct access — all connections proxied through JumpServer's `koko` component.
- Network is fully air-gapped from the host machine and internet except where explicitly needed (image pulls), mirroring how a real segmented environment isolates privileged access paths.

## What's been built

### 1. Network foundation
- Isolated VMware LAN segment (`pam-lab`, `10.0.0.0/24`) connecting the JumpServer host and target
- Static IP configuration via `nmcli` (NetworkManager-managed interfaces)
- Dual-NIC setup on the JumpServer host: isolated segment + NAT for image pulls only

### 2. JumpServer deployment
- Full 8-container stack deployed via the official quick-start installer (Docker Compose)
- All services verified healthy: `core`, `celery`, `koko`, `lion`, `chen`, `web`, `postgresql`, `redis`

### 3. Asset onboarding & access control
- Target asset (`analyst@10.0.0.18`) registered with SSH protocol
- Account credential stored and connectivity verified (green "OK" status)
- **Authorization** created binding: Administrator user → `analyst` asset → `analyst` account → Connect action
- Connectivity confirmed end-to-end through JumpServer's web-based Luna terminal (`koko` proxy)

### 4. Session recording & audit
- Live SSH session proxied through JumpServer, confirmed via `whoami` / `hostname` inside the session
- Session recording captured and replayable (`Audits → Asset sessions → Historical sessions`)
- Every command logged individually via `Audits → Session commands`

### 5. Command filtering (active enforcement)
- Custom **Command Group** (`Dangerous Commands`) defined: `rm`, `rm -rf`, `sudo su`, `sudo -i`, `dd if=`, `mkfs`
- **Command Filter ACL** created referencing that group, action set to **Reject**
- Verified the filter actively blocks matched commands mid-session — not just logging them after the fact

### 6. Access revocation
- Demonstrated that disabling an authorization immediately removes the asset from the user's accessible list in Workbench and blocks further connection attempts — proving enforcement, not just configuration

## Screenshots

All screenshots live in [`/screenshots`](./screenshots), numbered to follow the build order below.

### 1. Network foundation
| File | Description |
|---|---|
| `01a-vmware-lan-segment-rhadmin.png` | RH-LinuxAdmin network adapter set to the isolated `pam-lab` LAN segment |
| `01b-vmware-lan-segment-cyberlab.png` | Cybersecurity_Lab_VM (secOps) network adapter set to the same `pam-lab` segment |
| `01c-secops-manual-ip-config.png` | secOps — manual IP assignment before the NetworkManager fix (see Incident 1) |
| `01d-secops-ping-success.png` | secOps → RH-LinuxAdmin ping succeeding after the fix |
| `01e-rhadmin-ping-success.png` | RH-LinuxAdmin → secOps ping succeeding, confirming bidirectional connectivity |

### 2. JumpServer deployment
| File | Description |
|---|---|
| `02-jumpserver-containers-healthy.png` | Full 17/17 image pulls and 8/8 containers started |

### 3. Asset, account & authorization setup
| File | Description |
|---|---|
| `03a-jumpserver-signin-page.png` | JumpServer sign-in page, accessed from the secOps desktop |
| `03b-first-login-console-pam-audits-workbench.png` | First-login onboarding and the Console / PAM / Audits / Workbench module switcher |
| `03c-console-dashboard.png` | Console module dashboard and sidebar navigation |
| `03d-asset-protocol-config.png` | Asset edit panel — SFTP/SSH protocol configuration |
| `03e-authorization-details-actions.png` | Authorization detail — granted actions (Connect, Upload, Download, etc.) |
| `03f-users-list.png` | Users list — single Administrator account |
| `03g-user-authorization-rules-tab.png` | Administrator's Authorization rules tab, linked to `secOps-target` |
| `03h-asset-details-basic.png` | Asset details — basic information panel |
| `03i-asset-details-hardware.png` | Asset details — auto-discovered hardware info (CPU, OS, MAC) |
| `03j-authorization-full-config.png` | **Key fix** — full authorization config with User + Asset + Account all correctly bound (see Incident 4) |

### 4. Connectivity verification
| File | Description |
|---|---|
| `04-connectivity-test-passing.png` | Connectivity test output: "Connection and authentication successful" (after fixing the missing Ansible executor image — Incident 3) |

### 5. Live proxied session
| File | Description |
|---|---|
| `05a-workbench-access-assets-list.png` | Workbench showing the `analyst` asset available after the authorization fix |
| `05b-connect-dialog-analyst.png` | Connect dialog — account and connect-method selection |
| `05c-session-terminal-whoami-hostname.png` | Live proxied SSH session — `whoami`/`hostname` confirming a real connection to secOps |

### 6. Session recording & audit
| File | Description |
|---|---|
| `06a-session-playback-player.png` | Session replay player — command sidebar, scrubber, live terminal replay |
| `06b-session-playback-download.png` | Recording export as a downloadable `.tar` file |
| `06c-session-commands-log.png` | Per-command audit log for a session (7 commands, all "Accept" — pre-filter baseline) |
| `06d-pam-accounts-dashboard.png` | PAM module accounts overview |
| `06e-audits-dashboard-command-stats.png` | Audits dashboard — login logs, command stats, session trends |
| `06f-audits-historical-sessions-with-actions.png` | Audits → Historical sessions with play/download action icons |

### 7. Command filtering setup
| File | Description |
|---|---|
| `07a-command-group-dangerous-commands.png` | Command Group definition — `rm`, `rm -rf`, `sudo su`, `sudo -i`, `dd if=`, `mkfs` |
| `07b-command-filter-acl-config.png` | Command Filter ACL — group bound, Action: **Reject** |

### 8. Active enforcement
| File | Description |
|---|---|
| `08a-commands-blocked-live.png` | **Highlight** — all six dangerous commands rejected live, each with a `Command 'X' is forbidden` message |
| `08b-blocked-commands-alt-view.png` | Second view of the same enforcement test |
| `08c-asset-sessions-replayable-playback-columns.png` | Asset sessions table showing Replayable and Playback status columns |

## Known issues encountered & fixed

Three real infrastructure bugs were diagnosed and resolved during this build — see [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md) for full incident-style writeups:

1. **NetworkManager silently reverting manual IP assignments** — imperative `ip addr add` commands were overwritten by NetworkManager's own reconciliation loop; fixed by configuring via `nmcli` instead.
2. **Missing Python dependency (`requests_unixsocket`) in the official `jumpserver/core` image** — caused a crash loop on both `core` and `celery` containers (same image, separate writable layers, required patching each independently).
3. **Missing `jumpserver/ansible-executor` image** — JumpServer's connectivity test spins up a separate Ansible-based container to perform the actual SSH probe; this image was never pulled during install and had to be fetched manually.

## Roadmap / not yet implemented

- [ ] Tiered access: second low-privilege account with a separate, narrower authorization
- [ ] Time-boxed / expiring access grants (just-in-time access pattern)
- [ ] MFA enabled on the admin account
- [ ] Windows target VM — RDP proxying via `lion`, LOLBAS detection pairing
- [ ] Security Onion integration — network-level visibility into JumpServer-proxied sessions

## Tech stack

`JumpServer v4.10.19-ce` · `Docker` · `VMware Workstation` · `RHEL 10` · `Ubuntu 22.04` · `NetworkManager` · `Ansible` (via JumpServer's executor)

## Author's note

Built as a self-directed lab while studying for Cloud/Network/Ops/Security roles. The goal wasn't just to get JumpServer running, but to understand *why* each piece exists — the network isolation, the credential vaulting, the session recording, the command filtering — and to practice the kind of methodical, layer-by-layer debugging (network → routing → application → dependency) that real infrastructure work actually requires.
