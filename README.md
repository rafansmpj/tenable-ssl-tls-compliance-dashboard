# Tenable SSL & TLS - Compliance Dashboard

A Cloude Desktop Skill that works with the Hexa AI MCP to build/generates an interactive Executive Dashboard for SSL & TLS Visibility. 

## What it does

Performs an SSL/TLS encryption compliance analysis of the Tenable environment via MCP and delivers an interactive visual dashboard (standalone HTML) in the conversation.

Audience: Infrastructure/Security Engineers, PKI/certificate owners, Compliance, CISO. Typical use: give the infrastructure team an actionable list before an auditor (PCI DSS, SOC 2, ISO 27001) forces the fix.

Why this matters (business context): TLS 1.0/1.1, weak ciphers (RC4, 3DES, NULL) and expired/self-signed certificates are an automatic failure in most compliance frameworks (PCI DSS 4.0 requires TLS 1.2+; auditors treat an expired certificate as a control failure). Finding these via scan and fixing them proactively is far cheaper than finding them via an audit finding.

Mode: Read-only — diagnostic only. The skill never replaces certificates, disables protocols/ciphers, restarts services, or modifies the environment. Recommendations are text, not actions.


## Quick Start (5 minutes)

### 1. Prerequisites

- Claude (or another Claude environment with skills support) installed locally.
- A Tenable One MCP server connected and authenticated. Without it, the skill falls back to a "DEMO — illustrative data" mock dashboard.

### 2. Installation

**Claude Desktop**

Settings → Skills → Add Skill → Search/Browse → Click Install

Claude Desktop handles the file placement automatically. Restart Claude Desktop after installing so it picks up the skill.

**Claude Code**

Clone this repo into your skills directory:

```
git clone https://github.com/rafansmpj/tenable-ssl-tls-compliance-dashboard.git ~/.claude/skills/tenable-ssl-tls-compliance-dashboard
```

Or copy the repo folder into `~/.claude/skills/` manually. Claude Code auto-discovers skills in that directory — no restart required, just start a new session.

### 3. Configure

- Make sure the Tenable One HEXA AI MCP connector is set up and active in the same Claude environment (this is the one external dependency; everything else, like the color palette, phase logic, and scoring weights, is hardcoded in SKILL.md).
- Optionally, set organizational context the skill will use when invoked: industry/sector (for benchmark comparison), language preference (English/Portuguese), and any default filters (e.g., always focus on Crown Jewels, or always include regulatory mapping like LGPD/ISO 27001/NIST CSF/PCI-DSS).
- No API keys or .env files are needed inside the skill itself — credentials live entirely in the MCP connector configuration, not in the skill folder.


### 4. Usage

After installation, simply prompt Claude:

> "Generate an SSL/TLS compliance dashboard for my Tenable environment."

The skill will ask for your preferred expiry threshold, scope, and language, then collect data from Tenable via the Hexa AI MCP and deliver a standalone HTML dashboard in the conversation.

 ## Content Agreement
 I have reviewed and accept the CyberAgents Exchange Contribution Agreement

