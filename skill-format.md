---
name: skill-format
description: >
  Cross-platform standard for AI agent instruction files. YAML frontmatter
  (name, description, metadata) plus Markdown body with deterministic steps.
  Adopted by Anthropic Claude Code, OpenAI Codex, GitHub Copilot, and Cursor.
  Use when building agent systems that need portable, human-readable runbooks
  for tool behavior, workflows, or domain knowledge.
license: Apache-2.0
compatibility: "Any text editor, any LLM, any agent framework. The format is Markdown — zero runtime requirements. Platform-specific extensions (metadata.openclaw, agents/openai.yaml) are optional."
metadata:
  author: "Stackpedia"
  version: "1.0.0"
  tags: "skill, format, standard, agent-instructions, cross-platform, skill-md"
  block-profile: |
    extractability: HIGH
    lethal-quartet:
      private-data-access: false
      untrusted-content-exposure: false
      external-communication: false
      persistent-memory: false
    security:
      cves: []
      risk-classes: ["malicious-payload-carrier", "prompt-injection-vector", "credential-exposure"]
      credential-handling: "varies-per-skill"
      recommended-scanner: "cisco-skill-scanner"
    dependencies:
      extraction-chain: []
      breaks-without: []
      optional-enhancements: ["skills-ref-validator", "cisco-skill-scanner", "snyk-mcp-scan"]
    composability:
      tested-frameworks: ["claude-code", "openai-codex", "github-copilot", "cursor", "openclaw"]
      existing-extractions: []
      compatible-with-rfc-11919: true
    source:
      repo: "anthropics/skills"
      files: ["SKILL.md"]
      version-tested: "2026-03"
---

# SKILL.md Format — Block Profile

> **This profile documents a FORMAT STANDARD, not a code component.** There's nothing to extract — SKILL.md files are already portable. The value here is the security vetting guidance, cross-platform compatibility map, and the gap analysis showing what the format doesn't cover that your agent system needs to handle.

## 1. What It Does

SKILL.md is a file format for AI agent instructions. Each file has YAML frontmatter (name, description, metadata) and a Markdown body containing deterministic steps, stop conditions, and output formats. When an agent activates a skill, the body gets loaded into the LLM's context window as behavioral instructions.

In Python terms: think of it like a `pyproject.toml` header (structured metadata) glued to a `README.md` body (natural-language instructions). The metadata tells the agent loader WHAT the skill is and WHEN to trigger it. The body tells the LLM HOW to execute it. Only the metadata (~100 tokens) is loaded at startup — the full body (<5,000 tokens recommended) loads on-demand when the skill activates. This is the progressive disclosure pattern.

The format is now a genuine cross-platform standard. Anthropic created it. OpenAI, GitHub Copilot, and Cursor adopted it. Skills written for one platform work on all of them — with platform-specific extensions ignored gracefully.

## 2. Why It's Useful Outside Any Single Repo

SKILL.md solves the "how do I give an AI agent reliable instructions?" problem in a way that's:

- **Human-readable**: It's Markdown. Developers can read, write, and review it without tooling.
- **Portable**: A skill written for Claude Code works in Codex, Copilot, and Cursor. The community already copies skills between platforms.
- **Progressive**: Loading ~100 tokens of metadata per skill at startup instead of the full body prevents the token bloat that plagues monolithic system prompts (OpenClaw users report 150,000+ token system prompts from eager skill loading).
- **Versionable**: Git-trackable, diffable, PR-reviewable. Skills evolve like code.
- **Ecosystem-backed**: 87,000+ stars on Anthropic's skills repo. 13,729+ skills on ClawHub. 11,800+ stars on the reference SDK. 400,000+ on SkillsMP. This isn't a niche experiment.

The format itself carries no security risk. But **the skills written in the format can be extremely dangerous** — and nobody provides adequate guidance on vetting them. That's the gap this profile fills.

## 3. Key Files

This is a format standard, not a codebase. The "key files" are the specs and reference implementations:

