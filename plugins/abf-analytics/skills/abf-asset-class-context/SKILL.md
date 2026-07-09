---
name: abf-asset-class-context
description: >
  Use when an ABF transaction's asset class is known and you need its KPI catalog, entity model,
  applicability profile, or enum vocabulary. Triggers when list_transactions reveals a transaction's
  asset_class, when an analyst names an asset class (auto loans, credit cards, CRE hospitality, trade
  finance, cell tower, etc.), or asks "what KPIs / stratifications / dimensions apply to X". The
  knowledge is served by the ABF MCP server as resources; this skill is the protocol for resolving
  the asset_class string against the server's index and loading only the sections the question needs.
---

# ABF Asset-Class Context — Gateway

The asset-class catalog is **served by the ABF MCP server** as two resources — a JSON index and a
per-section leaf — read with the `read_resource` tool. This skill is the **protocol** for loading
that knowledge progressively: resolve the transaction's `asset_class` string against the index,
then read only that class's listed section keys. Never load the whole corpus. Never load another
class's sections. Always read from the **same server** abf-gateway selected for data queries —
never mix one environment's catalog with another environment's data.

## When to invoke

Load on ANY of:
- An ABF transaction is identified and its `asset_class` is known (from `list_transactions`).
- The analyst names an asset class ("auto loans report", "hospitality stratifications").
- The analyst asks "what KPIs / metrics / dimensions / data apply to {asset_class}".
- The `abf-report-runner` requests catalog injection.

Skip when:
- This conversation already loaded the catalog for the same asset class — reuse it.
- No transaction is active AND no asset class has been named — wait.
- The analyst asks about a non-ABF topic.

## Where the knowledge lives

Two resource URIs, read via `read_resource`. Do not use `list_resources` to discover sections —
the index is the only enumerator and its URI is fixed.

- `securitizations://asset-class-knowledge` — the **JSON index**, the entry point. Carries
  `always_load_global[]`, `asset_classes[]` (each with `label`, `synonyms`, `family`,
  `cf_determinism`, `loan_sim_fit`, `always_load[]`, `on_demand{}`), and `families[]`.
- `securitizations://asset-class-knowledge/{section}` — one section body. `{section}` keys are
  handed out by the index; never construct or guess one.

## Protocol

### 1. Read the index (once per session)

`read_resource("securitizations://asset-class-knowledge")`. Cache it in the conversation and
reuse it for every later class, transaction, or comparison — never re-read it in the same session.

### 2. Resolve exactly one asset class

Source the `asset_class` string from the active transaction (`list_transactions`; the valid enum
is `securitizations://asset-classes`) or the analyst's phrasing. Match against `asset_classes[]`:
exact `label` match → `synonyms` match → close-variant match. When no confident match, read
`securitizations://asset-class-knowledge/_global.classification` and apply its fallback heuristics
(entity-shape evidence). Still unresolved → load nothing further, surface the closest candidate
labels, and ask the analyst to pick. Never guess.

### 3. Load the resolved class — one parallel batch, and only it

Issue ONE parallel batch of `read_resource` calls: every key in `always_load_global[]` plus every
key in the resolved entry's `always_load[]`. **Never read a key listed under a different class.**
The entry itself supplies the `cf_determinism` / `loan_sim_fit` profile flags — no extra read.

### 4. Build the catalog

Assemble `{asset_class_catalog}` in memory, keyed on `(transaction_id, asset_class_label)`:
`profile` (the entry's flags), `risk_metrics[]` / `stratifications[]` / enum vocabulary merged
from the loaded sections, and the entry's `on_demand{}` map **verbatim** — downstream consumers
(`abf-report-runner`) read gated sections through this map and never touch the index themselves.

### 5. Load on-demand sections only when the question needs them

Read via the entry's `on_demand` keys — never speculatively:

- `data_model` — data-availability / "what fields exist" / field-grounding questions.
- `loss_vintage` — loss-curve / cohort-default questions. `null` means the methodology does not
  apply to this class — say so; do not invent a curve.
- `prepayment` — prepayment-speed / adverse-selection questions. `null` means the standard
  CPR/SMM model does not apply — name the class's own proxy instead.
- `concentration` — geographic / FICO / obligor / sector concentration.

The gates are pre-computed server-side: a `null` value IS the applicability answer.

### 6. Family-compare guard

When comparing multiple sub-classes **within one family**, read ONLY that family's `index` and
`common_kpis` keys from `families[]`. Do not read the individual classes' sections.

### 7. Override rule

When the class's own kpis section and its family common-kpis section define a KPI with the same
`### name`, the class-specific one wins.

## Failure handling

- **Unknown-section error** from `read_resource` → re-read the index once and use a key it lists.
  Never retry a constructed key.
- **Index read fails** (resource not published on this server) → tell the analyst asset-class
  knowledge is not available on this environment, then continue without a catalog: no
  applicability gates, generic analysis only. Never fabricate catalog content from priors.

## Output contract — never leak internal routing

Section keys, resource URIs, family names, tier names, and the index structure are **internal
routing metadata**. Surface NONE of them.

- Show ONE asset-class label — the index `label` ("Auto Loans", "CRE Hospitality", "Cell Tower").
  Never a section key, a family name, or a `securitizations://` URI.
- KPI / enum counts and dimension lists are merged across loaded sections — do not split by tier
  or say which section a metric came from.
- Never say "inherits from", "tier", "common", "section", or expose a key or URI.
- "Where does this metric come from?" → "Cardo standard catalog."
- Surface `cf_determinism` / `loan_sim_fit` only as plain applicability statements (e.g. "Credit
  Cards uses revolving master-trust dynamics, so standard CPR/CDR/vintage models don't apply —
  use Monthly Payment Rate framing instead").

## Authority boundary

The catalog defines what is **relevant** for an asset class; it is not a calculation authority.
Ground every number in live MCP data (`get_stratification_analytics_data`). A catalog
`computation` string is a framing reference, not a substitute for the server's computed value.

## Never

- Resolve to more than one asset class, or read a second class's sections after resolving one.
- Read a section key the index did not hand out for the resolved class.
- Load on-demand sections speculatively.
- Surface section keys, URIs, family/tier names, or "inherits from" to the analyst.
- Present a catalog `computation` as an observed value.
- Call `list_resources` to enumerate knowledge sections — the index is the only enumerator.
- Read knowledge from a different MCP server than the one answering the session's data queries.
