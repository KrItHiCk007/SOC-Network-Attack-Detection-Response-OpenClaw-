# 🛡️ OpenClaw SOC — Network Attack Detection & Response

A compact, modular Security Operations Center (SOC) reference stack that detects network threats in real time (Suricata), centralizes and correlates alerts (Wazuh), enriches and classifies them with an AI analyst (OpenClaw), then notifies analysts via Telegram and applies host-level blocks (UFW) when required.

This README follows the workflow shown in the project flowchart (Suricata → Wazuh → OpenClaw → Telegram → Analyst → UFW). It intentionally excludes persistent database and visualization components from the core workflow.

---

## Table of Contents

- [Project Summary](#project-summary)
- [Design & Architecture](#design--architecture)
- [Detection → Response Workflow](#detection--response-workflow)
- [Images & Mapping](#images--mapping)
- [Quick Install: Suricata, Wazuh (manager/agent) & UFW](#quick-install-suricata-wazuh-manageragent--ufw)
- [Wazuh localfile example (suricata log collection)](#wazuh-localfile-example-suricata-log-collection)
- [Telegram alert example](#telegram-alert-example)
- [Verification & Troubleshooting](#verification--troubleshooting)
- [Security Notes & Best Practices](#security-notes--best-practices)
- [Repository Layout (what's included)](#repository-layout-whats-included)

---

## Project Summary

OpenClaw SOC wires together best-of-breed open-source components into a pragmatic pipeline for detecting network threats, enriching alerts with AI, notifying analysts, and applying immediate host-level mitigations.

Short tagline: Detect fast. Analyze smart. Respond decisively.

Key components used in this project (core workflow):
- Suricata — network IDS (signature matching, eve.json / fast.log output)
- Wazuh — log collection, normalization and API access
- OpenClaw — AI analyst that reads Wazuh events and scores/enriches alerts
- Telegram — analyst notification channel (interactive alerts)
- UFW — host-level firewall used to apply blocks

This README and the repository files reflect the flow captured in the provided flowchart and screenshots.

---

## Design & Architecture

High-level responsibilities:

- Suricata (IDS)
  - Packet capture and signature matching using ET Open and custom rules.
  - Produces JSON and syslog outputs (e.g., /var/log/suricata/eve.json and /var/log/suricata/fast.log).

- Wazuh (SIEM/log collector)
  - Tails Suricata and host logs, normalizes events and exposes them through an API for programmatic access.

- OpenClaw (AI analyst)
  - Polls the Wazuh API, enriches events (geo-IP, ASN, WHOIS), classifies attack types and assigns severity/confidence.

- Telegram notifications
  - Delivers actionable alerts to analysts with suggested mitigations and quick actions (BLOCK / MONITOR).

- UFW (response)
  - Applies host firewall rules to deny traffic from confirmed malicious IPs on analyst decision or policy.

The canonical flow is: Attacker → Suricata → Wazuh → OpenClaw → Telegram → Analyst → UFW (see images/08-architecture-flowchart.png).

---

## Detection → Response Workflow

1. Suricata inspects incoming traffic and generates alerts when signatures match.  
2. Wazuh picks up Suricata outputs and system logs, normalizes events and makes them available via its API.  
3. OpenClaw pulls events from the Wazuh API, enriches and scores them; if thresholds are triggered it prepares a notification.  
4. Telegram alert is sent to the analyst with context and suggested actions.  
5. Analyst chooses BLOCK or MONITOR (interactive). If BLOCK is chosen, the system runs a UFW command to deny the attacker IP.  
6. Local logs record the block action for audit and later review.

Operational timing goals (targets):
- Detection: <100 ms
- Ingestion & normalization: <1 s
- AI classification/enrichment: ~2–3 s
- Human response: 30–60 s
- Automated block (no human): <5 s

---

## Images & Mapping

Place the provided images in an images/ folder (already present in the repo). Filenames used in this README:
- images/01-host-neofetch.png — Host environment (neofetch snapshot)
- images/02-services-status.png — systemctl outputs (ssh, vsftpd)
- images/03-suricata-wazuh-status.png — suricata & wazuh-manager status
- images/04-suricata-rules.png — /etc/suricata/rules listing
- images/05-wazuh-localfile-config.png — ossec localfile snippet showing suricata and other logs
- images/06-wazuh-indexer-hits.png — Wazuh indexer view (hits / rule descriptions)
- images/07-openclaw-install.png — OpenClaw installer output
- images/08-architecture-flowchart.png — flowchart (canonical architecture)
- images/09-telegram-alert.png — Telegram real incident alert (FTP brute-force example)

Use these images to visually document the environment and the pipeline. The README references them in the Design and Workflow sections and they are intentionally part of the repository's images/ folder.

---

## Quick Install: Suricata, Wazuh (manager/agent) & UFW

These condensed steps are intended for Ubuntu 22.04 / 24.04 and for testing or small-scale deployments. Adjust for production (separate manager/indexer, TLS, hardened configs).

1) Update system
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim build-essential ca-certificates

Install & enable UFW (host firewall)
bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
# Allow admin services (restrict to admin IPs where possible)
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status numbered
Install Suricata (stable PPA)
bash
sudo add-apt-repository -y ppa:oisf/suricata-stable
sudo apt update
sudo apt install -y suricata suricata-update

# Enable and verify
sudo systemctl enable --now suricata
sudo suricata -V
sudo systemctl status suricata
Enable Emerging Threats (ET) rules and update rules
bash
sudo suricata-update enable-source et/open
sudo suricata-update

# Test configuration
sudo suricata -T -c /etc/suricata/suricata.yaml
sudo systemctl restart suricata

# Optional: count loaded rules
sudo grep "^alert" /etc/suricata/rules/*.rules | wc -l
Install Wazuh (agent and manager example)
Add the Wazuh apt repository and GPG key:

bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
Install the Wazuh agent (on monitored hosts):

bash
sudo apt install -y wazuh-agent
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent
Install the Wazuh manager on the central host (minimal example):

bash
sudo apt install -y wazuh-manager
# Optionally install Wazuh API if needed
sudo apt install -y wazuh-api
sudo systemctl enable --now wazuh-manager
sudo systemctl status wazuh-manager
Configure Wazuh to collect Suricata logs (see next section for example localfile entries) and restart the Wazuh service.

Install OpenClaw (example shown in environment screenshots)

bash
curl -fsSL https://openclaw.ai/install.sh | bash
# Follow the installer's prompts and verify Node.js/npm versions
Wazuh localfile example (suricata log collection)
Add (or confirm) these localfile entries in /var/ossec/etc/ossec.conf so Wazuh tails Suricata and host logs:

XML
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/suricata/fast.log</location>
  <label key="source">suricata-alerts</label>
</localfile>

<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
  <label key="source">suricata</label>
</localfile>

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
  <label key="source">ssh-auth</label>
</localfile>

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/vsftpd.log</location>
  <label key="source">ftp</label>
</localfile>
After editing, restart Wazuh:

bash
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-agent
Telegram alert example
OpenClaw should send concise, actionable alerts via Telegram (image: images/09-telegram-alert.png). Example message body:

Code
🚨 SECURITY INCIDENT: FTP Brute-Force
Attacker IP: 100.117.121.92
Service: vsftpd
Start Time: 08:35:24 UTC
Attempts: 200+ failed logins in <3 minutes
Status: Ongoing

Suggested action:
sudo ufw insert 1 deny from 100.117.121.92
Include an inline keyboard with buttons for QUICK ACTIONS (BLOCK / MONITOR) so analysts can respond from their device.

Verification & Troubleshooting
Suricata

bash
sudo suricata -V
sudo systemctl status suricata
sudo tail -f /var/log/suricata/eve.json | jq '.'
Wazuh

bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log
OpenClaw (if installed as systemd service)

bash
sudo systemctl status openclaw
journalctl -u openclaw -f
UFW

bash
sudo ufw status numbered
sudo ufw status
If Wazuh is not seeing Suricata alerts: ensure Suricata writes to the configured paths and that ossec.conf contains the localfile entries shown above. Check permissions and AppArmor/SELinux policies if applicable.

Security Notes & Best Practices
Avoid automatic wide-scale blocking without whitelists and safeguards.
Apply temporary blocks and automatic expiry when possible.
Protect API keys and tokens; use secure secrets management.
Harden the Wazuh API with TLS and authentication.
Keep Suricata rules and system packages updated.
Test rules and automation in staging before production.
Repository Layout (what's included)
This push updates or overwrites only README.md on branch main. The repository is expected to already contain the images/ folder with the following filenames:

images/01-host-neofetch.png
images/02-services-status.png
images/03-suricata-wazuh-status.png
images/04-suricata-rules.png
images/05-wazuh-localfile-config.png
images/06-wazuh-indexer-hits.png
images/07-openclaw-install.png
images/08-architecture-flowchart.png
images/09-telegram-alert.png
No other files (scripts, .env, LICENSE, sql) will be added or modified.

Author: KrItHiCk007
Date: 2026-03-19