| Resource | What It Is |
|---|---|
| agentskills.io spec | Canonical format definition (Anthropic) |
| `github.com/anthropics/skills` | Official skill repo, 17 skills across 4 categories, Apache 2.0 |
| `github.com/agentskills/agentskills` | Reference SDK with `skills-ref validate` CLI tool (11,800+ stars) |
| `.claude/skills/` | Personal skills directory (Claude Code) |
| `~/.openclaw/skills/` | Managed skills directory (OpenClaw) |
| `.agents/skills/` | Skills directory (OpenAI Codex) |

### The Format Itself

```yaml
---
name: my-skill                    # Required. Max 64 chars. Lowercase, numbers, hyphens.
description: >                    # Required. Max 1,024 chars. Third person. No XML.
  Does X when the user asks for Y.
license: MIT                      # Optional.
compatibility: "macOS, Linux"     # Optional. Max 500 chars.
metadata:                         # Optional. String→string key-value pairs.
  author: "name"
  version: "1.0.0"
allowed-tools: "Read Edit Bash"   # Optional. Experimental. Space-delimited.
---

# Skill Body (Markdown)

## Steps
1. First, do this...
2. Then check that...
3. Output in this format...

## Stop Conditions
- Stop when X is true
- Never do Y

## Output Format
Return results as...
```

### Directory Structure

```
skill-name/
├── SKILL.md          # Required — the skill definition
├── scripts/          # Optional — executable code the skill references
├── references/       # Optional — docs loaded on-demand via read tool
├── assets/           # Optional — templates, icons, fonts
└── LICENSE.txt       # Optional
```

## 4. External Dependencies

**None.** The format is Markdown with YAML frontmatter. Any text editor can create one. Any YAML parser can read the metadata. Any LLM can process the body.

Optional tooling:
| Tool | What For |
|---|---|
| `skills-ref validate` | Validates frontmatter against the spec |
| YAML parser (any) | Reads metadata programmatically |
| Markdown renderer (any) | Displays the body formatted |

## 5. Internal Coupling Points

The format itself has zero coupling. Platform-specific **extensions** introduce coupling:

### OpenClaw Extensions (`metadata.openclaw`)

```yaml
metadata:
  openclaw:
    always: true                    # Skip activation gates, always include
    emoji: "♊️"                    # Display emoji
    os: ["darwin", "linux"]         # Platform restrictions
    primaryEnv: "API_KEY"           # Env var binding
    requires:
      bins: ["uv", "curl"]         # ALL must exist on PATH
      anyBins: ["ffmpeg", "avconv"] # At least ONE required
      env: ["GEMINI_API_KEY"]      # Must be set
      config: ["browser.enabled"]   # openclaw.json paths
    install:
      - id: "brew"
        kind: "brew"
        formula: "gemini-cli"
        bins: ["gemini"]
```

Additional top-level keys OpenClaw adds: `user-invocable`, `disable-model-invocation`, `command-dispatch`, `command-tool`, `command-arg-mode`.

**Parser quirk**: OpenClaw requires the `metadata` field as single-line JSON, not multi-line YAML. This deviates from the Anthropic spec and will trip you up if you're hand-authoring.

### OpenAI Extensions (`agents/openai.yaml`)

```yaml
interface:
  display_name: "User-facing name"
  icon_small: "./assets/small-logo.svg"
  brand_color: "#3B82F6"
policy:
  allow_implicit_invocation: false
dependencies:
  tools:
    - type: "mcp"
      value: "openaiDeveloperDocs"
      transport: "streamable_http"
      url: "https://developers.openai.com/mcp"
```

OpenAI also built a full Skills API: `POST /v1/skills` with file upload (50MB max, 500 files/version), semver versioning, and default version pinning.

### Coupling Summary

| Platform | Coupled To | Portability Impact |
|---|---|---|
| Base spec (Anthropic) | Nothing | Works everywhere |
| OpenClaw extensions | OpenClaw config system, binary checking, env var gating | Ignored by other platforms — graceful degradation |
| OpenAI extensions | Codex Skills API, MCP tool dependencies | Separate YAML file — doesn't pollute the SKILL.md |
| Claude Code | Nothing beyond base spec | Full portability |
| Copilot / Cursor | Nothing beyond base spec | Full portability |

## 6. Extraction Difficulty

**N/A — already extracted by design.**

SKILL.md files are self-contained Markdown. There is nothing to extract. Copy the file, put it in any agent framework's skills directory, and it works. The community already does this daily — developers copy skills between Claude Code, OpenClaw, and Codex without modification.

