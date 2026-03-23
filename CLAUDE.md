# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DShield Sensor is a medium-interaction honeypot system that collects and reports Internet threat data to the [DShield community](https://dshield.org). It runs on dedicated hardware (Raspberry Pi 3+, n100 mini PC) or VMs (AWS/Azure) running Raspberry Pi OS 64-bit or Ubuntu 24.04 LTS.

## Key Commands

**Installation:**
```bash
sudo bin/install.sh           # Fresh install
sudo bin/install.sh --update  # Update preserving config
sudo bin/uninstall.sh         # Remove installation
```

**Web Honeypot (isc_agent):**
```bash
cd srv/web
python3 ./isc_agent -h                    # Help
python3 ./isc_agent -c /etc/dshield.ini  # Run from source
./build_zipapp.sh                         # Build distributable zipapp
python ./isc_agent.pyz -c dshield.ini   # Run as zipapp
```

**Testing web honeypot rules:**
```bash
cd srv/web
python3 rule_checker.py   # Validate rule definitions
python3 test_web.py       # Test web responses
```

**Terraform deployment:**
```bash
cd terraform/aws   # or terraform/azure
terraform init && terraform apply
```

## Architecture

Traffic flows: Internet → Firewall (iptables/nftables) → Honeypots → DShield API

**Firewall layer** (`etc/network/`): Logs all new connections to `/var/log/dshield.log`; redirects ports 22→2222, 23→2223, 80/8080→8000, 443→8443.

**SSH/Telnet honeypot**: Cowrie (`srv/cowrie/cowrie.cfg`) runs on ports 2222/2223.

**HTTP/HTTPS honeypot** (`srv/web/isc_agent/`):
- `__main__.py` — HTTP server entry point, handles request routing
- `core.py` — Request scoring and signature matching against DShield rules
- `agent.py` — Background thread that batches and submits logs to DShield API
- `stunnel_manager.py` — Manages HTTPS proxy (stunnel4 forwarding 8443→8000)
- `ncat_manager.py` — Network handler

**Log submission** (`srv/dshield/`):
- `DShield.py` — API client class (authentication, formatting, POST to dshield.org)
- `fwlogparser.py` — Parses firewall logs and submits connection data
- `access_log_parser.py` — Parses HTTP access logs

**Cron jobs** (`etc/cron.hourly/`): Run twice per hour to submit firewall and web logs.

**Systemd services** (`lib/systemd/system/`): `cowrie.service`, `isc-agent.service`, `dshieldfirewall.service`, `dshieldnft.service`.

## Configuration

Main config: `/etc/dshield.ini` (created by `install.sh` from user's dshield.org account credentials). Contains user ID, API key, honeypot IP, port assignments, and anonymization settings.

The `install.sh` script (2,600+ lines) handles all system setup: installs dependencies, configures firewall, deploys services, and writes config files from templates in `etc/`.

## Rule System

The web honeypot uses a signature-based rule system fetched from DShield API. See `srv/web/RULES.md` and `srv/web/CUSTOMIZATIONS.md` for rule format documentation and `srv/web/RULES_TESTING.md` for testing guidance.

## Supported Platforms

- Raspberry Pi OS 64-bit (recommended)
- Ubuntu 24.04 LTS Server
- openSUSE Leap/Tumbleweed
- AWS EC2 / Azure VMs (via Terraform in `terraform/`)

Single network interface (eth0) required. Admin SSH access moves to port 12222 post-install.

## CI/CD

GitHub Actions workflows in `.github/workflows/`:
- `isc-agent-action.yml` — Code scan on PRs to `ISC-1-isc-agent-beta` branch
- `snyk.yml` — Security vulnerability scanning
- `poetry-to-legacy.yml` — Python dependency management
