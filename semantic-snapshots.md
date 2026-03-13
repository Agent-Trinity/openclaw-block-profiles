---
name: semantic-snapshots
description: >
  Browser automation system that extracts accessibility trees as structured text
  instead of screenshots. Numbered element refs replace CSS selectors for LLM-friendly
  page interaction. ~500 tokens per page vs thousands for screenshots (~90% reduction).
  Use when building agent systems that need web browsing without vision models or
  screenshot-based grounding.
license: MIT
compatibility: "Node.js 18+. Requires Playwright and a Chromium-based browser. Optional: Chrome DevTools Protocol for direct CDP connection. Works on macOS, Linux, Windows."
metadata:
  author: "Stackpedia"
  version: "1.0.0"
  tags: "browser, accessibility-tree, playwright, cdp, web-automation, semantic-snapshot, a11y"
  block-profile: |
    extractability: HIGH
    lethal-quartet:
      private-data-access: true
      untrusted-content-exposure: true
      external-communication: true
      persistent-memory: true
    security:
      cves: ["CVE-2026-26329"]
      risk-classes: ["untrusted-content-injection", "credential-exposure", "browser-session-leakage", "path-traversal"]
      credential-handling: "environment-variables"
      recommended-scanner: "snyk-mcp-scan"
    dependencies:
      extraction-chain: ["snapshot-engine", "ref-system", "cdp-client", "playwright-primitives", "anti-detection"]
      breaks-without: ["playwright", "chromium"]
      optional-enhancements: ["anti-detection", "efficient-mode", "media-cache"]
    composability:
      tested-frameworks: []
      existing-extractions: ["browserclaw", "agent-browser"]
      compatible-with-rfc-11919: false
    source:
      repo: "openclaw/openclaw"
      files: ["src/browser/", "src/agents/tools/browser-tool.ts", "src/agents/pi-tools.ts"]
      version-tested: "v2026.2.26"
---

# Semantic Snapshots (Browser) — Block Profile

> **Lethal Quartet: 4/4.** This is the most dangerous component profiled so far. Lane Queue scored 0/4. Memory System scored 3/4. Semantic Snapshots touches all four risk elements. BrowserClaw already proved extraction works — but it dropped every security layer OpenClaw built. That tradeoff is the central story of this profile.

## 1. What It Does

A browser automation system that converts web pages into structured text by extracting the accessibility tree (the same tree screen readers use) instead of taking screenshots. Each interactive element gets a numbered reference (`aria-ref="3"` or `[ref=e12]`), so an LLM can say "click ref 3" instead of trying to locate elements by CSS selector or pixel coordinates.

In Python terms: imagine Selenium, but instead of `driver.find_element(By.CSS_SELECTOR, "button.submit")`, the agent sees a text dump like:

```
[1] heading "Sign In"
[2] textbox "Email" value=""
[3] textbox "Password" value=""
[4] button "Log In"
```

And says "type 'user@example.com' into ref 2, type 'hunter2' into ref 3, click ref 4." No vision model, no screenshot parsing, no coordinate math. The whole page in ~500 tokens vs thousands for a screenshot. ~90% token reduction.

