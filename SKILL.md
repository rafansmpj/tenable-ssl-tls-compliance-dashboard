---
name: tenable-ssl-tls-compliance-dashboard
description: >
  Generates an interactive standalone-HTML SSL/TLS compliance dashboard from Tenable data, using Plugin 56984
  for protocols/ciphers plus certificate plugins 10863, 15901, 42981, 57582, 35291, 86067 and 45411. Identifies
  deprecated protocols (SSLv2, SSLv3, TLS 1.0, TLS 1.1), weak ciphers (RC4, DES/3DES, NULL, export, anonymous),
  missing Perfect Forward Secrecy, and certificates that are expired, self-signed, expiring soon, or signed with
  a weak algorithm/key (SHA-1, MD5, RSA under 2048 bits). ALWAYS use when the user asks for: SSL/TLS compliance,
  encryption compliance, deprecated protocols, TLS 1.0, TLS 1.1, SSLv2, SSLv3, weak ciphers, cipher suite audit,
  expired certificate, self-signed certificate, certificate expiring soon, SSL certificate audit, certificate
  hygiene, encryption hygiene, PCI DSS TLS, crypto audit preparation, or plugin 56984. Delivers a compliance
  overview plus an affected-assets tab. Read-only.
---

# Skill: Tenable SSL & TLS - Compliance Dashboard

Performs an **SSL/TLS encryption compliance analysis** of the Tenable environment via MCP and delivers an
**interactive visual dashboard** (standalone HTML) in the conversation.

