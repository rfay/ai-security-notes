# AI Agent Security
### Guardrails, Sandboxes, and Keeping Your Agents on a Leash

*Drupal AI Learners Club*

---

## Key Security Surfaces

When an AI agent can run code, five surfaces need to be controlled:

* **Filesystem** — what can it read, write, or execute?
* **Network** — where can it reach out to?
* **Process / Syscall** — what can spawned processes do?
* **Container / Host Escape** — can it break out of isolation?
* **Credentials** — what secrets are visible inside the environment?

Plus two softer surfaces:

* **Prompt Injection** — malicious content hijacking the agent's instructions
* **Supply Chain** — auto-pulled agent definitions or tools from external sources

---

## Filesystem Security

The agent should only touch what it needs to.

**Risks**
* Writing to `$PATH` directories or shell configs (`.bashrc`, `.zshrc`)
* Reading SSH keys, `.env` files, cloud credentials
* Modifying parent directories outside the working tree

**Controls**
* OS-level write isolation — [bubblewrap](https://github.com/containers/bubblewrap) (Linux), Apple Seatbelt (macOS)
* Explicit `allowWrite` / `denyRead` lists
* Read-only mounts for host credentials
* Non-root user inside the container

**Anthropic's built-in sandbox**

```bash
# Enable in Claude Code settings.json
{ "sandbox": { "enabled": true } }
# Or toggle interactively
/sandbox
```

Reduces permission prompts by ~84% in Anthropic's internal usage.
Docs: [code.claude.com/docs/en/sandboxing](https://code.claude.com/docs/en/sandboxing)

---

## Network Security

Without network isolation, an agent can exfiltrate anything it can read.

**Risks**
* Data exfiltration — files, keys, env vars sent to attacker-controlled endpoints
* Pulling and executing malicious code at runtime
* Domain fronting — using an allowed domain (e.g. `github.com`) to reach disallowed hosts
* IPv6 bypass of IPv4-only rules

**Controls**
* Outbound allowlist via iptables + ipset + dnsmasq
* DNS interception to prevent IP-level bypass
* Explicit block of IPv6 (separate rule required)
* `sandbox.network.allowedDomains` in Claude Code settings

**Anthropic's warning:**
> "Effective sandboxing requires **both** filesystem and network isolation.
> Without network isolation, a compromised agent could exfiltrate SSH keys.
> Without filesystem isolation, an agent can backdoor resources to gain network access."

**Note:** The built-in proxy does not inspect TLS — allowing broad domains like `github.com` permits domain fronting.

---

## Claude Code Add-ons for DDEV

Community add-ons that integrate Claude Code into DDEV projects:

| Add-on | Approach | Stars | Verdict |
|--------|----------|-------|---------|
| [FreelyGive/ddev-claude-code](https://github.com/FreelyGive/ddev-claude-code) | In web container | 18 | Most-used |
| [e0ipso/ddev-assistant-claude](https://github.com/e0ipso/ddev-assistant-claude) | In web container | 6 | Best-engineered (CI + tests) |
| [makraz/ddev-claude](https://github.com/makraz/ddev-claude) | Sidecar + iptables firewall | 1 | Most security-conscious |
| [trebormc/ddev-claude-code](https://github.com/trebormc/ddev-claude-code) | Sidecar + SSH | 0 | Drupal-specific ecosystem |

**Two philosophies:**
* **In-web-container** — simpler, smaller footprint, agent has full web container access
* **Dedicated sidecar** — more isolation, heavier, closer to DDEV's multi-service model

**`makraz/ddev-claude`** mirrors Anthropic's own devcontainer reference implementation:
iptables outbound allowlist + `NET_ADMIN`/`NET_RAW` caps + non-root user.

---

## AI Add-ons for DDEV (Broader Ecosystem)

Beyond Claude Code — the wider AI tooling landscape in DDEV:

| Add-on | What It Does | Verdict |
|--------|--------------|---------|
| [trebormc/ddev-ai-workspace](https://github.com/trebormc/ddev-ai-workspace) | Meta add-on: installs full AI stack (Claude, OpenCode, Ralph, Playwright MCP) | 20 ★ — Experimental |
| [trebormc/ddev-ai-ssh](https://github.com/trebormc/ddev-ai-ssh) | SSH into web container for AI agents — no Docker socket needed | Experimental |
| [tyler36/ddev-ollama](https://github.com/tyler36/ddev-ollama) | Run local LLMs (Llama, Mistral, etc.) via Ollama sidecar | Experimental |
| [e0ipso/ddev-playwright-cli](https://github.com/e0ipso/ddev-playwright-cli) | `@playwright/cli` for AI browser automation + Claude Code skills | Experimental |
| [trebormc/ddev-opencode](https://github.com/trebormc/ddev-opencode) | OpenCode AI TUI as DDEV sidecar | Early |

**Supply-chain note:** `trebormc/ddev-agents-sync` auto-pulls agent definitions from git on every `ddev start` — a concrete example of the supply-chain risk surface.

---

## Running Local LLMs with ddev-ollama

[tyler36/ddev-ollama](https://github.com/tyler36/ddev-ollama) runs Ollama as a DDEV sidecar.

**Install**
```bash
ddev add-on get tyler36/ddev-ollama
ddev restart
ddev ollama pull llama3.2:3b
```

**Use cases**

* **VS Code + [Continue](https://www.continue.dev/) extension** — local Copilot-style autocomplete, no API key, no data leaving the machine
* **Drupal AI module testing** — point the module at `http://ollama:11434/v1` (OpenAI-compatible) from inside the web container
* **Offline / air-gapped development** — no internet dependency once the model is pulled
* **Cost control** — use a local model for cheap/fast tasks, Claude API for complex reasoning

**Expose port to host (for VS Code)**
```yaml
# .ddev/docker-compose.ollama-port.yaml
services:
  ollama:
    ports:
      - "127.0.0.1:11434:11434"
```

Ollama 0.24.0 includes a TUI (`ddev ollama`) that offers built-in Claude Code and Hermes Agent launchers.

---

## Claude Code on a Remote VM

Claude Code supports a `--remote-control` mode — run the agent headlessly on a remote server while you interact via a local client.

**How this presentation was built**

```bash
# Claude Code running in a Coder workspace (remote VM)
# VS Code in the browser connects to the same workspace
# claude --remote-control bridges the two
```

* The agent runs on the remote host with access to the project files
* VS Code (browser or desktop) connects via [Coder](https://coder.com)
* No local install of Claude Code needed on the developer's machine

**Why this matters for security**

* The agent never runs on your laptop — blast radius is contained to the VM
* VM can be snapshot/restored; secrets are scoped to the workspace
* Network egress can be locked down at the VM/VPC level, independent of the agent
* Pairs naturally with Anthropic's devcontainer approach:
  [code.claude.com/docs/en/devcontainer](https://code.claude.com/docs/en/devcontainer)

**Anthropic's recommended approach for `--dangerously-skip-permissions`:**
> "Only use this mode in isolated environments like containers, VMs, or dev containers
> without internet access, where Claude Code cannot damage your host system."
