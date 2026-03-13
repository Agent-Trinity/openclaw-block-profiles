# OpenClaw Block Profiles

Structured extraction guides for OpenClaw's five extractable components. Each profile covers: what the component does, how to extract it, what dependencies follow, security assessment, and cross-framework integration patterns.

## The Security Gradient

| Component | Lethal Quartet | Extractability | What It Means |
|---|---|---|---|
| [Lane Queue](lane-queue.md) | 0/4 | HIGH | Pure logic, zero I/O. Two files, swap 3 imports, works standalone. No equivalent in any other framework. |
| [Workspace Config](workspace-config.md) | 2/4 | HIGH (format) | Seven Markdown files defining agent identity. Format is portable. MEMORY.md writeback is the poisoning vector. |
| [Memory System](memory-system.md) | 3/4 | MEDIUM | Hybrid BM25+vector search over Markdown files. memsearch extracted the basics. 10 production features remain. |
| [SKILL.md Format](skill-format.md) | 0/4 to 4/4 | N/A | Cross-platform standard. Already portable. 12-20% of ClawHub skills are malicious. Rate each skill individually. |
| [Semantic Snapshots](semantic-snapshots.md) | 4/4 | HIGH | Browser a11y tree extraction. BrowserClaw works but dropped all security wrapping. |

## What's a Block Profile?

A structured breakdown of a software component covering 14 fields:

1. What it does
2. Why it's useful outside this repo
3. Key files
4. External dependencies
5. Internal coupling points
6. Extraction difficulty
7. What breaks when you extract it
8. Existing standalone versions
9. Lethal Quartet assessment (private data / untrusted content / external comms / persistent memory)
10. Security considerations
11. Recommended mitigations
12. Tested integration
13. Composability notes
14. Token footprint

Think of them as nutrition labels for software components. SKILL.md tells agents HOW to use a tool. Block profiles tell developers WHETHER to use it.

## The Lethal Quartet

Security framework (Simon Willison / Palo Alto Networks) applied to every component. Four questions:

- Does it access private data or secrets?
- Does it process untrusted content?
- Can it communicate externally?
- Does it persist state across sessions?

A component scoring 0/4 (Lane Queue) is safe to use without caveats. A component scoring 4/4 (Semantic Snapshots) requires explicit security architecture.

## Source

Analysis performed against OpenClaw v2026.2.26. Based on source code review, dependency chain mapping, and cross-referencing with published CVEs and security research (Koi Security, Snyk ToxicSkills, Palo Alto Unit 42, Zenity).

## License

MIT