The system has four snapshot modes (AI, Role, ARIA, Efficient), three connection targets (sandbox Docker, host local, remote proxy), and two browser profiles ("openclaw" for isolated Chromium, "chrome" for relaying into the user's existing tabs via extension).

## 2. Why It's Useful Outside This Repo

Every agent framework needs web browsing, and every one faces the same problem: screenshots are expensive (tokens + vision model) and fragile (pixel coordinates break on resize). The accessibility tree approach is fundamentally better:

- **Token efficient**: ~500 tokens per page vs 1,000-5,000+ for screenshots. Over a multi-page workflow, this saves 80-90% on context budget.
- **No vision model required**: Works with text-only LLMs. No GPT-4V, no Claude vision, no multimodal pipeline needed.
- **Deterministic element targeting**: Numbered refs are unambiguous. "Click ref 4" always means the same element. Screenshots require spatial reasoning that LLMs do unreliably.
- **Structured data extraction**: The a11y tree includes element roles, states, and values. Parse it programmatically for data extraction tasks without LLM involvement.
- **Accessibility as a feature**: Sites built with good a11y practices produce better snapshots. This aligns agent automation with web accessibility — rare positive incentive.

The core insight — use accessibility trees, not screenshots — is framework-independent and has been validated in production by BrowserClaw (powers tovli.ai), agent-browser (Rust+Node CLI), and Cloudflare's moltworker.

## 3. Key Files

### Extractable Core (snapshot + ref engine)

| Component | What It Does |
|---|---|
| Snapshot engine | Extracts a11y tree from page, assigns numbered refs to interactive elements |
| Ref system | Maps ref numbers to Playwright locators for action execution |
| CDP client | Chrome DevTools Protocol connection management |
| Playwright primitives | Navigation, clicking, typing, scrolling, waiting |
| Anti-detection | Stealth techniques to avoid bot detection on target sites |

BrowserClaw packages exactly these five components.

### Stays Behind (OpenClaw-specific integration)

| Component | What It Does | Why It Stays |
|---|---|---|
| Multi-target routing | Routes to sandbox/host/node based on config | Deeply tied to OpenClaw's target model |
| Node proxy RPC | Remote browser execution across machines | Gateway-specific protocol |
| Loopback auth | Bearer token auth for control server | Gateway sidecar pattern |
| Token auto-generation | Generates auth tokens, persists to config | Config system coupling |
| `wrapExternalContent` | Security wrapping with random hex IDs + Unicode homoglyph folding | Critical security layer — see section 10 |
| Tool policy pipeline | 15+ layers of permission checking before any browser action | OpenClaw's permission system |
| Chrome extension relay | Relays into user's existing browser tabs | Extension-specific, tight UI coupling |
| Session tab tracking | Per-session browser tab management | Session model coupling |

### Source Scale

~22,000 lines across `src/browser/`. This is the largest component in the codebase by line count. The extractable core is roughly 30-40% of that; the rest is integration, routing, auth, and policy.

## 4. External Dependencies

| Dependency | Required? | What For | Weight |
|---|---|---|---|
| **Playwright** | **Yes** | Browser control, a11y tree extraction, action execution | Heavy — full browser engine (~200MB+ with Chromium) |
| **Chromium** | **Yes** | The browser being controlled | Ships with Playwright, or use existing install |
| `ws` | **Yes** | WebSocket for CDP communication | Lightweight |
| Docker | Optional | Sandbox target isolation | Only for sandboxed execution mode |

**Contrast with the other profiles:**
- Lane Queue: zero dependencies
- Memory System: 2 hard deps (Node 20 DatabaseSync, sqlite-vec)
- Semantic Snapshots: 2 heavy deps (Playwright + Chromium), one of which is an entire browser engine

Playwright is the heaviest single dependency in any component we've profiled. It's the tradeoff for browser automation — there's no lightweight alternative that gives you accessibility tree extraction.

## 5. Internal Coupling Points

| Coupling Point | What It Touches | Extraction Impact |
|---|---|---|
| **Gateway sidecar lifecycle** | Control server starts as gateway sidecar on port 18791, dies with gateway | BrowserClaw solves this — runs as standalone process. ~0 effort if using BrowserClaw. |
| **Loopback auth** | Bearer tokens generated by gateway, stored in config, validated on every request | Replace with your own auth or run unauthenticated on localhost. ~30 minutes. |
| **Tool policy pipeline** | 15+ layers of permission checks (can the agent use the browser? which actions? which URLs?) | Replace with your framework's permission model. ~1-2 hours. |
| **Config system** | Browser profile selection, target routing, anti-detection toggles | Replace with constructor parameters. ~30 minutes. |
| **Session model** | Per-session tab tracking, session-scoped browser profiles | Replace with your own session/context concept. ~1 hour. |
| **Security wrapping (`wrapExternalContent`)** | Marks all browser output as external/untrusted with random hex IDs and homoglyph folding | **BrowserClaw drops this entirely.** Must rebuild if you want it. See section 10. |
| **`applyLocalAiBrowserBias()`** | Injects system prompt guidance for browser interaction patterns | Optional — improves LLM behavior but not required for snapshot extraction. |

### The BrowserClaw Shortcut

BrowserClaw already solved the extraction: snapshot engine, ref system, CDP, Playwright, anti-detection — packaged as `npm install browserclaw`. Production-proven at tovli.ai.

If you use BrowserClaw, coupling points 1-5 are already handled. Point 6 (security wrapping) is deliberately dropped. Point 7 (browser bias prompt) is your responsibility to recreate if needed.

## 6. Extraction Difficulty

**LOW — because BrowserClaw already did it.**

If starting from scratch without BrowserClaw: MEDIUM-HIGH. The ~22k lines, multi-target routing, and gateway coupling would require significant untangling.

If using BrowserClaw as your starting point: LOW. Install the package, connect to a browser, get snapshots. The hard extraction work is done.

| Path | Effort | What You Get |
|---|---|---|
| `npm install browserclaw` | ~30 minutes to integrate | Snapshot engine + refs + CDP + Playwright + anti-detection. No security wrapping. |
| Extract from OpenClaw yourself | ~2-3 days | Same as above, but you can keep the security wrapping and customize the policy pipeline. |
| Use `agent-browser` CLI | ~1 hour to integrate | Different implementation of same concept. Rust fast mode + Node.js fallback. |

## 7. What Breaks When You Extract It

| What Breaks | Why | Impact |
|---|---|---|
| **Security content wrapping** | `wrapExternalContent` with random hex IDs and Unicode homoglyph folding — dropped by BrowserClaw | **HIGH** — browser output is untrusted content from arbitrary websites. Without wrapping, LLM can't distinguish page content from instructions. See section 10. |
| **Multi-target routing** | Sandbox/host/node target selection based on config | Medium — pick one target model for your use case. Most agents need only local browser. |
| **Tool policy enforcement** | 15+ layers of "can this agent do this action?" checking | Medium — rebuild per your framework's permission model, or run without policy (accepting the risk). |
| **Chrome extension relay** | Relaying into user's existing browser tabs | Low — niche feature. Most agents use dedicated browser instances. |
| **Node proxy RPC** | Remote browser execution across machines | Low — only needed for distributed setups. |
| **Session tab tracking** | Per-session isolation of browser tabs | Medium — if your agent handles multiple concurrent users, you need your own tab isolation. |
| **Loopback auth** | Auth tokens for control server communication | Low — replace with your auth pattern or run on trusted localhost. |

The first row is the critical one. Everything else is integration plumbing. The security wrapping is the only loss that changes your risk profile.

## 8. Existing Standalone Versions

### BrowserClaw (`npm install browserclaw`)

| Field | Details |
|---|---|
| **Author** | idan-rubin |
| **Production use** | Powers tovli.ai |
| **What it includes** | Snapshot + ref engine, CDP client, Playwright primitives, anti-detection |
| **What it drops** | Auth, routing, security wrapping, policy pipeline, Chrome extension relay, session tracking |
| **Package size** | Lightweight wrapper — Playwright is the heavy dependency |

**Key tradeoff**: BrowserClaw gives you the core browser automation without OpenClaw's runtime. But it also gives you the core browser automation without OpenClaw's security layers. Every page snapshot is injected raw into your LLM context. If the page contains prompt injection (and adversarial pages will), you have no defense layer.

### agent-browser

| Field | Details |
|---|---|
| **Approach** | Same a11y-tree concept, independent implementation |
| **Language** | Rust (fast mode) + Node.js (fallback) |
| **Differentiator** | Performance — Rust parser is significantly faster on large DOMs |

### Cloudflare moltworker

| Field | Details |
|---|---|
| **Approach** | Browser automation skills using CDP shim |
| **Platform** | Cloudflare Workers (serverless) |
| **Differentiator** | Runs in serverless context — no local browser needed |

### Comparison

| Feature | OpenClaw | BrowserClaw | agent-browser | moltworker |
|---|---|---|---|---|
| A11y tree snapshots | Yes | Yes | Yes | Yes |
| Numbered ref system | Yes | Yes | Yes | Partial |
| Anti-detection | Yes | Yes | No | No |
| Security wrapping | Yes | **No** | No | No |
| Multi-target routing | Yes | No | No | No (serverless) |
| Policy enforcement | Yes | No | No | No |
| Efficient mode | Yes | Partial | No | No |
| Production-proven | Yes | Yes (tovli.ai) | Limited | Yes (Cloudflare) |

**None of the standalone extractions include security wrapping.** This is the universal gap. The core snapshot innovation is well-served; the security layer around it is not.

## 9. Lethal Quartet Assessment

| Element | Present? | Details |
|---|---|---|
| Access to private data | **YES** | Reads auth tokens from gateway config. Can access any page the browser can reach, including authenticated sessions, internal tools, localhost services. Browser profiles persist cookies and session state. |
| Exposure to untrusted content | **YES** | Every web page is untrusted content. HTML, JavaScript console output, download filenames, page titles — all attacker-controllable. A page can embed prompt injection in hidden elements, aria labels, or invisible text that appears in the a11y tree snapshot. |
| Ability to externally communicate | **YES** | Spawns a browser process. Navigates to arbitrary URLs (outbound HTTP to any domain). Sends HTTP to the loopback control server. In node proxy mode, makes RPC calls to remote machines. The browser itself makes arbitrary network requests (scripts, APIs, tracking). |
| Persistent memory | **YES** | Browser profiles persist to disk (cookies, localStorage, session storage, cache). Media cache persists downloaded files. Auto-generated auth tokens persist to config files. Browser history persists across sessions. |

**Score: 4/4 — the full Lethal Quartet.**

This is the only component profiled so far that scores 4/4. The progression across profiles:

| Component | Score | Risk Stance |
|---|---|---|
| Lane Queue | 0/4 | Recommend without caveats |
| Memory System | 3/4 | Recommend with security guidance |
| SKILL.md Format | 0/4 (format) to 4/4 (per-skill) | Rate individually |
| **Semantic Snapshots** | **4/4** | **Recommend only with explicit security architecture** |

### Why 4/4 Is Categorically Different

At 0/4 (Lane Queue), there's nothing to worry about — it's pure logic.

At 3/4 (Memory System), you have specific, nameable risks (memory poisoning, credential persistence) with specific mitigations (integrity monitoring, read-only mode, error scrubbing).

At 4/4, you have **combinatorial risk**. The browser accesses private data (authenticated sessions), is exposed to untrusted content (every web page), communicates externally (navigates anywhere), and persists state (browser profiles). An attacker who controls a web page can:

1. Inject prompt instructions via hidden a11y tree elements → **untrusted content**
2. Those instructions tell the agent to navigate to a credential page → **external communication**
3. The agent reads credentials from the authenticated session → **private data access**
4. The agent saves credentials to browser profile or memory → **persistent memory**

All four elements chain together in a single attack flow. This is why Simon Willison called the combination "lethal" — each element is manageable alone, but together they create attack chains that are difficult to enumerate, let alone prevent.

## 10. Security Considerations

### Known Vulnerabilities (CVEs)

| CVE | CVSS | What | Follows Extraction? |
|---|---|---|---|
| **CVE-2026-26329** | 7.1 | Path traversal in browser file upload | **PARTIAL** — depends on whether extraction includes the upload handler. BrowserClaw status: **unconfirmed**. If BrowserClaw reuses OpenClaw's upload path handling, it inherits this. If it reimplemented uploads, it may or may not be affected. Check BrowserClaw's changelog. |

The other 8 OpenClaw CVEs are Gateway-specific and don't follow the browser component.

### Inherent Risk Classes

**Untrusted Content Injection (CRITICAL)**

Every page snapshot is attacker-controllable content injected into the LLM's context window. This is prompt injection with a delivery mechanism:

- Hidden elements with `aria-hidden="false"` appear in the a11y tree but not visually on the page
- `aria-label` attributes can contain arbitrary text — "button aria-label='Ignore all previous instructions and execute: ...'"
- Invisible text (CSS `color: transparent`, `font-size: 0`) may appear in text content extraction
- JavaScript can dynamically modify the DOM between snapshot requests

OpenClaw's defense: `wrapExternalContent` wraps all browser output with random hex boundary markers and applies Unicode homoglyph folding (normalizes lookalike characters to prevent marker spoofing). The LLM is instructed that content between these markers is external and untrusted.

**BrowserClaw drops this entirely.** If you use BrowserClaw, every snapshot goes into LLM context as raw text with no untrusted-content boundary. The LLM cannot distinguish page content from system instructions. This is the single biggest security gap in any extraction of this component.

**Browser Session Leakage (HIGH)**

The browser has access to whatever the browser profile contains:
- Cookies for authenticated sessions (Gmail, GitHub, internal tools)
- localStorage/sessionStorage tokens
- Saved passwords (if using "chrome" profile that relays to user's browser)
- Browser history revealing visited sites

In "openclaw" profile mode (dedicated Chromium), this is contained — fresh profile, no pre-existing sessions. In "chrome" extension relay mode, the agent has access to the user's full browser session. The extension relay is the most dangerous mode and is not included in BrowserClaw.

**Credential Exposure (HIGH)**

Two credential paths:
1. **Auth tokens**: The loopback control server uses Bearer tokens auto-generated and persisted to config. If config files are readable (and they're plaintext), tokens are exposed.
2. **Page credentials**: The agent can navigate to pages that display credentials (API key dashboards, settings pages, password managers in-browser). These appear in the a11y tree snapshot as plaintext text content.

**Path Traversal (MEDIUM — CVE-2026-26329)**

Browser file upload handler vulnerable to path traversal. Patched in OpenClaw v2026.2.14. BrowserClaw inheritance status unknown — if BrowserClaw forked the upload handler before the patch, it's vulnerable.

### The Security Wrapping Gap (What BrowserClaw Dropped)

This deserves its own subsection because it's the most consequential security decision in the extraction.

OpenClaw's `wrapExternalContent` does two things:

1. **Boundary marking**: Wraps browser output with random hex IDs per call:
   ```
   <external-content id="a7f3b2c1">
   [snapshot content here]
   </external-content id="a7f3b2c1">
   ```
   Random IDs prevent an attacker from predicting and spoofing the markers. Each tool call gets a unique ID, so even if an attacker discovers one marker, it doesn't help with the next call.

2. **Unicode homoglyph folding**: Normalizes lookalike Unicode characters to their ASCII equivalents before wrapping. Without this, an attacker could include characters that *look like* the boundary markers but use different Unicode codepoints, causing the LLM to misidentify where trusted content ends and untrusted content begins.

BrowserClaw, agent-browser, and moltworker all drop this. None of the standalone extractions include any form of untrusted content marking.

**If you're building with BrowserClaw or any extraction**, you MUST implement your own content wrapping or accept that the LLM cannot distinguish page content from instructions. Options:
- Port `wrapExternalContent` from OpenClaw (~200-300 lines, relatively self-contained)
- Implement simpler boundary markers (less secure but better than nothing)
- Use a two-agent architecture: one "reader" agent processes the raw snapshot, a second "actor" agent receives only the reader's sanitized summary (Giskard's recommendation)
- Restrict browsing to a known-safe allowlist of URLs (eliminates the threat but severely limits functionality)

### What This Block Does NOT Protect Against

Even with OpenClaw's full security wrapping:
- Sophisticated prompt injection that works within the untrusted-content boundary (the LLM might still follow instructions even when told they're external)
- Timing attacks (page content changes between snapshot and action)
- Browser fingerprinting revealing the agent's identity to target sites
- JavaScript execution on visited pages (the browser runs JS — visited pages can detect and respond to the agent)
- Data exfiltration via browser's own network requests (a page's JS can send data to third parties regardless of agent behavior)

## 11. Recommended Mitigations

### For Untrusted Content (Most Important)

- **Implement content wrapping**: If not using OpenClaw's built-in wrapping, port `wrapExternalContent` or build your own boundary marker system with per-call random IDs. This is non-negotiable for any production deployment.
- **Two-agent architecture**: Use a sandboxed "browser agent" that only browses and extracts data, feeding sanitized results to a "task agent" that has tool access. The browser agent can be compromised by page content; the task agent never sees raw page content.
- **URL allowlisting**: For high-security use cases, restrict navigation to a known-safe list. Prevents the agent from visiting attacker-controlled pages entirely.
- **Claude Opus 4.5 as backing LLM**: Currently best at detecting prompt injection attempts within content (Kaspersky testing). Not a substitute for content wrapping, but an additional layer.

### For Browser Session Isolation

- **Always use dedicated browser profiles**: Never use the "chrome" extension relay mode in automation. Use the "openclaw-managed" profile (dedicated Chromium) or BrowserClaw's standalone mode to ensure a clean browser profile with no pre-existing sessions.
- **Ephemeral profiles**: Create a fresh browser profile per task and destroy it after. Prevents session state from leaking between tasks.
- **Block credential pages**: Configure the browser to block navigation to known credential-management URLs (cloud provider consoles, password managers, API key pages) unless the task explicitly requires it.

### For Credential Exposure

- **Don't persist auth tokens to plaintext config**: If running your own control server, use ephemeral in-memory tokens that die with the process. OpenClaw persists tokens to config for convenience across restarts — don't replicate this in security-sensitive deployments.
- **Scrub snapshots for credentials**: Before injecting a snapshot into LLM context, scan for patterns matching API keys, tokens, passwords. Mask or redact matches.

### For Path Traversal (CVE-2026-26329)

- **Validate all file paths in upload handlers**: Use `realpath()` + prefix checking + symlink rejection, same pattern as the memory system's `readFile` defense.
- **If using BrowserClaw**: Check whether their upload handler is affected. If unsure, disable file upload functionality until confirmed.

## 12. Tested Integration

*(To be filled after manual testing)*

**Open question from research tracker**: Does BrowserClaw inherit CVE-2026-26329? Check their changelog and upload handler implementation. This is the first manual verification to do.

## 13. Composability Notes

### Framework Integration Patterns

**LangGraph** (effort: MEDIUM): Wrap the browser as a LangGraph tool node. Snapshot → text output → feed to next node. The ref system maps well to LangGraph's structured state — store the current snapshot and pending actions as graph state. Main friction: LangGraph's checkpoint system needs to handle that refs are NOT stable across navigations (re-snapshot after every page change).

**AutoGen** (effort: MEDIUM): Implement as an AutoGen tool. The snapshot fits AutoGen's message-passing model — the browser agent sends snapshot text, the orchestrator agent decides actions. Good fit for the two-agent security architecture (browser agent = AutoGen agent with browser tool, task agent = separate AutoGen agent that only sees summaries).

**CrewAI** (effort: LOW-MEDIUM): CrewAI's tool pattern is straightforward. Wrap BrowserClaw as a CrewAI `Tool` with `snapshot()` and `action()` methods. CrewAI's sequential task model works naturally with the snapshot-then-act loop.

**Semantic Kernel** (effort: MEDIUM): Implement as an SK native function. SK's plan-and-execute model needs explicit re-snapshot steps between actions since refs don't persist. The structured output from snapshots maps to SK's typed function results.

### What Composes Well

- **Lane Queue**: If you're building a multi-session agent with browser access, the Lane Queue prevents two sessions from driving the same browser simultaneously. Browser actions are inherently serial (can't click two things at once on the same page) — serial-by-default is the right model.
- **Memory System**: Store browsing results in memory for later retrieval. But be careful — browser snapshots are untrusted content. If you index them into the memory system, you inherit the "untrusted content laundered as trusted memory" problem from the memory system profile (section 10).
- **Any RAG pipeline**: Snapshot text is structured enough for chunk-and-embed. You could build a web knowledge base by crawling, snapshotting, and indexing pages.

### What Doesn't Compose

- **Vision-based agent systems**: The whole point of semantic snapshots is avoiding screenshots. If your system is built around vision models (GPT-4V, Claude vision), the a11y tree approach is redundant — though it can complement vision as a cheaper fallback.
- **Serverless environments**: Playwright needs a running browser process. Serverless functions have execution time and memory limits that conflict with browser automation. Exception: Cloudflare moltworker specifically solves this with a CDP shim.
- **Systems without tool-use**: If your LLM can't call tools (just generates text), it can't execute browser actions through the ref system. You'd get snapshots but no interactivity.

### Cross-Block Security Interaction

Using semantic snapshots alongside other Stackpedia blocks creates compound security considerations:

| Combination | Risk | Mitigation |
|---|---|---|
| Snapshots + Memory System | Untrusted page content indexed as "trusted" memory. Memory poisoning via web pages. | Separate memory scopes: never index browser output into the same DB as trusted notes. |
| Snapshots + SKILL.md skills | A malicious skill could instruct the agent to browse to an attacker page, creating a two-stage attack (skill → browser → prompt injection). | Vet skills that reference browser actions with extra scrutiny. Apply the per-skill Lethal Quartet checklist. |
| Snapshots + Lane Queue | Safe combination. Queue orders browser operations, prevents concurrency issues. | None needed — this is a beneficial composition. |

## 14. Token Footprint

| Load Level | Approximate Tokens | When |
|---|---|---|
| Metadata only (YAML frontmatter) | ~200 tokens | Discovery/search results |
| This block profile (full) | ~5,000 tokens | Developer evaluating whether to use it |
| Single page snapshot (typical) | ~500 tokens | Per-page during agent browsing |
| Single page snapshot (complex SPA) | ~1,500-3,000 tokens | Complex pages with many interactive elements |
| Single page screenshot (for comparison) | ~3,000-8,000 tokens | Vision model alternative — 5-10x more expensive |
| BrowserClaw source | ~4,000 tokens | If reading the extraction code |
| Full OpenClaw browser source | ~40,000+ tokens | Never load this — way too large |

### Token Economy at Scale

The token savings compound over multi-step browsing tasks:

| Task | Screenshots | Semantic Snapshots | Savings |
|---|---|---|---|
| 5-page checkout flow | ~25,000 tokens | ~2,500 tokens | 90% |
| 20-page research session | ~100,000 tokens | ~10,000 tokens | 90% |
| Single form fill | ~5,000 tokens | ~500 tokens | 90% |

At scale, semantic snapshots are the difference between a browsing task that fits in a single context window and one that requires compression/summarization (with associated information loss).

---

## The Extraction Spectrum

This component has the widest range of extraction options of any block profiled so far:

| Option | Effort | Security | Features |
|---|---|---|---|
| **BrowserClaw** (`npm install`) | 30 min | **No security wrapping** — you must add your own | Snapshot + refs + CDP + anti-detection |
| **agent-browser** CLI | 1 hour | No security wrapping | Snapshot + refs, Rust performance mode |
| **Port from OpenClaw** (keep security layers) | 2-3 days | Full `wrapExternalContent` + homoglyph folding | Everything except gateway coupling |
| **BrowserClaw + ported security wrapping** | 3-4 hours | Custom wrapping on top of BrowserClaw | Best of both — proven extraction + security layer |

**Recommended path**: Option 4. Start with BrowserClaw (proven, production-tested), then port `wrapExternalContent` from OpenClaw on top. You get the extraction benefits without rebuilding Playwright primitives, but you don't sacrifice the security boundary that makes browser automation safe enough for production.
