# The Shai-Hulud Worm — Static Analysis of a Live Blockchain-C2 Supply-Chain Campaign

**Campaign:** Shai-Hulud variant `9-5077-2` (variant `9-4659` also observed)
**Active since:** ~December 2025
**Last observed C2 rotation:** 2026-05-23 (4 days before this analysis)
**Affected ecosystem:** npm / JavaScript supply chain — primarily projects using Tailwind CSS
**Analyst note:** All findings below were obtained via **static analysis only**. At no point was any worm payload evaluated as JavaScript. Every layer was unwrapped with Python re-implementations of the worm's own decoders. Reproduction recipes are included in Section 11.

---

## TL;DR (one paragraph)

A supply-chain worm targeting JavaScript/Node projects has been spreading via injected JavaScript appended to `tailwind.config.js` (and `postcss.config.js`, `.eslintrc.js` variants) in popular open-source repositories. The malicious code is triggered silently the moment any developer opens an infected project in an IDE with the Tailwind CSS IntelliSense extension installed — no click required. Once running, it exfiltrates SSH keys, GitHub tokens, browser passwords, and `.env` secrets to one of three attacker-controlled IPs (**primarily `http://166.88.54.158`**). The novel architecture stores all command-and-control via public blockchain reads (**Tron, Aptos, and Binance Smart Chain**), making the C2 channel effectively un-takedownable: the attacker pushes ~$0.0003 dust transactions to rotate payloads. Propagation is achieved by `git commit --amend && git push --force-with-lease` on every repo the infected developer can write to, preserving the original author's name on the resulting commits — making detection by audit log alone impossible without commit signing.

This document is the result of unwrapping the worm's 5+ layers of nested obfuscation statically and extracting hard, durable IOCs.

---

## Table of contents

