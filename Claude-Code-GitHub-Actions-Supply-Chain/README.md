# Claude Code GitHub Actions Supply Chain Vulnerability
**Date Analyzed:** June 3, 2026  
**Source:** [Cyber Security News – June 2, 2026](https://cybersecuritynews.com/claude-codes-github-actions-vulnerability/)  
**Researcher Credit:** RyotaK, GMO Flatt Security  
**Patched In:** Claude Code GitHub Actions v1.0.94  
**CVSS v4.0 Score:** 7.8 (High)

---

## Overview

This writeup analyzes a critical supply chain vulnerability discovered in Anthropic's Claude Code GitHub Actions workflow. The flaw allowed a fully unauthenticated external attacker to compromise any repository, including Anthropic's own, by chaining a permission bypass with prompt injection and token theft. No special access was required to initiate the attack.

This is a real-world example of how a single misconfigured trust assumption can collapse an entire security boundary.

---

## Root Cause

The `checkWritePermissions` function in Claude Code GitHub Actions unconditionally trusted any actor whose username ended in `[bot]`, regardless of whether that actor had actual write permissions to the repository.

Since GitHub Apps have implicit read access to public repositories and can open issues or pull requests using only an installation token, this check was trivially bypassable by anyone who could create a GitHub App.

---

## Attack Chain (7 Steps)

### Step 1 - Create a Malicious GitHub App
The attacker creates a GitHub App under their own account. No special permissions are required, just a basic installation token.

**MITRE ATT&CK:** [T1585.003 - Establish Accounts: Cloud Accounts](https://attack.mitre.org/techniques/T1585/003/)

---

### Step 2 - Install the App on an Attacker-Controlled Repo
The app is installed on any repository the attacker owns. This generates a valid installation token that GitHub recognizes as a bot account, an automated account rather than a human user.

**MITRE ATT&CK:** [T1078 - Valid Accounts](https://attack.mitre.org/techniques/T1078/)

---

### Step 3 - Open an Issue or PR on the Target Repository
Using the installation token, the attacker opens an issue or pull request on the target repo. Because the account is recognized as a bot account (automated rather than human) the `checkWritePermissions` function returns `true`, granting full workflow access without any legitimate permissions.

**MITRE ATT&CK:** [T1190 - Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/)

---

### Step 4 - Inject a Malicious Prompt
The attacker crafts a fake error message inside the issue or PR description. Claude Code reads this content as part of its workflow and interprets the embedded instructions as legitimate commands, a classic prompt injection attack.

**MITRE ATT&CK:** [T1059 - Command and Scripting Interpreter](https://attack.mitre.org/techniques/T1059/)

> **Key Insight:** No exploit code was needed. The attack vector was natural language embedded in a GitHub issue.

---

### Step 5 - Steal Credentials from the Workflow Environment
The injected prompt instructed Claude to read `/proc/self/environ` — a Linux virtual file that exposes all environment variables for the current running process. In a GitHub Actions workflow, that includes any secrets passed into the environment, such as `GITHUB_TOKEN`. Because certain Bash commands were permitted without user approval, no additional bypass was needed. Claude was already allowed to read its own process information.

**MITRE ATT&CK:** [T1552.001 - Unsecured Credentials: Credentials in Files](https://attack.mitre.org/techniques/T1552/001/)

---

### Step 6 - Escalate to Privileged Repository Access
The attacker used those stolen credentials to impersonate a trusted identity and obtain write access to the repository. The stolen token was then written into a public GitHub issue where the attacker could simply read it.

**MITRE ATT&CK:** [T1528 - Steal Application Access Token](https://attack.mitre.org/techniques/T1528/)

---

### Step 7 - Push Malicious Code / Supply Chain Compromise
With write access to `anthropics/claude-code-action`, the attacker injects backdoored code directly into the action's source. Because downstream repositories depend on this action, the malicious code propagates automatically, a textbook supply chain attack.

**MITRE ATT&CK:** [T1195.002 - Supply Chain Compromise: Compromise Software Supply Chain](https://attack.mitre.org/techniques/T1195/002/)

---

## Attack Chain Summary

1. Create malicious GitHub App - T1585.003: Establish Accounts: Cloud Accounts (GitHub App)
2. Install on attacker-controlled repo - T1078: Valid Accounts (Bot Installation Token)
3. Open issue/PR to bypass permission check - T1190:  Exploit Public Facing Application (Permission Check Bypass)
4. Inject malicious prompt via issue content - T1059: Command and Scripting Interpreter (Prompt Injection)
5. Read workflow environment for secrets - T1552.001: Unsecured Credentials: Credentials in Files (Environment Variables)
6. Exchange tokens for privileged GitHub access - T1528: Steac Application Access Token (Token Exchange)
7. Push backdoored code to Anthropic's repo - T1195.002: Supply Chain Compromise Software Supply Chain (Backdoor Action)

Note: MITRE ATT&CK technique IDs were researched and verified during analysis.

---

## Possible Preventive Controls

Analyzing this from a risk and compliance perspective, what stands out is that most of these controls aren't exotic - they map to foundational security principles that already exist in frameworks like NIST CSF. The failure here wasn't a lack of available controls. It was a lack of applying them consistently.

### Least Privilege
The core issue in this attack is that permissions were broader than they needed to be. The `[bot]` actor was trusted unconditionally and it was given more access than its role required. A least privilege approach asks: *what is the minimum access this account actually needs to do its job?* Applying that question to automated workflows and bot accounts would have limited the blast radius significantly, even if the bypass still occurred.

### Permission Misconfiguration as a Risk
The wildcard `allowed_non_write_users: "*"` configuration in Anthropic's own example workflows is a misconfiguration risk, the kind of thing that often gets introduced during setup and never revisited. This is exactly why configuration reviews and periodic access audits matter. In a GRC context, this falls under ongoing monitoring and control validation, not just initial deployment.

### Third-Party / Vendor Risk
The downstream repositories that would have been compromised did nothing wrong. They trusted a vendor (Anthropic) and that trust became their exposure. Vendor risk management asks three questions: *What access does this vendor have? What happens to us if they are compromised? And how do we verify their controls over time?* This attack is a real-world example of why those questions matter before adopting any third-party tool.

### AI as a New Attack Surface in Risk Assessments
What this case makes clear is that when AI agents are embedded in automated workflows, the risk profile changes in ways traditional controls don't account for. Natural language becomes an input that can be weaponized. Any organization adopting AI tooling in their development pipelines should be explicitly assessing prompt injection as a threat vector, not just code vulnerabilities.

---

## Personal Takeaways

What struck me most about this chain is how **individually unremarkable each step looks**. Creating a GitHub App, opening an issue, reading a file, these are all normal developer activities. The danger only becomes visible when you trace the full chain.

This reinforced for me:

1. **One unconditional `return true` in a permission check cascaded into a potential full supply chain compromise.** The scope of the failure was invisible at the function level and only apparent at the system level.
2. **Prompt injection is an emerging attack class that defenders aren't fully prepared for yet.** As AI gets embedded deeper into enterprise tooling, this attack surface will grow.
3. **Supply chain risk is third-party risk.** The downstream repos that would have been compromised did nothing wrong — they simply trusted a vendor. This maps directly to GRC vendor risk management frameworks.

---

*Analyzed by Beth Watt | Cybersecurity Portfolio*  
*Connect on [LinkedIn](https://www.linkedin.com/in/elizabethwatt-)*
