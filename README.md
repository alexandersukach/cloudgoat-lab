# cloudgoat-lab

Resources and setup notes for practicing AWS cloud pentesting with
[CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat) in a Kali Linux VM.

This repo is the companion to my CloudGoat write-ups on Medium. The articles
walk through each scenario with screenshots and full explanations; this repo
holds the reusable bits — the **environment setup** and **command cheatsheets** —
so readers can stand up the same lab and follow along.

## What's here

### `docs/` — setup & reference (the full walkthroughs)
- **[CloudGoat-Docker-Setup.md](docs/CloudGoat-Docker-Setup.md)** — installing and
  running CloudGoat via the official Docker image, the state-in-container gotcha,
  scenario cost tiers, and DNS troubleshooting.
- **[Kali-VMware-Setup.md](docs/Kali-VMware-Setup.md)** — building the Kali Linux
  ARM64 VM on VMware Fusion (Apple Silicon), specs, and networking.
- **[Kali-AWS-PenTest-Reference.md](docs/Kali-AWS-PenTest-Reference.md)** — AWS CLI
  v2, credential profiles, and the attacker toolkit (Pacu, ScoutSuite, etc.).

### `cheatsheets/` — just the commands
Short, copy-paste command references for each scenario — no prose. Use these once
you've read the walkthroughs (or the Medium articles) and just need the commands.
- **[iam_enum_basics.md](cheatsheets/iam_enum_basics.md)** — IAM enumeration with the
  AWS CLI (managed/inline policies, groups, roles, policy versions).

## ⚠️ Safety & cost

CloudGoat deploys **real, billable, intentionally-vulnerable AWS resources.**

- Use a **personal/lab AWS account** — never work or production.
- Restrict deployed resources to your own IP (`cloudgoat config whitelist --auto`).
- **Always `cloudgoat destroy`** a scenario when you're done so it stops costing money.

See [docs/CloudGoat-Docker-Setup.md](docs/CloudGoat-Docker-Setup.md) for the full
safety notes.

## Disclaimer

For educational use and authorized security testing only. Everything here targets
infrastructure **you deploy in your own account** for learning.