1. [Plain-English explanation](#1-plain-english-explanation)
2. [Threat severity assessment](#2-threat-severity)
3. [Observed timeline](#3-observed-timeline)
4. [Self-check: am I affected?](#4-self-check)
5. [Full attack chain](#5-full-attack-chain)
6. [Indicators of Compromise](#6-indicators-of-compromise)
7. [Live URLs anyone can verify](#7-live-urls)
8. [Defensive actions](#8-defensive-actions)
9. [How to check a developer machine for past infection](#9-forensic-check)
10. [Glossary](#10-glossary)
11. [Technical appendix — reproducing the static decode](#11-technical-appendix)

---

## 1. Plain-English explanation

### What's a worm?

A computer worm is software that spreads from machine to machine on its own — like a chain letter that copies itself, except invisible. You don't have to click anything; just opening the wrong file at the wrong time is enough.

### What does this worm do?

Three things:

1. **Hides** inside `tailwind.config.js` — a file every modern web project that uses Tailwind CSS already has, looking like normal JavaScript config.
2. **Steals** SSH keys, GitHub tokens, browser passwords, and `.env` files (which contain database passwords, API keys, payment-processor secrets).
3. **Spreads** to other projects the infected developer can push to, by appending itself to *their* config files and force-pushing automatically.

### What makes it especially sneaky?

**(A) It uses your teammate's name when it pushes.** When the worm spreads, it creates Git commits that show the original author's name — because `git commit --amend` preserves the author by default. So even when looking at GitHub, the bad commits appear to come from your trusted colleague.

**(B) It doesn't have a "server" to take down.** Most malware talks to an attacker's server — security teams find the server, take it down, malware dies. This worm hides its instructions on **public blockchains** (Tron, Aptos, Binance Smart Chain). Those are run by legitimate companies serving millions of users. The attacker just writes their commands into a free public ledger that nobody can take down. Stop one server? They write a new command in 10 seconds for $0.0003.

### Analogy

Imagine a virus that:
- Lives inside the "Settings" tab of your design app (silent, no popup)
- Steals your house keys, car keys, and bank cards (credentials)
- Sneaks copies of itself into your friends' design files using YOUR fingerprint on the package (preserved commit authorship)
- Gets its updates from notes pinned to a public bulletin board at City Hall (blockchain) instead of a hacker hideout (no IP to seize)

That's this worm.

---

## 2. Threat severity

| Question | Answer |
|---|---|
| Is the worm currently active? | **Yes.** Last C2 rotation: 4 days prior to publication. |
| Does it require user interaction (clicking, etc.)? | **No.** Trigger fires the moment an IDE language-server (Tailwind / PostCSS / ESLint) evaluates an infected config — typically on project open or file save. |
| Are credentials stolen? | **Yes**, for any developer machine where the worm has executed. SSH keys, GitHub tokens, npm tokens, AWS/GCP creds, browser passwords, `.env` secrets — all candidates. |
| How does it spread? | Via force-pushed amended Git commits on every repo the infected developer can push to. |
| Can the underlying campaign be taken down? | **Not via traditional means.** The C2 lives on public blockchains. The exfiltration servers (3 known IPs) can be blocked but are likely VPS rentals trivially replaceable by the attacker. |
| What's the structural defense? | **Signed commits + branch protection requiring signatures.** This prevents the worm's force-pushed amended commits from being accepted even when a developer machine is infected. |

**Bottom line:** Moderately severe supply-chain compromise vector. Mitigations are well-understood and can be deployed within hours.

---

## 3. Observed timeline

- **~Dec 2025** — Earliest beacon transactions appear on Tron blockchain (first evidence of operational campaign).
- **May 2026** — Active infections observed in multiple repositories across a popular WordPress block-plugin ecosystem. Developer machines confirmed compromised in at least two cases.
- **2026-05-13** — Worm-laced commit pushed to a major plugin repository. Reverted by a vigilant maintainer within 24h.
- **2026-05-14** — Multiple "security: remove obfuscated payload from admin/tailwind.config.js" commits appear in cleanup wave across the plugin ecosystem.
- **2026-05-23** — Latest observable C2 rotation on the Tron blockchain (see Section 7 for transaction URL).
- **2026-05-27** — Two new infected PRs observed; static analysis of this document completed; report published.

The worm is in an **active rotation** phase: payloads are being updated approximately every 4–7 days based on observed beacon transactions.

---

## 4. Self-check: am I affected?

Run this single command in Terminal on macOS:

```bash
ls -la ~/.node_modules 2>/dev/null && echo "⚠️ WORM RUNTIME DIRECTORY FOUND" || echo "✅ No worm runtime dir"
find / -iname "config.bat" 2>/dev/null | head -5
pgrep -fl "node -e" 2>/dev/null && echo "⚠️ ACTIVE node -e PROCESS"
grep -rlE "global\[['\"]\![\"']\][[:space:]]*=" ~ --include="*.js" --include="*.ts" 2>/dev/null | head -5
```

If all four lines say "no" / blank → most likely clean.

If anything produces a hit → disconnect from network immediately and follow [Section 8 — Defensive Actions](#8-defensive-actions).

For deeper inspection on macOS, a community read-only scanner is available at https://github.com/anthropics/shai-hulud-checker (or use the equivalent forensic procedure in Section 9).

---

## 5. Full attack chain

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1 — Infection vector                                       │
│  Developer pulls a Git branch containing an infected             │
│  `tailwind.config.js`. The file is hundreds of lines of normal   │
│  config, followed by HUGE WHITESPACE PADDING, then a wall of     │
│  obfuscated JavaScript at the end. Reviewers scrolling the file  │
│  don't see it because it's pushed off-screen.                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 2 — Trigger (silent, no click needed)                      │
│  The moment any of these runs the config as JavaScript:          │
│   • VS Code Tailwind IntelliSense extension (most common)        │
│   • Cursor / VSCodium / Windsurf with the same extension         │
│   • `npm run build`, `npm run dev`, `tailwindcss --watch`        │
│   • Any pre-commit hook that lints or builds                     │
│  → The worm fires. No prompt. No popup.                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 3 — Fetch the actual payload from blockchain               │
│                                                                  │
│  Worm sends HTTPS GET to:                                        │
│    https://api.trongrid.io/v1/accounts/                          │
│      TMfKQEd7TJJa5xNZJZ2Lep838vrzrs7mAP/transactions?limit=1     │
│                                                                  │
│  ── reads the latest "Note" / data field on this Tron wallet     │
│  ── reverses the text → gets a Binance Smart Chain transaction   │
│     hash (e.g., 0x80a1148ee589125b...)                           │
│                                                                  │
│  Worm sends JSON-RPC call to:                                    │
│    https://bsc-dataseed.binance.org                              │
│      method: eth_getTransactionByHash                            │
│      params: [<the hash from above>]                             │
│                                                                  │
│  ── gets back the BSC transaction's "input" field, which is      │
│     the encrypted JavaScript payload (~6 KB)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 4 — Decrypt the payload                                    │
│                                                                  │
│  Worm has a 16-byte XOR key baked in: "2[gWfGj;<:-93Z^C"         │
│  → XOR decrypts the BSC input → JavaScript source code           │
│  → `eval()` runs that code                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 5 — Decide which attacker IP to talk to                    │
│                                                                  │
│  The decrypted code reads a campaign tag (`global._V`) and       │
│  routes to ONE OF THREE servers:                                 │
│                                                                  │
│   • _V starts with 'A' (CURRENT case for variant 9-5077-2):      │
│       → 166.88.54.158       ⭐ PRIMARY EXFIL                     │
│   • _V is a plain number:                                        │
│       → 198.105.127.210                                          │
│   • Otherwise:                                                   │
│       → 23.27.202.27        (also runs MongoDB on :27017)        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 6 — Steal credentials                                      │
│                                                                  │
│  The worm reads:                                                 │
│   • ~/.ssh/id_rsa, id_ed25519, known_hosts, config               │
│   • ~/.gitconfig, ~/.git-credentials                             │
│   • ~/.config/gh/hosts.yml      (GitHub CLI token)               │
│   • ~/.npmrc                    (npm publish token)              │
│   • ~/.aws/credentials                                           │
│   • ~/.kube/config                                               │
│   • ~/.docker/config.json                                        │
│   • Every `.env*` file in ~/Projects, ~/Code, ~/Sites,           │
│     ~/Local Sites, ~/Documents, ~/Desktop                        │
│   • Shell history (~/.zsh_history, ~/.bash_history)              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 7 — Send stolen data to attacker                           │
│                                                                  │
│  HTTPS POST to http://166.88.54.158:443                          │
│  Body: a JSON bundle of everything from Step 6                   │
│                                                                  │
│  Connection happens via the `axios` library that the worm        │
│  installs into a hidden `~/.node_modules/` directory.            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 8 — Propagate to other repos                               │
│                                                                  │
│  Worm finds every Git repo in dev folders.                       │
│  For each one the developer has push access to:                  │
│   1. Append the obfuscated payload to                            │
│      tailwind.config.js / postcss.config.js / .eslintrc.js       │
│   2. Add "config.bat" line to .gitignore                         │
│   3. Run `git commit --amend --no-edit`                          │
│      ↑ this PRESERVES the original author's name. The bad        │
│        commits show the *previous committer's* name even though  │
│        the worm wrote the code.                                  │
│   4. Run `git push --force-with-lease`                           │
│                                                                  │
│  → Other developers pull the branch → their IDE evaluates the    │
│  config → worm fires on THEIR machine. The cycle repeats.        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 9 — Self-cleanup                                           │
│  Worm deletes ~/.node_modules/. Visible footprint gone in        │
│  seconds. Only the payload file in the repo and the config.bat   │
│  line in .gitignore remain — those are the "seeds" for future    │
│  infections.                                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Why the file is the propagation vehicle, not a server

In traditional malware, removing the file from one machine kills it. In this worm, the file in the Git repository is the *vector* — the way new victims catch it. Even if every developer machine is cleaned, the next pull of an infected branch restarts the cycle.

**The fix has to be at the Git repository level**, not just per-machine. That's why "signed commits + branch protection" is the structural fix.

---

## 6. Indicators of Compromise

### ⭐ ATTACKER IPs — block immediately at firewall

| IP | Used when | Notes |
|---|---|---|
| **`166.88.54.158`** | Campaign tag starts with `A` (CURRENT) | Primary exfil — ports 80 & 443 |
| `198.105.127.210` | Campaign tag is a plain number | Secondary exfil |
| `23.27.202.27` | Fallback (else) | Also runs MongoDB on port 27017 (likely the attacker's database) |

### Legitimate hostnames the worm queries (block egress from dev workstations only)

| Host | What it really is | Why we block |
|---|---|---|
| `api.trongrid.io` | Tron Foundation's public blockchain API | Worm reads payload pointers from here |
| `fullnode.mainnet.aptoslabs.com` | Aptos Labs' public blockchain node | Fallback for above |
| `bsc-dataseed.binance.org` | Binance's BSC RPC node | Worm reads the encrypted payload from here |
| `bsc-rpc.publicnode.com` | PublicNode's BSC RPC | Fallback for above |

> Block these **for dev workstations only** — backend services may legitimately need them.

### Blockchain wallet IOCs

| Type | Address | Role |
|---|---|---|
| Tron | `TMfKQEd7TJJa5xNZJZ2Lep838vrzrs7mAP` | Beacon #1 (outer Layer-4 pointer) |
| Tron | `TXfxHUet9pJVU1BgVkBAbrES4YUc1nGzcG` | Beacon #2 (fallback) |
| Tron | `TA48dct6rFW8BXsiLAtjFaVFoSuryMjD3v` | Beacon #3 (inner Layer-6 pointer) |
| Tron | `TCqf6ZkaQD84vYsC2cu…veTaRrF` | **Funder wallet — attribution pivot point** |
| Aptos | `0xbe037400670fbf1c32364f762975908dc43eeb38759263e7dfcdabc76380811e` | Beacon #1 |
| Aptos | `0x3f0e5781d0855fb460661ac63257376db1941b2bb522499e4757ecb3ebd5dce3` | Beacon #2 |
| Aptos | `0x533b2dbcaeff19cd1f799234a27b578d713d8fcaa341b7501e4526106483e0b1` | Beacon #3 |

### Cryptographic keys (XOR keys used by the worm)

| Key | Used for |
|---|---|
| `2[gWfGj;<:-93Z^C` | Primary path BSC payload decryption |
| `m6:tTh^D)cBz?NM]` | Secondary path BSC payload decryption |

### File-level signatures (grep patterns for code repos)

```regex
# Definitive: campaign version key
global\[['\"]\![\"']\][[:space:]]*=[[:space:]]*['\"][A-Za-z0-9_]+-[A-Za-z0-9_]+['\"]

# Definitive: shuffle-decoder structure
\(function[[:space:]]*\([a-z],[a-z]\)[[:space:]]*\{[[:space:]]*var [a-z]=[a-z]\.length

# Almost-definitive: .gitignore added line
^config\.bat$

# IDE/runtime artifact
~/.node_modules/   (this directory should never exist on a normal developer Mac)
```

### Runtime globals on Node processes (forensic evidence of past execution)

```
global['!']             = '9-5077-2' (or '9-4659' for earlier variant)
global['_V']            = 'A9-5077-2'
global['_p_t']          = <timestamp>         (outer rate-limit)
global['_t_t']          = <timestamp>         (inner rate-limit)
global['_t_0']          = <atob-decoded loader>
global['_t_1']          = <currently set Tron address>
global['_t_2']          = <currently set Aptos address>
global['_t_s']          = 'http://166.88.54.158:443'   ← EXFIL TLS endpoint
global['_t_u']          = 'http://166.88.54.158'       ← EXFIL plain endpoint
global['_t_c']          = <cached fetcher function source>
global['___dirname']    = <Node module directory at infection time>
global['___filename']   = <Node module filename at infection time>
```

### Infected files to look for (the propagation vector)

```
tailwind.config.js
postcss.config.js
.eslintrc.js
.gitignore   (specifically: a line that is exactly "config.bat")
```

Plus a stray file `config.bat` anywhere in a repo or on disk.

---

## 7. Live URLs

> Open in a browser — do NOT curl from a developer workstation. Blockchain queries to the attacker's beacons could expose the inspector's IP to the campaign operator's monitoring.

### Investigate the attacker IPs (intel pivot)

| Tool | URL |
|---|---|
| VirusTotal — `166.88.54.158` | https://www.virustotal.com/gui/ip-address/166.88.54.158 |
| VirusTotal — `198.105.127.210` | https://www.virustotal.com/gui/ip-address/198.105.127.210 |
| VirusTotal — `23.27.202.27` | https://www.virustotal.com/gui/ip-address/23.27.202.27 |
| AbuseIPDB — `166.88.54.158` | https://www.abuseipdb.com/check/166.88.54.158 |
| AbuseIPDB — `198.105.127.210` | https://www.abuseipdb.com/check/198.105.127.210 |
| AbuseIPDB — `23.27.202.27` | https://www.abuseipdb.com/check/23.27.202.27 |
| Shodan — `166.88.54.158` | https://www.shodan.io/host/166.88.54.158 |
| Shodan — `198.105.127.210` | https://www.shodan.io/host/198.105.127.210 |
| Shodan — `23.27.202.27` | https://www.shodan.io/host/23.27.202.27 |
| BGP / ASN — `166.88.54.158` | https://bgp.he.net/ip/166.88.54.158 |
| BGP / ASN — `198.105.127.210` | https://bgp.he.net/ip/198.105.127.210 |
| BGP / ASN — `23.27.202.27` | https://bgp.he.net/ip/23.27.202.27 |

### Trace the attacker wallets

| Tool | URL |
|---|---|
| Tronscan — Beacon #1 | https://tronscan.io/#/address/TMfKQEd7TJJa5xNZJZ2Lep838vrzrs7mAP |
| Tronscan — Beacon #2 | https://tronscan.io/#/address/TXfxHUet9pJVU1BgVkBAbrES4YUc1nGzcG |
| Tronscan — Beacon #3 | https://tronscan.io/#/address/TA48dct6rFW8BXsiLAtjFaVFoSuryMjD3v |
| Aptoscan — Beacon #1 | https://aptoscan.com/account/0xbe037400670fbf1c32364f762975908dc43eeb38759263e7dfcdabc76380811e |
| Aptoscan — Beacon #2 | https://aptoscan.com/account/0x3f0e5781d0855fb460661ac63257376db1941b2bb522499e4757ecb3ebd5dce3 |
| Aptoscan — Beacon #3 | https://aptoscan.com/account/0x533b2dbcaeff19cd1f799234a27b578d713d8fcaa341b7501e4526106483e0b1 |

### View live malicious transactions

| What | URL |
|---|---|
| Latest C2 beacon (2026-05-23) on Tron | https://tronscan.io/#/transaction/914e5e0741f7f3bc725dd3345f8f7a72770c48c2f9f58c3799e773d433c3e6e6 |
| Current encrypted payload on BSC | https://bscscan.com/tx/0x80a1148ee589125bc1e57d36abac9f08089b2990d9372be3a33a1f057ad1ef89 |

### Submit to community threat intel

| Service | Where |
|---|---|
| VirusTotal | https://www.virustotal.com/gui/home/upload — submit one infected `tailwind.config.js` |
| AbuseIPDB | https://www.abuseipdb.com/report — report each of the 3 attacker IPs |
| urlhaus (Abuse.ch) | https://urlhaus.abuse.ch/api/ — submit `http://166.88.54.158/` etc. |

---

## 8. Defensive actions

### Priority 1 — DEPLOY NOW (5–60 min each)

1. **Block the 3 attacker IPs at corporate firewall:**
   ```
   166.88.54.158/32    DROP outbound
   198.105.127.210/32  DROP outbound
   23.27.202.27/32     DROP outbound
   ```
   Includes ports 80, 443, and 27017.

2. **Block the 4 blockchain APIs at firewall for developer subnets** (whitelist for backend services that legitimately need them):
   ```
   api.trongrid.io
   fullnode.mainnet.aptoslabs.com
   bsc-dataseed.binance.org
   bsc-rpc.publicnode.com
   ```

3. **Per-developer layer (defense in depth):** install [Little Snitch](https://www.obdev.at/products/littlesnitch/index.html) or [LuLu](https://objective-see.org/products/lulu.html) (free) and add the above hosts/IPs to deny lists.

4. **Audit GitHub org for the worm signature:**
   ```bash
   gh api search/code --paginate -q '.items[] | .repository.full_name + ": " + .path' \
     -f q="\"global['!']='\" org:<your-org>"
   ```
   Any hits → treat that repo as infected, revert offending commits, audit affected branches.

5. **For confirmed-infected developer machines:**
   - Take them off the network immediately.
   - From a clean device (phone or different computer), rotate **everything**: GitHub PATs, SSH keys, npm tokens, AWS/GCP creds, browser saved passwords, password manager master, all app sessions (Slack/Notion/Linear/Jira/Stripe).
   - The machine should be **wiped and macOS reinstalled**. Half-measures (just deleting `~/.node_modules` + the infected files) are NOT safe — the worm may have planted hooks we haven't detected.

### Priority 2 — DEPLOY THIS WEEK (1–4 hours each)

6. **Enforce signed commits on protected branches** in every repo. For each repo: Settings → Branches → Add rule → check:
   - ☑ Require a pull request before merging
   - ☑ Require **signed commits**
   - ☑ Require status checks to pass
   - ☑ Do not allow bypassing the above

   This is the **structural fix**. Once enforced, an unsigned commit (which is what the worm's amend-and-force-push produces) is rejected at the GitHub server level. The propagation cycle stops even if a developer machine remains infected.

7. **Every developer sets up Touch ID + Secure Enclave signed commits.** On macOS, use [Secretive](https://github.com/maxgoedjen/secretive) to host an SSH key in the Secure Enclave with "Require Authentication: Every Time Used". Touch ID prompts on every `git push` and `git commit`. The private key cannot be exfiltrated by malware — it never leaves the chip.

8. **Mandatory hardware MFA (YubiKey or equivalent) on GitHub** for all developers: https://github.com/settings/security-keys

9. **Threat intel submission** (Section 7) — submit IPs, wallets, and a payload sample.

### Priority 3 — STRUCTURAL (1–4 weeks)

10. **GitHub Action to scan PRs against the IOC regexes** (Section 6). Auto-comments on hits, blocks merge.

11. **Blockchain wallet monitoring:** alert when the beacon wallets receive a new transaction (= new C2 rotation). Services: [Tatum](https://tatum.io/), [Moralis](https://moralis.io/), or a self-hosted Tronscan/BSCScan webhook listener.

12. **Subpoena trail (with legal team):** the funder wallet eventually traces back to a centralized exchange withdrawal. CEXes do KYC. A subpoena via local CERT / law enforcement can reveal the attacker's KYC identity.

13. **EDR/AV on developer machines:** [CrowdStrike Falcon](https://www.crowdstrike.com/) or [SentinelOne](https://www.sentinelone.com/) will catch the worm's `~/.node_modules/` write and the connection to attacker IPs.

---

## 9. Forensic check

If you suspect a developer machine was infected:

### Step 1 — Disconnect them from the network
Turn off Wi-Fi, unplug ethernet. Cuts any active C2 connection.

### Step 2 — Run the read-only checks

```bash
# Live connections to attacker IPs RIGHT NOW
lsof -i -n -P | grep ESTABLISHED | grep -E "166\.88\.54\.158|198\.105\.127\.210|23\.27\.202\.27"

# Past DNS lookups (last 14 days, log auto-rotates after that)
log show --predicate '(process == "node" OR processImagePath CONTAINS "node")' --info --last 14d 2>/dev/null \
  | grep -iE "trongrid|aptoslabs|bsc-dataseed|166\.88\.54\.158" | head -20

# Past `node -e` process spawns
log show --predicate 'process == "node"' --info --last 14d 2>/dev/null \
  | grep -iE "node -e|child_process" | head -20

# Suspicious shell-rc additions
grep -nE "curl.*\| *(bash|sh)|node -e|atob|global\['" \
  ~/.zshrc ~/.zshenv ~/.zprofile ~/.bashrc ~/.bash_profile ~/.profile 2>/dev/null

# Suspicious LaunchAgents (persistence)
grep -lE "node -e|global\[|curl.*\|.*sh" \
  ~/Library/LaunchAgents/*.plist /Library/LaunchAgents/*.plist 2>/dev/null

# tailwind.config.js mtimes — anything modified spontaneously is suspicious
find ~ -name "tailwind.config.js" -not -path "*/node_modules/*" -exec stat -f "%Sm %N" {} \; 2>/dev/null | sort
```

### Step 3 — If anything suspicious
1. Keep machine offline.
2. Snapshot the disk (Time Machine, `dd`, or `tar` of home dir to external drive) for forensic preservation.
3. Rotate all credentials from a CLEAN device.
4. Wipe + reinstall macOS.

---

## 10. Glossary

| Term | Definition |
|---|---|
| **C2 (Command and Control)** | The attacker's server that malware calls home to. This worm hides its C2 on public blockchains. |
| **Exfiltration** | Sending stolen data out of a victim machine to an attacker server. |
| **IOC (Indicator of Compromise)** | Any fingerprint that proves an attack happened — file, IP, hash, wallet address, etc. |
| **XOR cipher** | A simple symmetric encryption: each byte of the message is mixed (via XOR) with a byte of the secret key. Trivial to break if the key is known. |
| **Shuffle decoder** | A custom string-decoder that reorders characters based on math. Used by the worm to hide its real code through multiple stacked layers. |
| **`eval()`** | A JavaScript function that takes a string and runs it as code. Dangerous because data becomes executable. The worm's primary mechanism. |
| **`tailwind.config.js`** | A JavaScript config file for the Tailwind CSS framework. Most modern web projects have one. The worm hides inside because IDEs auto-load configs. |
| **Tailwind IntelliSense** | A popular VS Code extension that provides autocomplete for Tailwind class names. To work, it has to run the config file. This is the trigger that fires the worm. |
| **EtherHiding** | The technique of storing malware command-and-control on a public blockchain rather than an attacker-controlled server. Makes takedown nearly impossible. |
| **Force-push** | A Git operation that overwrites the remote branch with the local — even discarding commits. The worm uses force-push to silently swap clean commits with poisoned ones. |
| **Signed commit** | A Git commit cryptographically signed by the author's private key. GitHub shows a "Verified" badge. Cannot be forged without the key. The structural fix. |
| **Secure Enclave** | A separate cryptographic chip in Apple Silicon Macs. Keys created here NEVER leave the chip — even malware running as root can't extract them. Touch ID gates their use. |

---

## 11. Technical appendix — reproducing the static decode

Anyone with Python 3 can independently verify everything in this report. **No JavaScript is executed at any point** — the worm's decoders are re-implemented as pure Python math.

### Decoder 1 — Outer Stage A shuffle (`sfL`)

```python
def sfL(w):
    n = 2667686
    b = list(w); L = len(b)
    for o in range(L):
        q = n * (o + 228) + (n % 50332)
        e = n * (o + 128) + (n % 52119)
        u, v = q % L, e % L
        b[u], b[v] = b[v], b[u]
        n = (q + e) % 4289487
    return ''.join(b)
```

### Decoder 2 — Stage A text-substitution

```python
def stageA(*args):
    m, s, u = 33, 93, 96
    alpha = "abcdefghijklmnopqrstuvwxyz"
    p = [82,60,80,88,76,72,81,85,75,90,89,79,65,94,71,70,66,74,87,86]
    n = {code: r + 1 for r, code in enumerate(p)}
    x_out = []
    for arg in args:
        k = arg.split(" ")
        for f in range(len(k) - 1, -1, -1):
            y = k[f]; z = None; i = 0; q = 0
            while q < len(y):
                j = ord(y[q]); h = n.get(j)
                if h:
                    a = (h - 1) * s + ord(y[q+1]) - m
                    w = q; q += 1
                elif j == u:
                    a = s * (len(p) - m + ord(y[q+1])) + ord(y[q+2]) - m
                    w = q; q += 2
                else:
                    q += 1; continue
                if z is None: z = []
                if w > i: z.append(y[i:w])
                if 0 <= a + 1 < len(k): z.append(k[a+1])
                i = q + 1; q += 1
            if z is not None:
                if i < len(y): z.append(y[i:])
                k[f] = ''.join(z)
        x_out.append(k[0])
    e = ''.join(x_out)
    v_arr = [92, 96, 32, 10, 42, 39] + p
    for r, code in enumerate(v_arr):
        e = e.replace('.' + alpha[r], chr(code))
    return e.replace('.!', '.')
```

### Decoder 3 — `ccfc` (the IOC array)

```python
def ccfc(f, v):
    DEL = chr(127); b = list(f); L = len(b)
    for g in range(L):
        a = v * (g + 139) + (v % 20044)
        w = v * (g + 473) + (v % 41543)
        b[a % L], b[w % L] = b[w % L], b[a % L]
        v = (a + w) % 5446973
    j = ''.join(b)
    return j.replace('%', DEL).replace('#1', '%').replace('#0', '#').split(DEL)
```

### Decoder 4 — XOR decryption of BSC payload

```python
key = "2[gWfGj;<:-93Z^C"
data = bytes.fromhex(BSC_INPUT.lstrip('0x')).decode('utf-8', errors='replace')
payload = data.split('?.?')[1]
plaintext = ''.join(chr(ord(payload[i]) ^ ord(key[i % len(key)])) for i in range(len(payload)))
```

### Decoder 5 — Environment-routing IOC array (`_$_bfdd`)

```python
def decoder_bfdd(m, s):
    DEL = chr(127); z = list(m); c = len(z)
    for b in range(c):
        v = s * (b + 242) + (s % 15897)
        u = s * (b + 488) + (s % 17788)
        z[v % c], z[u % c] = z[u % c], z[v % c]
        s = (v + u) % 3232083
    j = ''.join(z)
    return j.replace('%', DEL).replace('#1', '%').replace('#0', '#').split(DEL)
```

Running these five decoders end-to-end against the obfuscated worm payload extracted from any infected `tailwind.config.js` produces the IOCs listed in Section 6, with zero JavaScript execution.

---

## Closing note

This investigation was performed entirely with **static analysis**. No JavaScript was ever evaluated, no infected file was ever opened in an IDE with the Tailwind extension active, no `curl` was ever pointed at the worm's blockchain endpoints from a production machine. Every IP, every wallet, every transaction hash above was extracted by peeling back the worm's nested obfuscation one math step at a time.

**The single highest-impact action any organization can take is enabling "Require signed commits" on every protected branch.** That one setting takes about 60 seconds per repo, and it structurally ends the worm's ability to propagate via force-pushed amended commits — even if a developer machine remains infected.

If you found this useful, the most valuable next steps are:
1. Deploy the IP/host blocklist in Section 8
2. Submit the IOCs to VirusTotal / AbuseIPDB / your local CERT
3. Enforce signed commits in your own org
4. Forward this document to other security teams in your industry

---

*This document is published in the spirit of community defense. All technical IOCs are durable, durable IOCs derived from worm source via static analysis. Operational details that could identify specific affected organizations or individuals have been deliberately omitted.*
