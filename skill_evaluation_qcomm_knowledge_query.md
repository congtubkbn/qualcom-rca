# Skill Evaluation — qcomm-knowledge-query

## How the Skill Works

`qcomm-knowledge-query` wraps `spec-rag` with three layers of transformation:

```
LLM input (structured JSON)
    │
    ▼
[1] QUERY CONSTRUCTION
    "qualcomm {rat} {module} {procedure} {intent-suffix}" + optional context
    corpus = "qualcomm" (hardcoded)
    │
    ▼
[2] spec-rag retrieval
    k=10 chunks from cp_qualcomm vector store
    (semantic similarity search against query string)
    │
    ▼
[3] LLM EXTRACTION
    LLM reads 10 raw chunks
    extracts structured output per intent output schema
    returns typed data (SQL clause, cause table, etc.) — not prose
```

**What makes it different from calling spec-rag directly:**
- Input is constrained (intent field → only valid query shapes possible)
- Output is typed (SQL-injectable, table-structured, not paragraph prose)
- Corpus is isolated (cross-vendor contamination prevented)
- Query vocabulary is tuned (QXDM-specific terms activate the right embedding clusters)

---

## Version Comparison

### V0 — Raw spec-rag (no wrapper)

LLM receives user's free-form description → constructs own query → calls spec-rag → reads prose → extracts what it needs.

### V1 — qcomm-knowledge-query (current SKILL.md)

LLM receives structured intent input → constructs deterministic query via suffix map → calls spec-rag → extracts typed output per schema.

### V2 — Improved qcomm-knowledge-query (hypothetical)

V1 architecture + per-intent doc_type filters + split `behavior_compare` + per-intent k values + `known_patterns` with observed sequence + fixed `cross_layer_chain` multi-module query + confidence fields in outputs.

---

## Comparison Table

| Dimension | V0 raw spec-rag | V1 current skill | V2 improved skill |
|-----------|----------------|-----------------|-------------------|
| **Query construction** | Free-form — LLM decides | Deterministic suffix map — 8 intent types | Same as V1 + doc_type filter per intent |
| **Query vocabulary** | LLM may use wrong terms ("handover fail" vs "0xB193") | Intent suffix uses QXDM vocabulary | Same + doc_type narrows embedding space further |
| **Output format** | Free prose — requires downstream parsing | Typed schema per intent (SQL, tables, lists) | Same + completeness flag + confidence score |
| **Corpus isolation** | LLM may omit corpus param → cross-vendor contamination | Hardcoded `corpus=qualcomm` | Configurable corpus — allows cp_common for CP adaptation layer |
| **k value** | LLM decides or default | Fixed k=10 all intents | Per-intent k: crash=5, event_list=15, others=10 |
| **Crash handling** | No special handling — LLM may analyze post-crash logs | `crash_decode` intent — short-circuits analysis | Same + simplified query (no module/procedure required) |
| **Multi-layer issues** | LLM may miss layers in duckDB query | `cross_layer_chain` intent — but single-module query | Multi-module query construction — each layer in embedding query |
| **Pattern matching** | Ad-hoc — depends on LLM memory | `known_patterns` intent — but no observed sequence passed | `known_patterns` requires observed_sequence context → actual signature matching |
| **behavior_compare** | LLM retrieves timers + deviation + NV in one go | Single intent covers Types 5+6+7+9 — k=10 spread thin | Split into 4 intents: `timer_query`, `failure_handling`, `nv_item`, `vendor_deviation` |
| **Token cost per RCA** | High — LLM writes own queries, reads full prose, re-parses | Medium — precise queries, structured output, less re-reading | Similar to V1 — but fewer wasted queries from incorrect output |
| **False bug risk** | High — no enforcement of behavior_compare before conclusion | Medium — `behavior_compare` + `behavior_compare` (Type 9) available | Low — `vendor_deviation` intent mandatory before bug conclusion enforced in schema |
| **Consistency** | Varies session to session — LLM rewrites queries | Consistent — same procedure always generates same query | Same consistency + output completeness flag detects partial extraction |
| **Qualcomm-only** | Depends on LLM discipline | Hardcoded — MTK requires different skill | Corpus parameter — can route to cp_mtk or cp_common |

---

## V0 vs V1 — Concrete Example

**Task:** Get LTE RRC handover event types for duckDB.

