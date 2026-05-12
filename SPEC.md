# Prompt Provenance v0.1 — Specification

**Status:** Draft
**Version:** 0.1.0
**Editor:** Miz Causevic
**License:** AGPL-3.0 (this document, schema, and examples). Implementations are unrestricted.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 1. Scope

This specification defines a JSON document format for recording the provenance of an LLM prompt: identity, lineage, authorship, evaluation, and deprecation state. It is designed for consumption by humans (in code review), CI systems (gating production deploys), and audit tooling.

The specification does **not** standardize:
- The prompt content format itself (use plain text, Jinja2, Handlebars, or any other format)
- The storage backend (use Git, an artifact store, a database, or a vendor registry)
- The eval suite format (this is referenced by URI, not embedded)

## 2. Terminology

- **Prompt** — the text or template consumed by an LLM. Outside the scope of this spec.
- **Prompt record** — one provenance document describing one version of one prompt.
- **Lineage** — the chain of parent records leading back to the original.
- **Derivation type** — how a child relates to its parent: `fork`, `tune`, or `patch`.
- **Approval chain** — the ordered list of (author, reviewers, approver) for one version.

## 3. The three pillars

### 3.1 Record

Every distinct version of every prompt **MUST** have its own provenance document. Documents are immutable once approved — a change requires a new version, which is a new record.

The document **MUST** include a content hash (`prompt.hash`) that uniquely identifies the prompt text. Implementations **MUST** use SHA-256 over the canonicalized UTF-8 bytes of the prompt content with normalized LF line endings.

### 3.2 Reference

Every record except the original **MUST** reference its parent via `lineage.parent`, in the form `<prompt-id>@<version>`. The chain **MUST** be acyclic.

The `lineage.derivation` field classifies the relationship:

| Value | Meaning |
|---|---|
| `fork` | A new prompt purpose forked from the parent. Different `prompt.id` from parent. |
| `tune` | Same purpose, refined wording. Same `prompt.id` as parent, incremented `version`. |
| `patch` | Bugfix to existing version (e.g. typo, formatting). Same `prompt.id`, MAY use patch-level semver bump. |

### 3.3 Review

Every record **SHOULD** include an `evaluations` array linking to one or more eval results. Each entry references an external eval suite — the spec does not embed eval results.

A record's `approval.state` **MUST** be one of:

| State | Meaning |
|---|---|
| `draft` | Authored but not yet reviewed |
| `under_review` | Open for review |
| `approved` | Reviewed and approved for production use |
| `rejected` | Reviewed and rejected; record retained for history |
| `deprecated` | Previously approved, now superseded or withdrawn |

State transitions **SHOULD** be unidirectional. A record transitioning to `deprecated` **MUST** include `deprecation.reason` and `deprecation.superseded_by` (when applicable).

## 4. Document structure

All members marked **required** **MUST** be present in any conforming document.

### 4.1 `provenance_version` (required)

A semver string identifying the specification version. For documents conforming to this draft, the value **MUST** be `"0.1"`.

### 4.2 `prompt` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Stable kebab-case identifier, unique within the owning organization. |
| `name` | string | yes | Human-readable display name. |
| `version` | string | yes | Semver. MUST increment for every new record. |
| `hash` | string | yes | `sha256:<hex>`. Computed over canonicalized UTF-8 bytes of the prompt content. |
| `content_uri` | URI | yes | Location of the prompt content. MAY be a git URL, file URI, or HTTPS URI. |
| `content_type` | string | yes | MIME-like identifier: `text/plain`, `text/jinja2`, `text/handlebars`, etc. |

### 4.3 `lineage` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `parent` | string | no | `<prompt-id>@<version>` of the immediate parent. Omitted for the original record. |
| `derivation` | enum | yes when `parent` set | One of `fork`, `tune`, `patch`. |
| `change_summary` | string | yes when `parent` set | One- or two-sentence description of what changed. |

### 4.4 `authorship` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `created_by` | string | yes | Identifier of the author (email, GitHub handle, or org-internal ID). |
| `reviewed_by` | array of string | no | Identifiers of reviewers. |
| `approved_by` | string | no | Identifier of the final approver. |
| `created_at` | datetime | yes | ISO 8601 UTC. |
| `approved_at` | datetime | no | ISO 8601 UTC. Required when `approval.state` is `approved`. |

