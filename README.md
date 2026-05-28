# Shai-Hulud worm research

Static analysis of the **Shai-Hulud npm / JavaScript supply-chain worm campaign** — including IOCs, kill chain, attacker IPs, blockchain wallet addresses, and Python decoders that reproduce every step.

## 📄 Read the full report

**→ [REPORT.md](REPORT.md)** — full write-up (also mirrored to the [Wiki](../../wiki) once initialized)

## What's in the report

- **Plain-English explanation** of the attack (for non-technical readers)
- **Full kill chain** with diagrams and stage-by-stage breakdown
- **All Indicators of Compromise**: 3 attacker IPs, 4 blockchain APIs, 7 wallet addresses, XOR keys, file signatures, runtime globals
- **Live URLs** to verify every IOC on Tronscan / BSCScan / VirusTotal / AbuseIPDB / Shodan
- **Defensive actions** prioritized P1 / P2 / P3 — firewall rules, GitHub branch-protection settings, credential rotation
- **Forensic procedure** for checking past infection on a developer machine
- **Python decoders** (5 of them) that reproduce every IOC from the raw worm bytes — fully static, no JavaScript execution

## Public-interest disclosure

This research is published in the spirit of community defense. All IOCs are durable indicators derived from the worm's source via static analysis — no JavaScript was evaluated at any point. Operational details that could identify specific affected organizations or individuals have been deliberately omitted.

If this analysis is useful to your security team, please:
1. Deploy the IP/host blocklist from Section 8
2. Submit the IOCs to VirusTotal / AbuseIPDB / your local CERT
3. Enforce "Require signed commits" on every protected branch in your org
4. Forward this report to other security teams in your industry

## License

Released into the public domain (CC0). Cite, copy, repost, translate, fork freely.