**V0 (raw spec-rag):**
```
LLM query (what it might write):
  "LTE RRC handover log events QXDM"   ← vague, no vendor vocabulary
  corpus: (may forget to set)

spec-rag returns: 10 chunks mixing overview text + HO procedure description +
  some event mentions scattered in prose paragraphs.

LLM output:
  "For LTE handover analysis, you should look for events related to
  RRC reconfiguration. The QXDM log typically shows handover commands
  and responses. Key events include HO preparation and execution phases..."
  ← prose, no hex codes, not SQL-injectable
```

**V1 (current skill):**
```
Query (deterministic):
  "qualcomm LTE RRC handover log event type hex codes msg_type list duckDB filter keywords"
  corpus: "qualcomm"

spec-rag returns: 10 chunks from event_type_list sections of QCT TSG docs.

LLM extracts:
  event_type: 0xB193 (LTE RRC OTA), 0xB173 (RACH)
  msg_type: rrcConnectionReconfiguration, rrcConnectionReestablishmentRequest
  WHERE clause: event_type IN ('0xB193','0xB173') OR msg_type IN (...)
  ← SQL-injectable, deterministic format
```

**V2 adds doc_type filter:**
```
Query: same as V1
Filter: doc_type = "event_type_list"   ← 10 chunks ALL from event tables, not mixed
k: 15   ← more coverage for large event lists split across chunks
```

---

## V1 Weaknesses — Current qcomm-knowledge-query

### W1 — `behavior_compare` covers 4 Types with k=10 (retrieval dilution)

`behavior_compare` maps to Types 5+6+7+9: timers, failure handling, NV items, vendor deviation. Each type retrieves from different doc sections. With k=10 shared across all 4, the vector search returns ~2-3 chunks per topic — not enough for complete coverage.

**Effect:** Timer table returned with 2 of 5 timers. NV items missing entirely. LLM reports "no relevant NV items found" when they exist but weren't in top-k.

**Fix in V2:** Split into 4 separate intents, each with its own targeted query and k value.

---

### W2 — `cross_layer_chain` uses single `{module}` in query template

Query construction: `"qualcomm {rat} {module} {procedure} cross-layer trigger chain..."`. For a VoLTE drop analysis, `{module}=IMS` — the embedding pulls IMS-centric chunks. PHY and RRC chunks for the cascade may not rank in top-10.

**Effect:** `cross_layer_chain` output may have complete IMS steps but missing PHY OOS → T310 trigger chain — exactly the part that identifies root cause.

**Fix in V2:** Accept `layers: [PHY, RRC, NAS, IMS]` array. Build query as `"qualcomm {rat} PHY RRC NAS IMS {procedure} cross-layer..."` or run separate sub-queries per layer pair and merge.

---

### W3 — `known_patterns` doesn't pass observed log sequence

Current query: `"qualcomm LTE IMS volte_drop known failure signature patterns root cause log sequence classification"`. This retrieves generic pattern documentation — not matched against what was actually observed.

**Effect:** Pattern library returns all known VoLTE drop patterns. LLM must manually compare each against the log results. This is what the LLM would do without the skill — no acceleration.

**Fix in V2:** Add `observed_sequence` input field (required for this intent):
```json
"observed_sequence": "0xB0C2 BYE at t=14:03:22, 0xB193 rrcConnectionRelease at t=14:03:20, NO 0xB115 OOS"
```
Query becomes: `"... known failure signatures [observed_sequence]"` — vector search anchors to observed events, retrieves matching signatures.

---

### W4 — `crash_decode` requires `{module}` and `{procedure}` (often unknown at crash time)

When CP_CRASH fires, the engineer doesn't know which module or procedure was running. Filling `{module}=unknown` or guessing degrades query embedding. The assert code (`context` field) is all that's available.

**Effect:** Query `"qualcomm LTE unknown unknown assert crash 0x1234..."` → embedding misses the crash decode doc section; returns procedure overview chunks instead.

**Fix in V2:** For `crash_decode` intent, bypass the standard template. Query: `"qualcomm modem assert error code {context} task module meaning known patch"` — no module/procedure fields required.

---

### W5 — No doc_type filter passed to spec-rag

`rag_architecture_deep_dive.md §4.3` states: filtering by `doc_type` is recommended for Types 2, 4, 5, 7. Current skill passes no filter → k=10 mixes `event_type_list` chunks with `procedure` and `deviation` chunks even when only event types are needed.

**Effect:** For `log_keywords` intent, 3 of 10 chunks may be procedure overview text. That's 3 wasted slots. May push relevant event-type chunks outside top-10.

