# Troubleshooting Log — PAM Deployment (Scenario 1)

Three real infrastructure issues were diagnosed and resolved while deploying JumpServer for this lab. Summarized here from the original PAM-lab build notes — replace/expand each section with the full incident write-up if you have more detail than the summary below.

---

## Incident 1 — NetworkManager silently reverting manual IP assignments

**Symptom:** Static IP addresses assigned imperatively (`ip addr add ...`) on the secOps VM did not persist — the interface would revert to its previous address shortly after.

**Root cause:** NetworkManager's own reconciliation loop was overwriting the imperative assignment, since the interface was still under NetworkManager's management and had no matching persistent profile.

**Fix:** Configured the static IP via `nmcli` instead of raw `ip addr add`, so the assignment is written into a NetworkManager-managed connection profile rather than being immediately reconciled away.

📸 Related: `screenshots/01-scenario1-pam-enforcement/01-network-foundation/01c-secops-manual-ip-config.png`, `01d-secops-ping-success.png`, `01e-rhadmin-ping-success.png`

---

## Incident 2 — Missing Python dependency (`requests_unixsocket`) in the official `jumpserver/core` image

**Symptom:** Both the `core` and `celery` containers entered a crash loop shortly after startup.

**Root cause:** The official `jumpserver/core` image was missing the `requests_unixsocket` Python dependency at the pulled version. Since `core` and `celery` share the same base image but run in separate containers with independent writable layers, the missing-dependency fix had to be applied to each container independently.

**Fix:** Installed the missing dependency inside each affected container's writable layer.

📸 Related: `screenshots/01-scenario1-pam-enforcement/02-jumpserver-deployment/02-jumpserver-containers-healthy.png` (post-fix, all 8/8 containers healthy)

---

## Incident 3 — Missing `jumpserver/ansible-executor` image

**Symptom:** JumpServer's built-in "Test Connectivity" action failed for the newly onboarded asset.

**Root cause:** JumpServer's connectivity test spins up a separate, Ansible-based executor container to actually perform the SSH probe against the target asset. This image was never pulled during the initial install and had to be fetched manually.

**Fix:** Manually pulled the `jumpserver/ansible-executor` image; connectivity test then passed cleanly.

📸 Related: `screenshots/01-scenario1-pam-enforcement/04-connectivity-verification/04-connectivity-test-passing.png`

---