> ⚠️ **Plugin correction, validated against real data:** the user who originated this skill asked to cross
> "Plugin 56984 (SSL/TLS Cipher Suites Supported)" with "Plugin 45411 (SSL Certificate Information)". Running
> against a real tenant, we found that **45411 is actually "SSL Certificate with Wrong Hostname"** (it only
> compares the certificate CN against the hostname — no expiry date, no issuer, no signature algorithm). The
> plugin that actually delivers full certificate detail is **10863 — SSL Certificate Information**. This skill
> therefore uses 10863 as the primary certificate data source, plus the set of dedicated plugins Tenable already
> maintains for each specific condition (expired, self-signed, weak signature, expiring soon) — far more reliable
> than deriving those conditions by parsing free text. **45411 stays in the skill as a bonus finding** ("wrong
> hostname"), since it is relevant SSL hygiene — it just isn't the source for expiry/self-signed/signature.

Plugins used:

- **Plugin 56984** — SSL/TLS Versions Supported → protocols and ciphers each host accepts.
- **Plugin 10863** — SSL Certificate Information → Subject, Issuer, validity, signature algorithm, key size.
- **Plugin 15901** — SSL Certificate Expiry → already-expired certificate (with dates).
- **Plugin 42981** — SSL Certificate Expiry - Future Expiry → certificate expiring soon.
- **Plugin 57582** — SSL Self-Signed Certificate → self-signed certificate (explicit, with Subject).
- **Plugin 35291** — SSL Certificate Signed Using Weak Hashing Algorithm.
- **Plugin 86067** — SSL Certificate Signed Using SHA-1 Algorithm.
- **Plugin 45411** — SSL Certificate with Wrong Hostname → bonus finding (CN does not match the hostname).

Audience: Infrastructure/Security Engineers, PKI/certificate owners, Compliance, CISO. Typical use: give the
infrastructure team an actionable list **before** an auditor (PCI DSS, SOC 2, ISO 27001) forces the fix.

**Why this matters (business context):** TLS 1.0/1.1, weak ciphers (RC4, 3DES, NULL) and expired/self-signed
certificates are an automatic failure in most compliance frameworks (PCI DSS 4.0 requires TLS 1.2+; auditors
treat an expired certificate as a control failure). Finding these via scan and fixing them proactively is far
cheaper than finding them via an audit finding.

Mode: **Read-only** — diagnostic only. The skill **never** replaces certificates, disables protocols/ciphers,
restarts services, or modifies the environment. Recommendations are text, not actions.

---

## Prerequisite: Collect configuration BEFORE any data collection

Ask the user (in a single message):

> "Before I collect the data, a few quick questions:
> 1. **'Expiring soon' threshold** — how many days before expiry should a still-valid certificate be flagged as
>    'expiring soon'? (default: **60 days**)
> 2. **Scope** — all assets, or filter by a specific tag/subnet? (default: all)
> 3. **Dashboard language** — Portuguese or English? (default: conversation language)"

- If the user does not answer or asks for defaults, use `EXPIRING_SOON_DAYS = 60`, `SCOPE = "all"`, `LANG` =
  conversation language.
- Record `EXPIRING_SOON_DAYS`, `SCOPE_FILTER` and `LANG` and use them throughout the run.
- Do not invent a scope on your own. If the environment is large, offer to filter, but only if the user wants it.

---

## Execution Flow

Order: **config → aggregate counts → detailed protocol/cipher sample → detailed certificate sample → Crown Jewel
lookup → asset assembly → classification → render**.

### Step 0.5 — Aggregate counts per plugin (source of the header/card numbers)

> ⚠️ **Field lesson:** `workbenches_get_vulnerability_outputs(plugin_id)` returns at most ~10 detailed host
> blocks, followed by text like `"... and 50 more outputs"` — it does **not** list every affected host, even when
> the plugin has hundreds of findings. For the **real aggregate numbers** (top cards, %, score), use
> `workbenches_list_vulnerabilities` (once with no severity filter, then again with `severity: "info"`, because
> several of these plugins — 56984, 10863, 42981, 86067 — are **Info** severity and are excluded by the default
> filter) and locate each plugin's row by ID. Each row carries `Count` = number of findings (host+port instances)
> for that plugin across the environment — that is what feeds the "Overview" cards. If the result is too large for
> context, it is saved to a file — use the file search tool (grep) to extract only the rows of interest by plugin
> ID instead of trying to read the whole file.

Run this once for each plugin: **56984, 10863, 45411, 15901, 42981, 57582, 35291, 86067**. Store
`total_per_plugin[plugin_id] = Count`.

### Step 1 — Detailed protocol and cipher sample (Plugin 56984)

```
tool: workbenches_get_vulnerability_outputs
params: { plugin_id: 56984 }
```

The output arrives in per-host blocks listing the supported protocols and, within each protocol, the cipher
suites (typical format: protocol header — `SSLv2`, `SSLv3`, `TLSv1.0`, `TLSv1.1`, `TLSv1.2`, `TLSv1.3` — followed
by cipher lines with `Kx=... Au=... Enc=... Mac=...`). Since only ~10 hosts come back in detail per call (see
Step 0.5), treat this as a **real sample** to populate the Assets tab and the protocol distribution chart — not as
the complete list. Make this explicit in the dashboard (e.g., "sample of N hosts detailed in this run").

> If you need full coverage (production, not just a demo), use `workbenches_export_workbenches`
> (entity=vulnerabilities, plugin 56984) + `workbenches_download_export`, or iterate `scan_results(scan_id)` per
> scan — both return the full per-host text without the ~10-block limit.

For each host, parse:

**1a. Deprecated protocols present** — any protocol header equal to `SSLv2`, `SSLv3`, `TLSv1.0` or `TLSv1.1` in
the host block = deprecated protocol enabled. Store the exact list found.

**1b. Weak ciphers** — within any protocol (including TLS 1.2/1.3), mark a cipher line as weak if the name
contains any of: `RC4`, `DES` (includes `3DES`/`DES-CBC3`), `NULL`, `EXPORT`, `anon`/`ADH`/`AECDH`, `MD5` (as
Mac). Store the cipher name + the protocol where it appeared.

**1c. No Perfect Forward Secrecy (PFS)** — a cipher whose `Kx=` is **not** `ECDHE` or `DHE`/`EDH` = no PFS. Treat
this as lower severity than a deprecated protocol / classic weak cipher, but still report it.

**1d. Per-host classification:**
- 🔴 **Critical** — any deprecated protocol (SSLv2/v3, TLS 1.0/1.1) enabled.
- 🟠 **Warning** — no deprecated protocol, but weak cipher and/or no PFS.
- 🟢 **Healthy** — TLS 1.2+/1.3 only, with strong ciphers and PFS on every cipher.

### Step 2 — Detailed certificate sample (dedicated plugins, not free-text parsing)

Unlike the first version of this skill, **do not try to derive expired/self-signed/weak-signature from generic
text** — Tenable already has a plugin for each condition, and the output is structured and explicit. Run
`workbenches_get_vulnerability_outputs` for each one (the same ~10-detailed-host limitation from Step 0.5 applies
to all of them):

| Plugin | What the output already delivers |
|---|---|
| **10863** — SSL Certificate Information | Full block: `Subject Name`, `Issuer Name`, `Serial Number`, `Signature Algorithm`, `Not Valid Before`/`Not Valid After`, `Public Key Info` (algorithm + size in bits). Use as the source of truth for the detail drawer and for `Signature Algorithm`/key size. |
| **15901** — SSL Certificate Expiry | Only appears for hosts whose certificate is **already expired**. Ships `Subject`, `Issuer`, `Not valid before/after` ready to use — no need to compare dates yourself; the host's presence in this list is the classification. |
| **42981** — SSL Certificate Expiry - Future Expiry | Only appears for hosts whose certificate **will expire soon** (per Nessus's internal window, typically 60 days). To apply the user's exact `EXPIRING_SOON_DAYS`, cross the `Not Valid After` date from 10863 with the threshold; use 42981 as a confirmation/quick list. |
| **57582** — SSL Self-Signed Certificate | Only appears for hosts with a **self-signed** certificate — presence in the list = self-signed, with `Subject` already extracted. No need to compare Issuer==Subject manually. |
| **35291** — SSL Certificate Signed Using Weak Hashing Algorithm | Certificate signed with a weak hash (not SHA-256+). |
| **86067** — SSL Certificate Signed Using SHA-1 Algorithm | A specific SHA-1 subset. Some hosts appear in both 35291 and 86067 — deduplicate by host when counting "weak signature/key". |
| **45411** — SSL Certificate with Wrong Hostname (bonus finding) | The certificate CN does not match the host's hostname. Report as a separate category — it is not the same as self-signed or expired, but it is relevant SSL hygiene. |

**2a. Expired** = host present in the **15901** findings list (or `Not Valid After` from 10863 already past).

**2b. Expiring soon** = host present in **42981**, OR (from 10863) `Not Valid After − now <= EXPIRING_SOON_DAYS`
days and not yet expired.

**2c. Self-signed** = host present in the **57582** findings list.

**2d. Weak signature/key** = host present in **35291** and/or **86067**; or, from 10863, `Signature Algorithm`
contains `SHA-1`/`MD5`, or the public key is RSA/DSA < 2048 bits.

**2e. Wrong Hostname (bonus)** = host present in **45411**.

> ⚠️ If a host has more than one certificate (leaf + intermediates) in the 10863 block, evaluate the **leaf**
> (the service's own certificate, normally the first in the block) for the classifications above.

**2f. Per-host classification:**
- 🔴 **Critical** — expired certificate (15901).
- 🟠 **Warning** — self-signed (57582), expiring soon (42981), weak signature/key (35291/86067), or wrong hostname
  (45411) — any of them.
- 🟢 **Healthy** — does not appear in any of the plugins above.

### Step 3 — Identify Crown Jewel assets

Combine two sources (tag preferred, ACR as fallback):

**3a. Tag:**
```
tool: tagging_list_tag_categories_and_values
```
Look for a tag category equivalent to "Crown Jewel" (e.g., `Crown Jewel`, `Crown Jewels`, `Joia da Coroa`,
`Critical Asset`). If found, collect the values and, when assembling the asset table, mark as Crown Jewel
(source = "Tag") any asset carrying that tag.

**3b. ACR (fallback):** for assets **without** the tag above, treat as Crown Jewel (source = "ACR") if
`acr_score >= 9`. This covers environments that do not yet formally tag their critical assets.

**3c.** If an asset has both signals, show source = "Tag + ACR".

If the tag category does not exist in the tenant, do not block execution — use the ACR criterion only and make
that transparent in the dashboard (note: "No Crown Jewel tag found in the tenant; classification by ACR ≥ 9").

> ⚠️ **Field lesson:** in a real test, neither a Crown Jewel tag nor ACR was available for the Windows/Linux hosts
> typically affected by SSL/TLS findings — `tenable_one_search_assets` returned `ACR: N/A` for them. ACR is often
> populated only on a subset of assets (e.g., OT/ICS assets in a Tenable.ot environment, or assets with Lumin
> explicitly calculated). **Do not assume every asset will have an ACR** — treat `N/A`/missing as "signal
> unavailable", not as "not a Crown Jewel". If no asset in the pool has a tag or an ACR, be transparent in the
> dashboard rather than forcing a classification: "Crown Jewel could not be determined for X of Y assets — neither
> tag nor ACR available on those hosts."

### Step 4 — Assemble the applicable asset pool

Goal: every asset appearing in at least one of the Step 1/2 plugins (56984, 10863, 15901, 42981, 57582, 35291,
86067, 45411) goes into the Assets tab.

For each asset, collect via `tenable_one_search_assets` (or `workbenches_get_asset_info` by `asset_id`/hostname
extracted from the plugin outputs):
- Asset name (`name`/hostname)
- Operating System (`os`)
- IP address (`ipv4`)
- ACR (`acr_score`)
- Crown Jewel + source (Step 3)
- Relevant tags
- Last Seen
- Summary of the asset's findings: deprecated protocols found, number of weak ciphers, no PFS (yes/no),
  certificate status (expired / self-signed / expiring soon / weak signature / OK)
- Overall asset status (🔴/🟠/🟢) = the worse of the protocol/cipher classification (Step 1d) and the certificate
  classification (Step 2f)

If `SCOPE_FILTER` was set by the user (tag/subnet), apply the filter here before continuing.

### Step 5 — Calculate percentages and the overall score

Two levels of number coexist in the dashboard — make clear which is which:

- **Top cards (real environment totals):** use `total_per_plugin[plugin_id]` from Step 0.5 directly — these are
  the official finding counts (host+port instances) per category across the whole environment, independent of the
  ~10-host sample. E.g., "106 expired certificates" comes straight from `total_per_plugin[15901]`.
- **Detailed sample (distribution charts, Assets tab, Top Offenders):** uses the hosts actually collected in
  Steps 1/2 (~10 per plugin, or more if you used export). Always label it as a sample and state the size (e.g.,
  "sample of N hosts detailed in this run") — do not present it as if it were the total.

About the asset sample (`total_evaluated` = union of unique assets seen in any of the Step 4 plugins):

- `% Deprecated Protocol` = sample assets with ≥1 deprecated protocol / sample total
- `% Weak Cipher` = sample assets with ≥1 weak cipher / sample total
- `% No PFS` = sample assets with ≥1 cipher without PFS / sample total
- `% Expired Certificate` = sample assets with ≥1 expired cert / sample total
- `% Self-Signed Certificate` = sample assets with ≥1 self-signed cert / sample total
- `% Expiring Soon` = sample assets with ≥1 cert expiring within ≤ `EXPIRING_SOON_DAYS` days / sample total
- `% Weak Signature/Key` = sample assets with ≥1 cert with weak signature/key / sample total
- `SSL/TLS Compliance Score` = % of sample assets that are 🟢 Healthy (no protocol/cipher and no certificate
  problem) over the sample total — make clear this is an illustrative score based on the sample, not the entire
  environment, unless you collected via a full export.

---

## Step 6 — Render the Dashboard

Generate **a single standalone HTML file** (inline styles and scripts — do **not** use React/artifact), save it to
`outputs/` and deliver it via `present_files`.

### Structure (2 main views: Compliance Overview + Assets)

**Header:** title "Tenable — SSL/TLS Compliance" + **Compliance Score** (gauge/ring) + the "expiring soon"
threshold used + timestamp + total count of assets evaluated.

**Top tabs:** `[ Overview ]` `[ Applicable Assets ]`

#### Tab 1 — Overview

**Summary Cards (percentages, at the top, responsive grid):**
1. **Deprecated Protocol** — % + count, red if > 0.
2. **Weak Cipher** — % + count, orange.
3. **No PFS** — % + count, orange.
4. **Expired Certificate** — % + count, red.
5. **Self-Signed Certificate** — % + count, orange.
6. **Expiring in ≤ N days** — % + count, orange (N = `EXPIRING_SOON_DAYS`).
7. **Weak Signature/Key** — % + count, orange.
8. **Wrong Hostname (bonus)** — % + count, orange (Plugin 45411).
9. **SSL/TLS Compliance Score** — % 🟢 healthy, highlighted card with a mini-bar 🟢/🟠/🔴.

**Additional useful views (below the cards):**
- **Protocol Distribution** — horizontal bar chart counting how many hosts support each protocol (SSLv2, SSLv3,
  TLS 1.0, TLS 1.1, TLS 1.2, TLS 1.3), with deprecated-protocol bars in red/orange and TLS 1.2/1.3 in green.
- **Certificate Issue Breakdown** — donut chart with slices: Expired / Self-Signed / Expiring Soon / Weak
  Signature or Key / OK.
- **Crown Jewel Exposure** — highlight card: "`X` of `Y` Crown Jewels have SSL/TLS findings" with a link that
  filters Tab 2 to affected Crown Jewels only. Treat this as top remediation priority.
- **Top offenders** — compact table of the 10 assets with the most combined issues (protocol + cipher + cert),
  sorted by finding count desc, each row clickable → opens the asset detail drawer.
- **Recommendations (read-only, text):** action cards, shown only when applicable:
  - Deprecated protocol > 0: "X assets accept SSLv2/v3 or TLS 1.0/1.1. Disable those protocols and enforce TLS 1.2+."
  - Weak cipher > 0: "Y assets accept weak ciphers (RC4/3DES/NULL/export/anonymous). Reconfigure the cipher suite list."
  - No PFS > 0: "Z assets do not offer PFS. Prioritize ECDHE/DHE suites."
  - Expired cert > 0: "W expired certificates — outage risk and audit failure. Renew immediately."
  - Self-signed > 0: "V self-signed certificates in use — replace with a trusted CA (internal or public)."
  - Expiring soon > 0: "U certificates expire within N days — schedule renewal now."
  - Weak signature/key > 0: "T certificates use SHA-1/MD5 or a key < 2048 bits — reissue with SHA-256+/RSA 2048+ (or ECC)."
  - All healthy: "✅ Environment compliant with corporate encryption standards."

#### Tab 2 — Applicable Assets

Interactive table with **all** assets from Step 4. Columns:

| Asset | OS | IP | Crown Jewel | ACR | Deprecated Protocols | Weak Ciphers | No PFS | Certificate Status | Overall Status |
|---|---|---|---|---|---|---|---|---|---|

- **Crown Jewel** column: 👑 badge if yes (with a tooltip for the source: Tag/ACR/Tag+ACR), empty if not.
- **Overall Status** column: 🔴/🟠/🟢 (Step 4).
- Quick filters: `[ All ]` `[ 👑 Crown Jewels ]` `[ 🔴 Deprecated Protocol ]` `[ 🟠 Weak Cipher ]`
  `[ 🔴 Expired Cert ]` `[ 🟠 Self-Signed ]` `[ 🟠 Expiring Soon ]` · search by name/IP · click-to-sort on any
  column · pagination if > 50 rows.
- Clicking a row opens a **drawer** with: all the asset's tags, full detail of the protocols/ciphers found (list
  with Kx/Au/Enc/Mac), certificate detail (subject, issuer, validity, algorithm, key size), and a specific fix
  hint per finding.

### Design — Mandatory Palette (same as the other Tenable skills)

| Role | HEX | Usage |
|---|---|---|
| Background | `#44494B` | Page background, cards, tables |
| Highlight | `#E7FF00` | Titles, key values, primary labels |
| White | `#FFFFFF` | Secondary text, icons |
| Blue | `#4EA5FF` | Links, buttons, active filters, active columns |
| Green | `#71FFC6` | Healthy status 🟢 |
| Purple | `#BB8FF2` | Crown Jewel badge 👑 |
| Orange | `#FF8837` | Warning status 🟠 |
| Red (derived) | `#FF4B4B` | Critical 🔴 — use sparingly |

Rules: background and cards `#44494B`; titles and primary numbers `#E7FF00`; body text `#FFFFFF`; card borders
`1px solid rgba(231,255,0,0.20)`; row hover `rgba(78,165,255,0.10)`; Inter/system-ui font; cards
`border-radius:10px` + `box-shadow:0 2px 12px rgba(0,0,0,0.4)`; fully responsive; horizontal scroll on tables in
mobile.

---

## Error Handling

| Situation | Action |
|---|---|
| Tenable MCP unavailable | Inform the user and offer to generate with mock data for demonstration |
| No assets returned by the SSL/TLS plugins | "No SSL/TLS findings on these plugins. Check whether there are recent scans covering TLS ports." |
| `workbenches_get_vulnerability_outputs` only returns ~10 detailed hosts | Expected (see Step 0.5) — use `workbenches_list_vulnerabilities` for the totals and label the sample as a sample. For full coverage, use export/scan_results |
| `workbenches_list_vulnerabilities` missing the expected plugin | The plugin is probably Info severity (the case for 56984, 10863, 42981, 86067) — call again passing `severity: "info"` |
| Crown Jewel tag category does not exist in the tenant | Use the ACR ≥ 9 criterion only and note this in the dashboard |
| ACR returns `N/A` for the affected assets | Do not classify as "not a Crown Jewel" — report as "not determined" and note in the dashboard that neither tag nor ACR was available |
| Certificate without a readable `Not Valid After` in 10863 | Skip the expiry classification for that certificate, but keep the others (signature/key, self-signed via 57582) |
| Host present in one plugin but absent in another | Normal — show only the applicable columns; do not flag as an error |
| Timeout during collection | Use the partial data and flag it in the header: "⚠️ Partial data — timeout during collection" |

---

## Important Notes

- **Read-only:** no action is executed in the environment. No disabling protocols, replacing certificates, or
  restarting services.
- **Configurable threshold:** `EXPIRING_SOON_DAYS` always comes from the user's answer (default 60).
- **Leaf vs. chain:** certificate classification is about the host's own certificate (leaf), not the
  intermediate/root CA.
- **Crown Jewel source transparency:** always make clear whether it came from a tag, ACR, or both.
- **Language:** respond and label the dashboard in the chosen language (`LANG`), default = conversation language.
- **Not the same as "generic SSL vulnerability":** the focus here is specifically protocol/cipher (56984) and the
  certificate plugin set (10863, 15901, 42981, 57582, 35291, 86067, 45411) — do not mix in other SSL plugins
  unless the user asks.
- **Sample vs. total:** always make clear in the dashboard when a number comes from the aggregate environment
  total (Step 0.5) and when it comes only from the detailed host sample collected in this run (Steps 1/2).

---

## Usage Context

Used by **Infrastructure/Security** teams to:
- Quickly surface every internal server that falls outside the corporate encryption standard (TLS 1.2+, strong
  ciphers, valid and non-self-signed certificates).
- Prioritize remediation by Crown Jewel ahead of lower-criticality assets.
- Prepare the environment ahead of an audit (PCI DSS, SOC 2, ISO 27001) covering encryption-in-transit controls.
- Give the infrastructure team an actionable asset list + technical evidence (protocol/cipher/certificate) to fix
  without hunting it down manually in Tenable.

---

## Content Agreement
I have reviewed and accept the CyberAgents Exchange Contribution Agreement
  