**Fix in V2:** Add doc_type to spec-rag call per intent:
| Intent | doc_type filter |
|--------|----------------|
| `log_keywords` | `event_type_list` |
| `cause_analysis` | `cause_code` |
| `timer_query` (V2 split) | `timer` |
| `nv_item` (V2 split) | `nv_item` |
| `vendor_deviation` (V2 split) | `deviation` |
| `crash_decode` | `crash_decode` |
| `known_patterns` | `failure_pattern` |
| `procedure_flow`, `cross_layer_chain`, `failure_handling` | no filter (broad coverage needed) |

---

### W6 — Query example inconsistency with updated suffix map

Line 80 (query example):
```
"qualcomm NR_NSA RRC scg_failure cause codes failure classification known signatures root cause patterns t310-Expiry"
```
Current suffix map for `cause_analysis`:
```
"cause codes failure classification meaning trigger UE behavior"
```

The example uses the old suffix (before the edit). Query example and suffix map now diverge → LLM reading the skill may use either version inconsistently.

**Fix:** Update line 80 example to use current suffix.

---

### W7 — No output completeness signal

Skill outputs have no field indicating whether extraction was complete. If spec-rag retrieved only 2 of 8 expected event types (because the rest were in unchunked parts of the doc), the output looks complete but is truncated.

**Effect:** LLM builds duckDB WHERE clause with 2 event types instead of 8. Queries return partial log coverage. RCA may miss the failure event.

**Fix in V2:** Add `completeness` field to output schemas:
```markdown
**completeness**: FULL | PARTIAL (reason: [X event types expected, Y retrieved])
```
If PARTIAL → trigger a second query with refined suffix before proceeding to duckDB.

---

### W8 — Corpus hardcoded to qualcomm — CP common layer issues excluded

Samsung CP adaptation layer (factory NV, secril bridge, IMS adaptation) uses `corpus=cp_common`, not `corpus=qualcomm`. Current skill hardcodes qualcomm. CP common layer issues on Qualcomm devices (e.g., factory mode attach fail due to Samsung NV override) cannot be analyzed.

**Fix in V2:** Add optional `corpus` parameter. Default: `qualcomm`. Allow `cp_common` for CP adaptation layer procedures.

---

## Summary Scorecard

| Criterion | V0 raw spec-rag | V1 current | V2 improved |
|-----------|:--------------:|:----------:|:-----------:|
| Query reproducibility | ✗ variable | ✓ deterministic | ✓ deterministic |
| Output SQL-injectable | ✗ prose | ✓ typed schema | ✓ typed + completeness flag |
| Corpus isolation | ✗ depends on LLM | ✓ hardcoded | ✓ + configurable |
| Multi-type coverage (behavior) | ✗ scattered | ✗ diluted k=10 | ✓ split intents |
| Cross-layer query coverage | ✗ single layer | ✗ single module query | ✓ multi-module query |
| Pattern matching accuracy | ✗ no anchoring | ✗ no observed sequence | ✓ sequence-anchored |
| Crash handling | ✗ no short-circuit | ✓ crash_decode intent | ✓ + no module required |
| doc_type filtering | ✗ none | ✗ none | ✓ per-intent filter |
| Token cost | High | Medium | Medium (fewer wasted queries) |
| False bug rate | High | Medium | Low (vendor_deviation enforced) |
| CP common layer support | ✗ | ✗ | ✓ via corpus param |

---

## Recommendation

V1 is a significant improvement over V0 for standard single-layer procedures (attach fail, HO fail, RACH fail). Return value is clear: deterministic SQL output vs prose parsing.

V1 breaks down on complex cases: VoLTE drop (multi-layer), CP crash, pattern-based triage. These are exactly the hard cases where automation value is highest.

V2 changes are targeted and non-breaking — all existing V1 inputs remain valid. Priority order for implementation:
1. Fix W6 (query example inconsistency) — 1 line change, zero risk
2. Fix W5 (doc_type filters) — improves all intents, low implementation cost
3. Fix W4 (crash_decode template) — unblocks crash RCA
4. Fix W3 (known_patterns observed_sequence) — unblocks pattern matching
5. Fix W1 (split behavior_compare) — most complex, highest quality gain
6. Fix W2 (cross_layer_chain multi-module) — needed for VoLTE/SCG RCA accuracy
7. Fix W7 (completeness signal) — quality gate, catches silent failures
8. Fix W8 (corpus parameter) — enables CP common layer analysis