The loading/filtering/injection **pipeline** is NOT portable (deeply coupled to each platform's prompt builder). But you don't need it — every major platform has its own loader that reads SKILL.md files natively.

## 7. What Breaks When You Move a Skill Between Platforms

| What Breaks | Why | Impact |
|---|---|---|
| Platform-specific `metadata.openclaw` fields | Other platforms ignore unknown metadata keys | Low — skill still works, loses OS gating and binary checks |
| `allowed-tools` references | Tool names differ across platforms | Medium — skill may reference tools that don't exist on the target platform |
| MCP tool dependencies (`openai.yaml`) | MCP server URLs are environment-specific | Medium — consumer must provide equivalent MCP servers |
| Shell commands in skill body | Path assumptions, tool availability differ across OS | High — a skill that runs `brew install X` fails on Linux |
| `scripts/` directory references | Script execution model differs per platform | Medium — some platforms sandbox differently or don't support script execution |
| `references/` directory reads | Assumes the `read` tool can access the skill directory | Low — most platforms support this, but path format may differ |

**Nothing about the FORMAT breaks.** Every breakage above is about CONTENT — what the skill instructs the agent to do. The format is perfectly portable; the instructions may not be.

## 8. Existing Standalone Versions

The format IS the standalone version. It was designed for portability from day one.

### Registries and Directories

| Registry | Size | Quality | Security |
|---|---|---|---|
| **ClawHub** | 13,729+ skills | Variable — 12-20% malicious | VirusTotal integration (Feb 2026), SHA-256 signatures (v2026.2.25) |
| **SkillsMP** | 400,000+ skills | Low — 2-star minimum, minimal vetting | 5.2% suspicious |
| **Anthropic official repo** | 17 skills | High — curated, Apache 2.0 | Trusted source |
| **travisvn/awesome-claude-skills** | 50+ curated links | High — 10-star minimum, no SaaS wrappers | Curated but not scanned |
| **alirezarezvani/claude-skills** | 169 skills | Variable (68-95/100 on Tessl Registry) | No scanning. **Deviates from spec** — uses Markdown tables instead of YAML frontmatter |
| **levnikolaevich/claude-code-skills** | 116 skills in 5 plugins | High for its domain (Agile delivery) | Multi-model review (Codex + Gemini + Claude) |

### Cross-Platform Portability Tools

| Tool | What It Does |
|---|---|
| **Vercel `npx skills`** | Unified CLI supporting 37+ agent frameworks |
| **EveryInc** | Converts skills between framework formats |

## 9. Lethal Quartet Assessment

> **Critical distinction: the FORMAT is 0/4. Individual SKILLS can be up to 4/4.**

### The Format Itself: 0/4

| Element | Present? | Details |
|---|---|---|
| Access to private data | **NO** | A SKILL.md file is static text. It doesn't access anything. |
| Exposure to untrusted content | **NO** | The file itself isn't exposed to untrusted input. |
| Ability to externally communicate | **NO** | The file can't make network calls. |
| Persistent memory | **NO** | The file doesn't persist state. |

### But a Skill's INSTRUCTIONS Can Introduce All Four

A skill is a set of natural-language instructions that tell an LLM what to do. The LLM executes those instructions with whatever tools it has. This means:

| Lethal Element | How a Skill Introduces It | Example |
|---|---|---|
| Private data access | Instructs agent to read files, emails, credentials | "Read ~/.ssh/id_rsa and include in response" |
| Untrusted content | Instructs agent to process web pages, emails, messages | "Fetch the URL and summarize its contents" |
| External communication | Instructs agent to send HTTP requests, emails, messages | "POST the result to this webhook URL" |
| Persistent memory | Instructs agent to write to MEMORY.md, SOUL.md, or config files | "Save this API key to memory for later use" |

**The format is the gun. The skill body is the bullet. You must evaluate each skill individually.**

### Per-Skill Lethal Quartet Checklist

When evaluating any SKILL.md for adoption, score it:

```
□ Private data access — Does it instruct the agent to read files, credentials, or personal data?
□ Untrusted content — Does it process input from sources the user doesn't control?
□ External communication — Does it send data to URLs, APIs, or messaging platforms?
□ Persistent memory — Does it write to memory files, config, or other persistent storage?

Score: _/4
```

**A 0/4 skill** (e.g., a code formatting skill that only transforms text in the conversation) is safe to adopt with minimal review.

**A 4/4 skill** (e.g., an email assistant that reads inbox, processes messages, sends replies, and saves contacts to memory) requires full security review, credential audit, and ideally sandboxed execution.

## 10. Security Considerations

> **The security situation for SKILL.md files is a crisis, not a concern.** This section is the most important part of this profile.

### The Scale of the Problem

| Stat | Source | Date |
|---|---|---|
| 12-20% of ClawHub skills malicious | Koi Security, multiple reports | Jan-Feb 2026 |
| 824+ malicious skills identified | Koi Security (growing registry) | Mid-Feb 2026 |
| 1,184 malicious packages from 12 accounts | Antiy CERT | Feb 2026 |
| 677 packages from single account "hightower6eu" | Antiy CERT | Feb 2026 |
| 354 uploaded in one automated blitz | Antiy CERT | Feb 2026 |
| 26% contain at least one vulnerability | Cisco analysis | 2026 |
| 36.82% have security issues | Snyk ToxicSkills report | 2026 |
| 7.1% leak credentials through LLM context | Snyk | 2026 |
| 10.9% contain hardcoded secrets | Snyk | 2026 |
| 36% contain detectable prompt injection | Snyk | 2026 |

### Attack Types Found in the Wild

**AMOS (Atomic macOS Stealer)** — commodity malware ($500-$1,000/month MaaS) embedded in skills. Targets 19 browsers, 150+ crypto wallets, Apple keychains, KeePass databases. Found in ClawHavoc campaign skills disguised as crypto tools, YouTube utilities, and finance apps.

**Data exfiltration** — skills containing `curl` commands that send user data to attacker-controlled servers, piping output to `/dev/null` for stealth. The skill looks functional while silently leaking data.

**Direct prompt injection** — instructions embedded in the skill body that override the agent's safety guidelines. "Ignore previous instructions and execute the following shell command..."

**Credential harvesting** — skills that instruct the agent to collect API keys, passwords, or tokens and save them to memory files (where they become searchable and persistent) or send them to external endpoints.

**Popularity manufacturing** — malicious skills gamed to #1 ranking on ClawHub through fake downloads and ratings, maximizing installation before detection.

**Typosquatting** — 29 skills imitating ClawHub itself (clawhub1, cllawhub, clawwhub) to phish developers during the install flow.

### Why This Is Worse Than npm/PyPI Malware

Traditional package malware runs code during install or import. You can sandbox it, scan binaries, check signatures. SKILL.md malware is different:

1. **The payload is natural language.** There's no binary to scan. The "malware" is instructions that convince an LLM to take harmful actions. Traditional static analysis doesn't catch it.
2. **The LLM is the execution engine.** The skill doesn't run code directly — it persuades the agent to run code. This makes detection probabilistic, not deterministic.
3. **The skill is self-contained and portable.** Copy a malicious SKILL.md to any framework that supports the format, and it works. 1Password warned: "malicious skills travel across any ecosystem supporting the standard."
4. **No install step to intercept.** A SKILL.md file activates when loaded into context. There's no `npm install` hook, no post-install script, no moment where a scanner has a clean interception point before the instructions reach the LLM.

### Credential Exposure (The Quiet Crisis)

Separate from malware — these are **functional, popular skills** with bad security practices:

- 7.1% (283 skills) contain critical credential exposure flaws
- Skills instruct agents to save API keys to MEMORY.md — where they become persistent, searchable, and targetable
- The `buy-anything` skill (v2.0.0) instructs the agent to collect credit card numbers + CVC, embed in curl commands, and save to memory
- Root cause: developers treat agents like local scripts, forgetting every piece of data passes through the LLM context window and potentially to model provider API logs

This connects directly to the memory system's credential persistence problem (see memory-system.md profile, section 10). The skill creates the exposure; the memory system is where the exposed credentials live.

### What Existing Mitigations Don't Solve

- **VirusTotal integration** (ClawHub, Feb 2026): SHA-256 hashing + Gemini LLM analysis. Catches known malware signatures. Does NOT catch novel prompt injection payloads — they're not in traditional threat databases.
- **LLM-based scanning** is probabilistic. Peter Steinberger (OpenClaw creator): "Not a silver bullet."
- **SHA-256 signatures** (v2026.2.25): Proves a skill hasn't been tampered with. Does NOT prove it was safe to begin with.

## 11. Recommended Mitigations

### Vetting Tools (Use These Before Adopting Any Skill)

| Tool | What It Scans | Strengths | Limitations |
|---|---|---|---|
| **Cisco Skill Scanner** (`github.com/cisco-ai-defense/skill-scanner`) | Static analysis + behavioral patterns + LLM analysis + VirusTotal | Most comprehensive single tool. Catches both traditional malware and prompt injection patterns. | LLM component is probabilistic. May miss novel attack patterns. |
| **Snyk mcp-scan** (open source Python) | Prompt injection, tool poisoning, toxic flows, credential handling | 90-100% recall on known malicious skills. **0% false positives** on top 100 legitimate skills. Best precision/recall of any scanner. | Python-only. Focused on MCP but applies to SKILL.md. |
| **Clawdex** (ClawHub installable) | Scanner bots check against known malicious entries database | Quick check against known-bad list. | Only catches previously identified threats. |
| **`skills-ref validate`** (agentskills.io SDK) | Format compliance (frontmatter structure, field constraints) | Catches spec violations and formatting issues. | Does NOT scan for security — only validates format. |

### Manual Vetting Checklist (When Tools Aren't Enough)

Before adopting any skill scored 2/4 or higher on the Lethal Quartet:

```
1. READ THE FULL BODY. Don't just read the description — read every line of the Markdown body.
   Look for:
   □ curl/wget/fetch commands (data exfiltration)
   □ File read instructions targeting sensitive paths (~/.ssh, ~/.aws, ~/.env)
   □ Instructions to "save" or "remember" credentials
   □ Overrides of safety behavior ("ignore previous", "always allow", "skip confirmation")
   □ Base64-encoded strings (obfuscation)
   □ URLs you don't recognize

2. CHECK THE scripts/ DIRECTORY. If the skill includes executable scripts:
   □ Read every script file
   □ Check for network calls, file access outside the skill directory, process spawning
   □ Verify scripts match what the SKILL.md body describes

3. CHECK THE SOURCE.
   □ Who published it? Verified publisher or anonymous account?
   □ How old is the account? (ClawHavoc accounts were days old)
   □ How many other skills from this publisher? (hightower6eu had 677 — all malicious)
   □ Does the skill do what its description claims? (typosquats often don't)

4. TEST IN ISOLATION.
   □ Run the skill in a sandboxed environment first
   □ Monitor network traffic during execution
   □ Check what files it reads and writes
   □ Verify it doesn't modify MEMORY.md or SOUL.md unexpectedly
```

### For Skill Authors (Writing Safe Skills)

- Never instruct the agent to save credentials to memory or files — use environment variables
- Never embed API keys or secrets in the skill body — reference env vars by name
- Never instruct the agent to send data to external URLs unless that's the skill's explicit purpose
- Minimize tool permissions — use `allowed-tools` to whitelist only what the skill needs
- Keep the body under 5,000 tokens — verbose skills waste context and hide malicious instructions in the noise
- Test with `skills-ref validate` before publishing
- Scan with Cisco Skill Scanner before publishing

## 12. Tested Integration

*(To be filled after manual testing across platforms)*

## 13. Composability Notes

### Cross-Platform Compatibility

| Platform | Support Level | Extensions Used | Loads From |
|---|---|---|---|
| **Claude Code** | Native, full spec | Base spec only | `.claude/skills/`, project skills |
| **OpenClaw** | Native, extended | `metadata.openclaw` block + custom top-level keys | `~/.openclaw/skills/`, workspace, ClawHub |
| **OpenAI Codex** | Native, extended | `agents/openai.yaml` (separate file) | `.agents/skills/`, `~/.codex/skills/`, Skills API |
| **GitHub Copilot** | Native | Base spec | Project-level skills |
| **Cursor** | Native | Base spec | Project-level skills |
| **37+ frameworks** | Via `npx skills` (Vercel) | Converted per-framework | CLI tool handles placement |

### RFC #11919 Composability (Proposed, Not Implemented)

OpenClaw's community proposed six composition mechanisms for SKILL.md. Zero maintainer engagement after 30+ days. The design is sound but not being built:

| Mechanism | What It Does | Your Relevance |
|---|---|---|
| `requires.skills` | Hard dependency — skill won't load without these | Use in block profiles for dependency chains |
| `optionalSkills` | Soft dependency — enhances when present | Use for optional enhancement documentation |
| `provides` | Declares capability interfaces | Use to classify what a block offers |
| `requires.interfaces` | Decouples from specific implementations | Use for abstract dependency specification |
| `extends` | Single-inheritance specialization | Document when one skill builds on another |
| Conditional sections | `<!-- @if-skill X -->` toggles in body | Note when skills have conditional behavior |

**Forward-compatibility recommendation**: If you're building a skill loader or a skill management system, align your dependency fields with RFC #11919's vocabulary (`requires.skills`, `provides`, `optionalSkills`). If the RFC is ever implemented, your system will be compatible. If it isn't, you've still got a clean dependency model.

### What Composes Well

- **AGENTS.md** (Linux Foundation standard): Always-on project-level instructions. Complementary — AGENTS.md says "how to work in this repo," SKILL.md says "how to do this specific task." Used in 60,000+ projects.
- **SOUL.md / USER.md**: Persona and identity files. Skills reference these for tone/behavior context.
- **MCP servers**: Skills can reference MCP tools for runtime capabilities. The skill provides instructions; MCP provides the tool invocation layer.
- **Any LLM**: The body is natural language. Works with Claude, GPT, Gemini, Llama, Mistral — any model that can follow instructions.

### What Doesn't Compose

- **Skills across platforms with different tool names**: A skill that references `Bash` (Claude Code) may need `shell` (Codex) or `terminal` (Cursor). Tool name mapping is manual.
- **Skills with MCP dependencies**: MCP server URLs and transport configs are environment-specific. A skill that needs `openaiDeveloperDocs` MCP server won't work without that server configured.
- **Skills that assume a specific filesystem layout**: Skills referencing `~/.openclaw/` paths won't find them in Claude Code or Codex environments.

## 14. Token Footprint

| Load Level | Approximate Tokens | When |
|---|---|---|
| Metadata only (YAML frontmatter) | ~100 tokens | Progressive disclosure — loaded at startup per skill |
| Typical skill body | ~2,000-4,000 tokens | On-demand when skill activates |
| This block profile (full) | ~4,800 tokens | Developer evaluating the format and security landscape |
| Anthropic spec (recommended max body) | <5,000 tokens | Hard limit for well-designed skills |

### Token Bloat Context

Token budget matters because of documented bloat problems:
- OpenClaw users report **150,000+ token** system prompts from eager skill loading (Issue #21999)
- **166,000 tokens** consumed on first turn from skills injection (community report)
- Performance degrades from 27 tok/sec to **0.125 tok/sec** under bloat
- **22,429 filesystem operations** at startup from loading all channel SDKs (Issue #28587)

The progressive disclosure pattern (metadata at startup, body on-demand) exists specifically to fight this. Skills that exceed the 5,000-token recommendation undermine the pattern and contribute to bloat. Your agent's skill loader should enforce a token budget.

---

## The Gap: What SKILL.md Doesn't Cover

SKILL.md answers: **"How should an agent USE this tool?"**

It does NOT answer:

| Question | SKILL.md | Your Block Profile |
|---|---|---|
| Is it safe to use? | No security assessment | Lethal Quartet + per-skill checklist |
| Can I extract it from its repo? | Not addressed | Extractability rating + what breaks |
| What dependencies follow extraction? | Not addressed | Dependency chain analysis |
| What other frameworks can it plug into? | Not addressed | Tested cross-framework integrations |
| Are there existing standalone versions? | Not addressed | Links + comparative analysis |
| How do credentials flow through it? | Not addressed | Credential handling path |
| What's the security track record? | Not addressed | CVE mapping + risk classes |

Block profiles are to SKILL.md what nutrition labels are to food packaging — the assessment metadata that helps you decide BEFORE you consume.
