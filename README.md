# Prompt Provenance

A draft specification for **versioned, lineaged, reviewable prompt records** that travel with the prompt itself — not in a wiki, not in a spreadsheet.

Every team running LLMs in production hits the same problem: prompts are the most-edited, least-version-controlled artifact in the stack. The same prompt exists in three notebooks, two repos, and someone's Slack DM, with no record of who changed what, what they were trying to fix, and whether the new version actually passed the evals.

Prompt Provenance is a JSON document that fixes this. One record per prompt version, machine-readable, designed to live alongside the prompt content in any repo, registry, or artifact store.

## The three pillars

| Pillar | What it does |
|---|---|
| **Record** | Every prompt version gets a provenance document with a content hash, author chain, and approval state |
| **Reference** | Every version cites its parent — a directed chain of derivations (forks, tunes, patches) |
| **Review** | Every record carries the eval results that gated approval, including which suites ran and what they returned |

## Why not just put it in Git?

Git tracks file changes. It does *not* track:

- **Eval suites that ran against a prompt version** — these are external artifacts living in CI logs, eval tooling, or a vendor dashboard
- **Approval chain** — who reviewed, who approved, with what scope
- **Derivation type** — was this a fork (new purpose), a tune (same purpose, better wording), or a patch (bugfix)
- **Cross-repo lineage** — a prompt forked from a vendor's library into your internal repo can't be expressed in a single Git history
- **Deprecation** — Git tells you what changed, not what is *no longer fit for production*

Prompt Provenance lives in Git, but adds the semantic layer Git can't represent on its own.

## Why not LangChain Hub / PromptHub / vendor X?

Vendor prompt registries solve part of this for their customers. Prompt Provenance is an open, vendor-neutral format that any registry can emit, any CI system can validate, and any reader can audit without an account.

## Quickstart

1. Pick a stable `prompt.id` (kebab-case, unique within your org).
2. Compute a content hash over the canonicalized prompt text (SHA-256 over the UTF-8 bytes with normalized line endings).
3. Write a provenance document conforming to [`provenance.schema.json`](provenance.schema.json). Start from [examples](examples/).
4. Commit the provenance document alongside the prompt content. One file per version. Path convention: `prompts/<id>/v<version>/provenance.json`.
5. (Optional) Publish a directory at `/.well-known/prompts/<id>/latest.json` pointing to the current production version.

## Files in this repo

- [`SPEC.md`](SPEC.md) — full v0.1 specification
- [`provenance.schema.json`](provenance.schema.json) — JSON Schema (draft 2020-12)
- [`examples/`](examples/) — reference documents for a new prompt, a tuned version, and a deprecated version

## Status

**v0.1 draft.** Issues and pull requests welcome.

## License

MIT-licensed. The specification text, JSON Schema, and example documents in this repository may be freely implemented, extended, redistributed, or incorporated into commercial or non-commercial products with attribution. Reference implementations of this spec (such as [mcp-kinetic-gain](https://github.com/mizcausevic-dev/mcp-kinetic-gain)) are licensed separately under AGPL-3.0.

## Kinetic Gain Protocol Suite

A family of open specifications for the answer-engine era. Each spec is a self-contained JSON document format with its own JSON Schema and reference examples; together they compose into an end-to-end account of entity, agent, prompt, tool, and citation.

| Spec | What it does |
|---|---|
| [AEO Protocol](https://github.com/mizcausevic-dev/aeo-protocol-spec) | Entity declaration at `/.well-known/aeo.json` — authoritative claims, citation preferences, audit hooks |
| **[Prompt Provenance](https://github.com/mizcausevic-dev/prompt-provenance-spec)** | Versioned, lineaged, reviewable LLM prompt records |
| [Agent Cards](https://github.com/mizcausevic-dev/agent-cards-spec) | Declarative agent capability and refusal disclosure |
| [AI Evidence Format](https://github.com/mizcausevic-dev/ai-evidence-format-spec) | Structured citations that travel with LLM-generated claims |
| [MCP Tool Cards](https://github.com/mizcausevic-dev/mcp-tool-card-spec) | Per-tool disclosure layered on Model Context Protocol servers |

---

**Connect:** [LinkedIn](https://www.linkedin.com/in/mirzacausevic/) · [Kinetic Gain](https://kineticgain.com) · [Medium](https://medium.com/@mizcausevic/) · [Skills](https://mizcausevic.com/skills/)