### 4.5 `intent` (optional but recommended)

| Field | Type | Description |
|---|---|---|
| `purpose` | string | One-sentence description of what the prompt is for. |
| `in_scope` | array of string | Topics or use cases the prompt is designed to handle. |
| `out_of_scope` | array of string | Topics or use cases the prompt is NOT designed for. |
| `models_supported` | array of string | LLM identifiers the prompt has been tested with. Wildcards permitted (e.g. `claude-opus-4-*`). |

### 4.6 `evaluations` (optional)

An array of eval results. Each entry:

| Field | Type | Required | Description |
|---|---|---|---|
| `suite` | string | yes | Name of the eval suite. |
| `result_uri` | URI | yes | Location of the detailed results. |
| `score` | number | no | Overall score (0.0–1.0 if normalized; spec does not mandate a scale). |
| `passed` | boolean | yes | Whether the suite passed the configured threshold. |
| `ran_at` | datetime | yes | ISO 8601 UTC. |

### 4.7 `approval` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `state` | enum | yes | `draft` / `under_review` / `approved` / `rejected` / `deprecated`. |
| `policy_uri` | URI | no | Link to the approval policy that governed this decision. |

### 4.8 `deprecation` (required when state is `deprecated`)

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | yes | Why the prompt was deprecated. |
| `superseded_by` | string | no | `<prompt-id>@<version>` of the replacement. |
| `deprecated_at` | datetime | yes | ISO 8601 UTC. |

## 5. Hashing rules

The content hash **MUST** be computed as follows:

1. Read the prompt content as UTF-8.
2. Normalize line endings to LF (`\n`).
3. Strip a single trailing newline if present.
4. Compute SHA-256 over the resulting byte string.
5. Encode as lowercase hexadecimal, prefixed `sha256:`.

This canonicalization ensures CRLF/LF differences and trailing-newline artifacts do not produce false-negative provenance mismatches.

## 6. Conformance levels

| Level | Requirements |
|---|---|
| **Level 1 — Record** | Valid document, content hash present, derivation type set where applicable. |
| **Level 2 — Review** | Level 1, plus `approval.state` reaches `approved` only after at least one eval entry with `passed: true`. |
| **Level 3 — Chain** | Level 2, plus every parent referenced in `lineage.parent` is also retrievable and conforms to this spec. |

## 7. Security and privacy considerations

- **Prompt confidentiality.** The provenance document references the prompt content via `content_uri` — it does not embed the content. Implementers controlling proprietary prompts **SHOULD** keep `content_uri` on a private access surface even when the provenance document itself is public.
- **Author identity.** `created_by` / `reviewed_by` may contain personal identifiers. Implementations **SHOULD** support pseudonymous IDs for org-internal use.
- **Hash collisions.** SHA-256 is collision-resistant for the foreseeable future. Implementations **MAY** add additional hashes (e.g. `sha3-256`) by extending the schema; consumers **MUST** treat unrecognized hash prefixes as informational, not authoritative.

## 8. Relationship to existing work

| Standard | Relationship |
|---|---|
| **LangChain Hub / PromptHub** | These are vendor registries. Prompt Provenance is a vendor-neutral document format that any registry can emit. |
| **Model cards** | Model cards describe a model; this spec describes a prompt that targets a model. Companion specs. |
| **Agent Cards** ([agent-cards-spec](https://github.com/mizcausevic-dev/agent-cards-spec)) | Agent cards reference the prompts they use; Prompt Provenance provides the citation target. |
| **AEO Protocol** ([aeo-protocol-spec](https://github.com/mizcausevic-dev/aeo-protocol-spec)) | An `aeo:authoredPrompt` predicate MAY reference a prompt-provenance record. |

## 9. Open questions

- **Multilingual prompts.** Should the content hash apply to a single language version, or to a structured multilingual document?
- **Templates with variable substitution.** Should the hash apply to the template, or to the rendered output for a canonical example set?
- **Prompt chains.** Should a multi-turn chain be one record with N turns, or N linked records?
