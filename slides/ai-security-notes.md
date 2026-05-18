# AI Agent Security
### Guardrails, Sandboxes, and Keeping Your Agents on a Leash

*Drupal AI Learners Club*

[Event: lu.ma/wmlu8k9y](https://lu.ma/wmlu8k9y)

---

## The Problem

Prompts and skills can *guide* agent behavior.

When an agent decides to go rogue — **soft guardrails won't save you.**

You need hard, OS-level controls.

---

## Key Security Surfaces

* **Filesystem** — what can it read, write, or execute?
* **Network** — where can it reach out to?
* **Process / Syscall** — what can spawned processes do?
* **Container escape** — can it break out of isolation?
* **Credentials** — what secrets are visible inside the environment?
* **Prompt Injection** — malicious content hijacking instructions
* **Supply Chain** — auto-pulled tools or agent definitions from external sources

---

## Filesystem Security

* Non-root user inside the container
* Write isolation — only the working directory
* Read-only mounts for host credentials (SSH keys, `.env`)
* Don't mount `~/.ssh` or cloud credential files into the container

**Claude Code:** `/sandbox` enables OS-level filesystem isolation

---

## Network Security

**Risks**
* Exfiltrate files, keys, env vars to attacker endpoints
* Pull and execute malicious code at runtime
* Domain fronting — reach blocked hosts via an allowed domain
* IPv6 bypass of IPv4-only rules

**Controls**
* Outbound allowlist — iptables + ipset + dnsmasq
* DNS interception to prevent IP-level bypass
* `sandbox.network.allowedDomains` in Claude Code settings

---

## Both Are Required

> "Without network isolation, a compromised agent could exfiltrate SSH keys.
> Without filesystem isolation, an agent can backdoor resources to gain network access."
>
> — Anthropic

**Note:** The Claude Code built-in proxy does not inspect TLS.
Allowing `github.com` permits domain fronting to arbitrary hosts.

---

## Claude Code Add-ons for DDEV

| Add-on | Approach | Stars |
|--------|----------|-------|
| [FreelyGive/ddev-claude-code](https://github.com/FreelyGive/ddev-claude-code) | In web container | 18 |
| [e0ipso/ddev-assistant-claude](https://github.com/e0ipso/ddev-assistant-claude) | In web container, CI+tests | 6 |
| [makraz/ddev-claude](https://github.com/makraz/ddev-claude) | Sidecar + iptables firewall | 1 |
| [trebormc/ddev-claude-code](https://github.com/trebormc/ddev-claude-code) | Sidecar + SSH, Drupal-specific | 0 |

`makraz/ddev-claude` mirrors Anthropic's own devcontainer firewall approach.

---

## AI Add-ons for DDEV

| Add-on | What It Does |
|--------|--------------|
| [trebormc/ddev-ai-workspace](https://github.com/trebormc/ddev-ai-workspace) | Installs full AI stack (Claude, OpenCode, Ralph, Playwright) ★20 |
| [trebormc/ddev-ai-ssh](https://github.com/trebormc/ddev-ai-ssh) | SSH into web container — no Docker socket needed |
| [tyler36/ddev-ollama](https://github.com/tyler36/ddev-ollama) | Run local LLMs as a DDEV sidecar |
| [e0ipso/ddev-playwright-cli](https://github.com/e0ipso/ddev-playwright-cli) | Playwright for AI browser automation |

⚠️ `ddev-agents-sync` auto-pulls agent definitions from git on every start — supply-chain risk.

---

## Local LLMs with ddev-ollama

```bash
ddev add-on get tyler36/ddev-ollama && ddev restart
ddev ollama pull llama3.2:3b
```

**Use cases**
* VS Code + [Continue](https://www.continue.dev/) — local Copilot-style autocomplete, no API key, no data leaving the machine
* Drupal AI module testing against `http://ollama:11434/v1`
* Offline / air-gapped development
* Cost control — local model for cheap tasks, Claude for complex ones

Ollama 0.24.0 has a built-in TUI with a **Launch Claude Code** option.

---

## Claude Code on a Remote VM

Run Claude Code on a remote server; interact via VS Code in the browser.

```bash
# On the remote VM / Coder workspace:
claude --remote-control
```

**Security benefits**
* Agent never runs on your laptop
* Blast radius contained to the VM
* Network egress lockable at VPC level
* Secrets scoped to the workspace; VM is snapshot/restorable

This presentation was built exactly this way — Claude Code in a [Coder](https://coder.com) workspace, VS Code in the browser.

[code.claude.com/docs/en/devcontainer](https://code.claude.com/docs/en/devcontainer)

---

## Claude Code Sandboxing

[code.claude.com/docs/en/sandboxing](https://code.claude.com/docs/en/sandboxing)

Enable with `/sandbox` or `settings.json`:
```json
{ "sandbox": { "enabled": true } }
```

* Filesystem — read/write only inside working directory
* Network — outbound blocked except your `allowedDomains` list
* Enforced at OS level (bubblewrap / Apple Seatbelt)
* Applies to every tool the agent spawns — not just Claude's own file tools
* Reduces permission prompts by ~84%

Open-source runtime for sandboxing anything (e.g. MCP servers):
```bash
npx @anthropic-ai/sandbox-runtime <command>
```
