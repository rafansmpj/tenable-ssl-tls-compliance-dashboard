# Tenable SSL & TLS - Compliance Dashboard

An Cloude Desktop Skill that works with the Hexa AI MCP to build/generates an interactive Executive Dashboard for SSL & TLS Visibility. 

## What it does

Performs an SSL/TLS encryption compliance analysis of the Tenable environment via MCP and delivers an interactive visual dashboard (standalone HTML) in the conversation.

Audience: Infrastructure/Security Engineers, PKI/certificate owners, Compliance, CISO. Typical use: give the infrastructure team an actionable list before an auditor (PCI DSS, SOC 2, ISO 27001) forces the fix.

Why this matters (business context): TLS 1.0/1.1, weak ciphers (RC4, 3DES, NULL) and expired/self-signed certificates are an automatic failure in most compliance frameworks (PCI DSS 4.0 requires TLS 1.2+; auditors treat an expired certificate as a control failure). Finding these via scan and fixing them proactively is far cheaper than finding them via an audit finding.

Mode: Read-only — diagnostic only. The skill never replaces certificates, disables protocols/ciphers, restarts services, or modifies the environment. Recommendations are text, not actions.


## Quick Start (5 minutes)

### 1. Prerequisites

- Claude Code (or another Claude environment with skills support) installed locally.
- A Tenable One MCP server connected and authenticated, this is what provides live Exposure Score, vulnerability findings, Crown Jewels, and attack path chokepoint data. Without it, the skill falls back to a "DEMO — illustrative data" mock dashboard.
- An environment capable of rendering the resulting React/HTML artifact (Claude.ai or Claude Code's artifact viewer).

### 2. Installation

Copy the skill folder into the local Claude Code skills directory:

~/.claude/skills/tenable-ssl-tls-compliance-dashboard/

The folder should contain SKILL.md (and optionally README.md, LICENSE, listing.yaml as packaged). No build step or dependencies to install — it's a prompt-driven skill, not code that needs compiling.

### 3. Configure

- Make sure the Tenable One HEXA AI MCP connector is set up and active in the same Claude environment (this is the one external dependency; everything else, like the color palette, phase logic, and scoring weights, is hardcoded in SKILL.md).
- Optionally, set organizational context the skill will use when invoked: industry/sector (for benchmark comparison), language preference (English/Portuguese), and any default filters (e.g., always focus on Crown Jewels, or always include regulatory mapping like LGPD/ISO 27001/NIST CSF/PCI-DSS).
- No API keys or .env files are needed inside the skill itself — credentials live entirely in the MCP connector configuration, not in the skill folder.

 ## Content Agreement
 I have reviewed and accept the CyberAgents Exchange Contribution Agreement

