# AI Agent Security
### Guardrails, Sandboxes, and Keeping Your Agents on a Leash

*Drupal AI Learners Club*

[Event](https://lu.ma/wmlu8k9y) | [Slides](https://rfay.github.io/ai-security-notes/) | [Repo](https://github.com/rfay/ai-security-notes)

---

## The Problem

Prompts and skills can *guide* agent behavior.

If an agent were to go rogue — **soft guardrails won't save you.**

Maybe even if you're approving each action you'll lose control.

You need hard, OS-level controls.

---

## Key Security Surfaces

* **Filesystem** — what can it read, write, or execute?
* **Network** — where can it reach out to?
* **Process / Syscall** — what can spawned processes do?
* **Container escape** — can it break out of isolation?
* **Credentials** — what secrets are visible inside the environment?
* **Prompt Injection** — malicious content hijacking instructions
* **Supply Chain** — auto-pulled tools or agent definitions

---

## Filesystem Security

* With Docker, use a non-root user inside the container
* Write isolation — only allow the working directory
* Read-only mounts for host credentials
* Never mount `~/.ssh` or cloud credentials into the container
* Evil/incorrect generated code remains a problem even with filesystem security

---

## Network Security

**Risks:** file exfiltration, malicious code pull, domain fronting, IPv6 bypass, local network exploration

**Controls**
* Outbound allowlist — iptables + ipset + dnsmasq
* DNS interception to prevent IP-level bypass
* `sandbox.network.allowedDomains` in Claude Code settings

---

## Both Filesystem and Network Security Are Required

> "Without network isolation, a compromised agent could exfiltrate SSH keys.
> Without filesystem isolation, an agent can backdoor resources to gain network access."
>
> — Anthropic

---

## Claude Code Add-ons for DDEV

| Add-on | Notes |
|--------|-------|
| [FreelyGive/ddev-claude-code](https://github.com/FreelyGive/ddev-claude-code) | Most-used ★18 |
| [e0ipso/ddev-assistant-claude](https://github.com/e0ipso/ddev-assistant-claude) | Best-engineered, has CI ★6 |
| [makraz/ddev-claude](https://github.com/makraz/ddev-claude) | Sidecar + iptables firewall |

`makraz/ddev-claude` mirrors Anthropic's own devcontainer firewall approach.

---

## AI Add-ons for DDEV

| Add-on | Notes |
|--------|-------|
| [trebormc/ddev-ai-workspace](https://github.com/trebormc/ddev-ai-workspace) | Full AI stack meta add-on ★20 |
| [tyler36/ddev-ollama](https://github.com/tyler36/ddev-ollama) | Local LLMs as a DDEV sidecar |
| [e0ipso/ddev-playwright-cli](https://github.com/e0ipso/ddev-playwright-cli) | Playwright for AI browser automation |

⚠️ `ddev-ai-workspace` installs `ddev-agents-sync` as a dependency — it auto-pulls agent definitions from external git repos on every `ddev start`. If a repo is compromised, malicious agent instructions execute automatically. **Supply-chain risk.**

---

## Local LLMs with ddev-ollama

```bash
ddev add-on get tyler36/ddev-ollama && ddev restart
ddev ollama pull llama3.2:3b
```

* VS Code + [Continue](https://www.continue.dev/) — local autocomplete, no API key, no data leaving the machine
* Drupal AI module testing against `http://ollama:11434/v1`
* Offline / air-gapped development
* Ollama TUI includes a **Launch Claude Code** option

---

## Claude Code on a Remote VM

```bash
claude --remote-control   # on the remote VM / Coder workspace
```

* Agent never runs on your laptop
* Blast radius contained to the VM
* Network egress lockable at VPC level
* Secrets scoped to the workspace

*This presentation was built this way — Claude Code in a [Coder](https://coder.com) workspace, VS Code in the browser.*

---

## Claude Code Sandboxing

[code.claude.com/docs/en/sandboxing](https://code.claude.com/docs/en/sandboxing)

```json
{ "sandbox": { "enabled": true } }
```

* Filesystem — read/write only inside working directory
* Network — outbound blocked except `allowedDomains`
* OS-level enforcement (bubblewrap / Apple Seatbelt)
* Applies to every tool the agent spawns
* ~84% reduction in permission prompts

```bash
npx @anthropic-ai/sandbox-runtime <command>   # sandbox anything
```
