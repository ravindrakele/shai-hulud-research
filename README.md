# Shai-Hulud worm research

Static analysis of the Shai-Hulud npm / JavaScript supply-chain worm campaign — including IOCs, kill chain, attacker IPs, blockchain wallet addresses, and Python decoders that reproduce every step.

**Full write-up → [see the Wiki](../../wiki).**

This repository's wiki contains:
- Plain-English explanation of the attack
- Full kill chain with diagrams
- All Indicators of Compromise (IPs, blockchain wallets, XOR keys)
- Live URLs to verify on Tronscan / BSCScan / VirusTotal
- Defensive actions (firewall rules, GitHub branch protection)
- Python decoders to reproduce every IOC

## Public-interest disclosure

This research is published in the spirit of community defense. All IOCs are durable indicators derived from the worm's source via static analysis (no JavaScript execution). Operational details that could identify specific affected organizations or individuals have been deliberately omitted.
