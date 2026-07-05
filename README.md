# android-reversing

> **Agentic Android traffic inspection, verification, and exfiltration detection.**
>
> Spin up emulators, intercept every byte an app sends, map its network footprint, and surface malicious data exfiltration — all driven by an autonomous agent pipeline.

---

## Table of Contents

1. [Overview](#overview)
2. [How It Works](#how-it-works)
3. [Architecture](#architecture)
4. [Prerequisites](#prerequisites)
5. [Quick Start](#quick-start)
6. [Configuration](#configuration)
7. [Running an Analysis](#running-an-analysis)
8. [What the Agent Checks](#what-the-agent-checks)
9. [Output & Reports](#output--reports)
10. [Extending the Pipeline](#extending-the-pipeline)
11. [Security & Legal Considerations](#security--legal-considerations)
12. [Contributing](#contributing)
13. [License](#license)

---

## Overview

`android-reversing` is an **agentic analysis pipeline** that automates the full lifecycle of Android app traffic inspection:

| Stage | What happens |
|-------|-------------|
| **Ingest** | Receive an APK (file path, URL, or package ID) |
| **Provision** | Spin up a clean Android emulator snapshot |
| **Instrument** | Install the app and inject a trusted CA certificate so all TLS traffic is intercepted |
| **Execute** | Drive the app through automated UI scenarios (login, onboarding, background sync, etc.) |
| **Capture** | Record every network request/response via an intercepting proxy |
| **Analyze** | Run the captured traffic through an AI agent that classifies endpoints, detects PII leakage, flags suspicious destinations, and scores exfiltration risk |
| **Report** | Produce a human-readable report and machine-readable JSON artifact |

The pipeline is fully automated — drop in an APK and get back a threat assessment with zero manual steps.

---

## How It Works

```
APK / package name
        │
        ▼
┌───────────────────┐
│  Emulator Manager │  ← provisions a fresh AVD snapshot
└────────┬──────────┘
         │ ADB
         ▼
┌───────────────────┐
│  Proxy Controller │  ← mitmproxy / Burp Suite (headless)
│  (TLS intercept)  │
└────────┬──────────┘
         │ HAR / raw streams
         ▼
┌───────────────────┐
│  App Driver       │  ← Appium / UIAutomator2 scenarios
└────────┬──────────┘
         │ captured traffic
         ▼
┌───────────────────┐
│  Analysis Agent   │  ← LLM + rule-based classifiers
│  · endpoint map   │
│  · PII detector   │
│  · TLS validator  │
│  · geo/ASN lookup │
└────────┬──────────┘
         │
         ▼
  JSON + Markdown report
```

### Agent Decision Loop

The analysis agent runs a **ReAct** (Reason + Act) loop:

1. **Observe** — load the captured HAR/PCAP
2. **Reason** — identify hosts, decode payloads, extract fields
3. **Act** — call tools (GeoIP lookup, WHOIS, VirusTotal, DNS, certificate transparency logs)
4. **Evaluate** — score each finding against an exfiltration rubric
5. **Summarise** — produce structured findings

---

## Architecture

```
android-reversing/
├── agent/                  # Core agentic pipeline
│   ├── orchestrator.py     # Entry point; drives all stages
│   ├── emulator.py         # AVD lifecycle (create / snapshot / restore / delete)
│   ├── proxy.py            # mitmproxy addon — captures & streams HAR entries
│   ├── driver.py           # Appium session management & scenario runner
│   ├── analyzer.py         # LLM + rule-based traffic analysis
│   ├── tools/              # Agent tools (GeoIP, WHOIS, VirusTotal, CT logs …)
│   └── prompts/            # System & analysis prompt templates
├── scenarios/              # YAML-defined UI automation scenarios
│   ├── generic_onboarding.yaml
│   ├── account_creation.yaml
│   └── background_sync.yaml
├── rules/                  # YARA / Sigma rules for traffic pattern matching
│   ├── pii_exfiltration.yar
│   ├── c2_beaconing.yar
│   └── ad_sdk_telemetry.yar
├── infra/                  # Docker Compose & Terraform for the intercept stack
│   ├── docker-compose.yml
│   ├── mitmproxy/
│   └── certificates/
├── reports/                # Generated reports (git-ignored)
├── config.yaml             # Master configuration
├── requirements.txt        # Python dependencies
└── README.md
```

---

## Prerequisites

### Host machine

| Requirement | Minimum version | Notes |
|-------------|----------------|-------|
| Python | 3.11+ | `python3 --version` |
| Android SDK / cmdline-tools | latest | `sdkmanager`, `avdmanager`, `emulator` must be on `$PATH` |
| ADB | 1.0.41+ | bundled with platform-tools |
| Docker & Docker Compose | 24+ | for the proxy stack |
| Node.js | 18+ | for Appium server |
| Appium | 2.x | `npm install -g appium` |
| UIAutomator2 driver | latest | `appium driver install uiautomator2` |

### API keys (optional but recommended)

| Service | Environment variable | Purpose |
|---------|---------------------|---------|
| VirusTotal | `VT_API_KEY` | Reputation lookup for IPs / domains |
| Shodan | `SHODAN_API_KEY` | Banner / service info for endpoints |
| OpenAI / Anthropic | `LLM_API_KEY` | LLM-powered analysis agent |
| ipinfo.io | `IPINFO_TOKEN` | GeoIP + ASN enrichment |

Keys are read from environment variables or a `.env` file at the repo root (never commit `.env`).

---

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/alshain/android-reversing.git
cd android-reversing

# 2. Create and activate a Python virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 3. Install Python dependencies
pip install -r requirements.txt

# 4. Install and start Appium
npm install -g appium
appium driver install uiautomator2
appium &

# 5. Start the intercept proxy stack
docker compose -f infra/docker-compose.yml up -d

# 6. Copy and edit the configuration
cp config.yaml.example config.yaml
# Set your API keys and preferred AVD image in config.yaml

# 7. Run the pipeline against an APK
python -m agent.orchestrator --apk /path/to/suspicious.apk

# 8. View the report
open reports/suspicious_<timestamp>/report.md
```

---

## Configuration

`config.yaml` controls every aspect of the pipeline:

```yaml
emulator:
  avd_name: "android-reversing-avd"
  api_level: 33                      # Android 13
  abi: "x86_64"
  device_profile: "pixel_6"
  snapshot: "clean_baseline"         # restored before each analysis run
  proxy_host: "10.0.2.2"            # host loopback as seen from emulator
  proxy_port: 8080

proxy:
  host: "0.0.0.0"
  port: 8080
  tls_intercept: true
  export_har: true
  export_pcap: true

driver:
  appium_url: "http://localhost:4723"
  scenarios:
    - scenarios/generic_onboarding.yaml
    - scenarios/background_sync.yaml
  timeout_seconds: 300               # max runtime per scenario

analysis:
  llm_provider: "openai"             # openai | anthropic | local
  model: "gpt-4o"
  pii_fields:
    - email
    - phone
    - device_id
    - android_id
    - imei
    - gps_coordinates
    - contacts
  risk_threshold: 7                  # 0-10; findings above this are HIGH severity

reporting:
  output_dir: "reports"
  formats:
    - markdown
    - json
    - html
```

---

## Running an Analysis

### From an APK file

```bash
python -m agent.orchestrator --apk suspicious_app.apk
```

### From a Play Store package ID (requires device with Play)

```bash
python -m agent.orchestrator --package com.example.suspiciousapp
```

### From a URL

```bash
python -m agent.orchestrator --url https://example.com/app.apk
```

### Options

```
--apk PATH          Path to the APK file
--package ID        Android package identifier
--url URL           Direct download URL for the APK
--scenario FILE     Override the scenario YAML to use
--no-restore        Skip snapshot restore (faster, less clean)
--keep-emulator     Leave the AVD running after analysis
--output DIR        Override the report output directory
--config FILE       Path to an alternate config.yaml
--verbose           Stream detailed logs to stdout
--dry-run           Validate config and APK without running the pipeline
```

---

## What the Agent Checks

### Network Behaviour

| Check | What it looks for |
|-------|------------------|
| **Endpoint inventory** | All unique hosts / IP addresses contacted |
| **TLS certificate validation** | Self-signed certs, certificate pinning bypass, weak cipher suites |
| **Unencrypted traffic** | Any HTTP (non-HTTPS) requests carrying sensitive data |
| **DNS behaviour** | DoH/DoT usage that may evade proxy; DNS-over-HTTPS leaks |
| **Geo & ASN analysis** | Traffic to unusual jurisdictions, bulletproof hosting ASNs |
| **VirusTotal / reputation** | Domains and IPs checked against threat intelligence feeds |

### Data Exfiltration

| Check | What it looks for |
|-------|------------------|
| **PII extraction** | Email, phone, IMEI, Android ID, advertising ID in request bodies |
| **Device fingerprinting** | Harvesting of hardware identifiers beyond what's needed |
| **Location exfiltration** | GPS coordinates, cell tower IDs sent to third-party endpoints |
| **Contact / media scraping** | Uploads to endpoints that are not the primary app backend |
| **Clipboard harvesting** | Clipboard content sent in background requests |
| **Credential exfiltration** | Passwords / tokens forwarded to non-primary endpoints |

### SDK & Third-Party Libraries

| Check | What it looks for |
|-------|------------------|
| **Ad SDK telemetry** | Known ad-network endpoints receiving excessive device data |
| **Analytics SDKs** | Fingerprinting or tracking beyond typical analytics scope |
| **C2 beaconing patterns** | Regular heartbeat-style requests with encoded payloads |
| **Dynamic code loading** | Requests to fetch and execute additional code at runtime |

### Static Cross-Check

The agent also performs a **light static pass** (using `apktool` + `jadx`) and cross-references:

- Declared permissions vs. observed network activity
- Hardcoded URLs in smali / decompiled code vs. observed traffic
- Manifest receivers / services that may trigger network calls silently

---

## Output & Reports

Every run produces a timestamped directory under `reports/`:

```
reports/suspicious_app_20250705_151045/
├── report.md          # Human-readable Markdown summary
├── report.html        # Self-contained HTML version with charts
├── findings.json      # Machine-readable structured findings
├── traffic.har        # Full HAR archive of all intercepted requests
├── traffic.pcap       # Raw PCAP (requires tcpdump on host)
├── endpoints.csv      # Deduplicated endpoint list with enrichment
└── screenshots/       # UI screenshots taken during automation
```

### Sample `findings.json` schema

```json
{
  "app": {
    "package": "com.example.suspiciousapp",
    "sha256": "abc123…",
    "analysis_timestamp": "2025-07-05T15:10:45Z"
  },
  "risk_score": 8.4,
  "risk_level": "HIGH",
  "findings": [
    {
      "id": "F-001",
      "category": "pii_exfiltration",
      "severity": "HIGH",
      "description": "IMEI and Android ID transmitted to analytics.thirdparty.io over HTTPS",
      "evidence": {
        "request_url": "https://analytics.thirdparty.io/collect",
        "request_method": "POST",
        "pii_fields_detected": ["imei", "android_id"],
        "destination_asn": "AS12345 SomeShadyHosting Ltd",
        "destination_country": "XX"
      },
      "recommendation": "Verify necessity of IMEI collection. Consider using resettable identifiers only."
    }
  ],
  "endpoints": [
    {
      "host": "analytics.thirdparty.io",
      "ip": "1.2.3.4",
      "asn": "AS12345",
      "country": "XX",
      "vt_malicious": 3,
      "request_count": 47,
      "pii_observed": true
    }
  ]
}
```

---

## Extending the Pipeline

### Adding a new analysis scenario

Create a YAML file in `scenarios/`:

```yaml
# scenarios/my_scenario.yaml
name: "Custom Login Flow"
description: "Logs into the app and navigates to the profile page"
steps:
  - action: launch_app
  - action: tap
    locator: { resource_id: "com.example.app:id/btn_login" }
  - action: type
    locator: { resource_id: "com.example.app:id/email_field" }
    text: "test@example.com"
  - action: type
    locator: { resource_id: "com.example.app:id/password_field" }
    text: "TestPassword123!"
  - action: tap
    locator: { resource_id: "com.example.app:id/btn_submit" }
  - action: wait
    seconds: 5
  - action: navigate
    locator: { content_desc: "Profile" }
```

Run with:

```bash
python -m agent.orchestrator --apk app.apk --scenario scenarios/my_scenario.yaml
```

### Adding a new agent tool

Drop a Python file into `agent/tools/` implementing the `BaseTool` interface:

```python
# agent/tools/my_tool.py
from agent.tools.base import BaseTool, ToolResult

class MyTool(BaseTool):
    name = "my_tool"
    description = "Does something useful during analysis"

    def run(self, input: str) -> ToolResult:
        # … your logic …
        return ToolResult(output="result", metadata={})
```

The orchestrator auto-discovers tools in `agent/tools/` at startup.

### Adding YARA / detection rules

Drop `.yar` files into `rules/`. They are applied to decoded request/response bodies. Rules should follow the convention:

```yara
rule SuspiciousIMEIExfiltration {
    meta:
        severity = "HIGH"
        category = "pii_exfiltration"
        description = "IMEI transmitted in plaintext POST body"
    strings:
        $imei_key = "imei=" nocase
        $imei_key2 = "\"imei\":" nocase
    condition:
        any of them
}
```

---

## Security & Legal Considerations

> ⚠️ **This tool is intended for security research, threat intelligence, and legitimate app vetting only.**

- **Only analyze apps you own, have explicit written permission to test, or are covered by a responsible disclosure program.**
- Traffic interception involves a man-in-the-middle proxy. Using this tool against apps or networks without authorisation may violate computer fraud laws (e.g., CFAA, Computer Misuse Act, GDPR).
- The synthetic test account credentials used during automation should be dedicated throwaway accounts with no real personal data.
- Emulators should be isolated from the host network (use the Docker Compose bridge network) and torn down after each run to prevent cross-contamination.
- Reports may contain sensitive data recovered from app traffic. Store and share them securely.

---

## Contributing

1. Fork the repository and create a feature branch: `git checkout -b feat/my-improvement`
2. Make your changes and add tests where applicable
3. Run the linter and test suite: `ruff check . && pytest`
4. Open a pull request describing your change and its motivation

Please follow the [Conventional Commits](https://www.conventionalcommits.org/) specification for commit messages.

---

## License

This project is licensed under the **MIT License** — see [LICENSE](LICENSE) for details.

---

*Built for defenders, researchers, and anyone who wants to know exactly what their Android apps are saying behind their back.*
