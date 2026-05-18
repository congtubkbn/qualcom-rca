---
name: qcomm-knowledge-query
description: >
  Query indexed Qualcomm vendor documentation via the spec-rag skill to retrieve
  structured knowledge for RCA analysis. Use this skill whenever the chipset is
  confirmed as Qualcomm AND the analysis needs any of: QXDM log event types or
  hex codes, procedure state machine steps, cause code tables, field decode
  information, vendor-specific timer/NV item values, failure handling behavior,
  or vendor deviation from 3GPP spec. Trigger BEFORE building duckDB queries
  (need hex codes), DURING analysis (need cause interpretation), and AT root cause
  step (need vendor behavior reference). Do NOT use for 3GPP spec lookups (use
  3gpp-spec skill), and never use for Exynos chipsets (use semantic_code_search).
---

# Qualcomm Knowledge Query

Structured wrapper over the `spec-rag` skill for Qualcomm RCA. Converts a
structured intent request into a precisely-worded spec-rag query with doc_type
filter, then formats the output into a schema the RCA pipeline can consume directly.

## Why This Exists

Using spec-rag directly produces free-form prose optimized for humans. The RCA
pipeline needs typed data: SQL filters, cause code tables, JSON field paths.
This skill closes that gap via four mechanisms:

1. **Query precision** - QXDM doc sections use specific vocabulary (`0xB`,
   `event_type`, `DIAG`, `subsys`). Intent-specific query suffixes activate the
   right embeddings rather than retrieving overview sections.

2. **Intent disambiguation** - the same procedure (`LTE RRC handover`) has eleven
   distinct information needs. One free-form query cannot retrieve all correctly.
   The intent field selects the right suffix, doc_type filter, and k value.

3. **Corpus isolation** - `corpus=qualcomm` prevents MTK MDLOG string names from
   contaminating output. Qualcomm hex format (`0xBxxx`) is vendor-unique.

4. **doc_type filtering** - each intent filters by document category, ensuring
   k chunks come from the right doc section (event tables, not procedure prose).

## Input Schema

```json
{
  "rat":               "LTE | NR_NSA | NR_SA | WCDMA",
  "module":            "RRC | NAS | IMS | PHY | MAC | RLC | PDCP",
  "procedure":         "handover | attach | volte_drop | scg_failure | rlf | registration | plmn_search | sms | rach | ...",
  "intent":            "log_keywords | procedure_flow | cause_analysis | field_decode | cross_layer_chain | crash_decode | known_patterns | timer_query | failure_handling | nv_item | vendor_deviation",
  "context":           "(see per-intent requirements below)",
  "layers":            "(cross_layer_chain only) [PHY, RRC, NAS, IMS] — array of layers in the cascade",
  "observed_sequence": "(known_patterns only) comma-separated observed log events, e.g. '0xB0C2 BYE, 0xB193 rrcConnectionRelease-2s-before, NO 0xB115'",
  "corpus":            "(optional) qualcomm | cp_common — default: qualcomm; use cp_common for Samsung CP adaptation layer issues"
}
```

### `context` field requirements by intent

| Intent | context | Required? |
|--------|---------|-----------|
| `crash_decode` | assert/error code from CP_CRASH or MD_EX | **Required** |
| `field_decode` | field name from raw log JSON | **Required** |
| `cause_analysis` | specific cause code value | **Required** |
| `known_patterns` | use `observed_sequence` field instead | N/A |
| all others | additional qualifier (timer name, NV item number, etc.) | Optional |

---

## Query Construction

### Standard Template (all intents except crash_decode)

```
query  = "{corpus} {rat} {module} {procedure} {suffix}"
         if context provided: query += " {context}"
         if cross_layer_chain AND layers provided: replace {module} with layers.join(" ")
corpus = input.corpus ?? "qualcomm"
filter = doc_type per intent (see suffix map)
k      = per-intent value (see suffix map)
```

### Special Template — `crash_decode`

Bypass standard template entirely. Module and procedure are usually unknown at crash time.

```
query  = "{corpus} modem assert error {context} task module meaning known issue patch"
corpus = input.corpus ?? "qualcomm"
filter = doc_type: "crash_decode"
k      = 5
```

### Special Template — `cross_layer_chain`

If `layers` array provided, expand to cover all layers in query:

```
query  = "{corpus} {rat} {layers.join(' ')} {procedure} cross-layer trigger chain cascade duckDB coverage"
         e.g. layers=[PHY, RRC, NAS] → "qualcomm LTE PHY RRC NAS volte_drop cross-layer trigger chain..."
filter = none (broad coverage required)
k      = 10
```

### Intent → Suffix Map

| Intent | Type | Suffix | doc_type filter | k | When to Use |
|--------|------|--------|----------------|---|-------------|
| `log_keywords` | 2 | `"log event type hex codes msg_type list duckDB filter keywords"` | `event_type_list` | 15 | Phase 1 — before first duckDB query |
| `procedure_flow` | 1 | `"procedure state machine sequence steps expected behavior flow"` | `procedure` | 10 | Phase 1 — establish expected baseline |
| `cause_analysis` | 4 | `"cause code meaning trigger UE behavior network response"` | `cause_code` | 8 | Phase 3 — context=cause code required |
| `field_decode` | 3 | `"field meaning interpretation raw log JSON values decode"` | none | 5 | Phase 3 — context=field name required |
| `cross_layer_chain` | 8 | `"cross-layer trigger chain source layer event target layer cascade effect duckDB coverage"` | none | 10 | Phase 1 multi-layer — use layers array |
| `crash_decode` | 10 | *(see special template)* | `crash_decode` | 5 | Phase 0 — CP_CRASH/MD_EX only; context=assert code |
| `known_patterns` | 11 | `"known failure signature patterns root cause log sequence classification"` | `failure_pattern` | 10 | Phase 2 — after first duckDB; use observed_sequence |
| `timer_query` | 5 | `"timer value default range expiry behavior start stop condition"` | `timer` | 8 | Phase 4 — when timeout suspected; context=timer name |
| `failure_handling` | 6 | `"failure handling recovery action retry fallback behavior"` | `failure_handling` | 8 | Phase 4 — after failure event identified |
| `nv_item` | 7 | `"NV item configuration effect default value behavior"` | `nv_item` | 8 | Phase 4 — when config error suspected; context=NV name or number |
| `vendor_deviation` | 9 | `"vendor specific behavior deviation 3GPP spec intentional implementation"` | `deviation` | 8 | Phase 4 — mandatory before any bug conclusion |

### Query Examples

```
{rat:LTE, module:RRC, procedure:handover, intent:log_keywords}
-> query:  "qualcomm LTE RRC handover log event type hex codes msg_type list duckDB filter keywords"
-> filter: doc_type=event_type_list
-> k:      15

{rat:NR_NSA, module:RRC, procedure:scg_failure, intent:cause_analysis, context:"t310-Expiry"}
-> query:  "qualcomm NR_NSA RRC scg_failure cause code meaning trigger UE behavior network response t310-Expiry"
-> filter: doc_type=cause_code
-> k:      8

{rat:LTE, module:IMS, procedure:volte_drop, intent:field_decode, context:"q850_cause"}
-> query:  "qualcomm LTE IMS volte_drop field meaning interpretation raw log JSON values decode q850_cause"
-> filter: none
-> k:      5

{rat:LTE, procedure:volte_drop, intent:cross_layer_chain, layers:[PHY, RRC, NAS, IMS]}
-> query:  "qualcomm LTE PHY RRC NAS IMS volte_drop cross-layer trigger chain cascade duckDB coverage"
-> filter: none
-> k:      10

{intent:crash_decode, context:"0x1234ABCD"}
-> query:  "qualcomm modem assert error 0x1234ABCD task module meaning known issue patch"
-> filter: doc_type=crash_decode
-> k:      5

{rat:LTE, module:IMS, procedure:volte_drop, intent:known_patterns,
 observed_sequence:"0xB0C2 BYE at 14:03:22, 0xB193 rrcConnectionRelease at 14:03:20, NO 0xB115 before release"}
-> query:  "qualcomm LTE IMS volte_drop known failure signature patterns root cause log sequence 0xB0C2 BYE 0xB193 rrcConnectionRelease NO 0xB115"
-> filter: doc_type=failure_pattern
-> k:      10

{rat:LTE, module:RRC, procedure:rlf, intent:timer_query, context:"T310"}
-> query:  "qualcomm LTE RRC rlf timer value default range expiry behavior start stop condition T310"
-> filter: doc_type=timer
-> k:      8

{rat:LTE, module:RRC, procedure:handover, intent:vendor_deviation}
-> query:  "qualcomm LTE RRC handover vendor specific behavior deviation 3GPP spec intentional implementation"
-> filter: doc_type=deviation
-> k:      8
```

---

## Calling spec-rag

Invoke spec-rag with:
```
spec-rag(
  query  = <constructed query>,
  corpus = <input.corpus ?? "qualcomm">,
  filter = {"doc_type": "<per-intent value>"},   ← omit filter field if doc_type=none
  k      = <per-intent value>
)
```

Parse returned chunks. Extract structured output per the intent output schema below.
Do not return raw prose — always reformat into the structured template.
If extraction is incomplete, set `completeness: PARTIAL` and state the reason.

---

## Output Schemas

### `log_keywords`

```markdown
## Log Keywords - Qualcomm {RAT} {MODULE} {PROCEDURE}

**completeness**: FULL | PARTIAL (reason if partial)

**event_type** (QXDM hex):
- 0xBXXX - [description]
- 0xBXXX - [description]

**msg_type**:
- [rrcMessageName]
- [rrcMessageName]

**raw JSON fields** (json_extract_string paths):
- $.fieldPath.subField - [meaning]

**duckDB WHERE clause**:
```sql
AND (event_type IN ('0xBXXX', '0xBXXX')
     OR msg_type IN ('[msgType1]', '[msgType2]'))
```
```

### `procedure_flow`

```markdown
## Procedure Flow - Qualcomm {RAT} {MODULE} {PROCEDURE}

**completeness**: FULL | PARTIAL (reason if partial)

**Expected Sequence**:
1. [Message/Event] - [UE or NW action]
2. [Message/Event] - [UE or NW action]
...

**Failure Exit Points**:
- Step N: [condition] -> [failure path / timer expiry]
```

### `cause_analysis`

```markdown
## Cause Analysis - {context}

**completeness**: FULL | PARTIAL (reason if partial)

**Cause Code**: {context}
**Field**: [field name in log]
**Meaning**: [human-readable description]
**Trigger**: [what network or UE condition causes this]
**UE Behavior**: [what UE does after receiving this cause]
**Network Behavior**: [what network does]
**RCA Implication**: [how to interpret in failure context]
```

### `field_decode`

```markdown
## Field Decode - {context}

**Field**: {context}
**JSON path**: $.{path}
**Type / Range**: [type and valid range]
**Meaning**: [what this field indicates]
**RCA Relevance**: [how this field relates to the failure being analyzed]
```

### `cross_layer_chain`

```markdown
## Cross-Layer Chain - Qualcomm {RAT} {LAYERS} {PROCEDURE}

**completeness**: FULL | PARTIAL (layers covered: [list] / layers missing: [list])

**Trigger Chain**:
| Step | Layer | Event | Triggers |
|------|-------|-------|----------|
| 1 | [PHY/MAC/RRC/NAS/IMS] | [event or state] | [next layer trigger] |
| 2 | ... | ... | ... |

**duckDB Coverage** (all layers — combine into single WHERE clause):
```sql
AND (event_type IN ('[PHY_hex]', '[RRC_hex]', '[NAS_hex]', '[IMS_hex]')
     OR msg_type IN ('[rrc_msg]', '[nas_msg]', '[ims_msg]'))
```
```

### `crash_decode`

```markdown
## Crash Decode - {context}

**Assert Code**: {context}
**Task**: [task/thread name]
**Module**: [module name]
**Meaning**: [what this crash indicates]
**Trigger Condition**: [what causes this assert]
**Known Issue**: [yes — patch: [ref] | no]
**RCA Action**: All log events AFTER crash_timestamp are invalid. Analyze crash cause only.
```

### `known_patterns`

```markdown
## Known Failure Patterns - Qualcomm {RAT} {MODULE} {PROCEDURE}

**Observed sequence**: {observed_sequence}

**Matched Patterns**:
| Pattern Name | Match Confidence | Log Signature | Root Cause | Next Step |
|--------------|-----------------|--------------|------------|-----------|
| [name] | HIGH/MEDIUM/LOW | [signature vs observed] | [cause] | [action] |

**No Match**: proceed to full manual analysis (cause_analysis + failure_handling + vendor_deviation)
```

### `timer_query`

```markdown
## Timer Reference - Qualcomm {RAT} {MODULE} {context}

**completeness**: FULL | PARTIAL (reason if partial)

**Timer**: {context}
**Standard ref**: [TS XX.XXX §Y.Y]
**Default value**: [value + unit]
**Configurable range**: [values]
**Configured by**: [network IE / NV item]
**Start condition**: [what starts the timer]
**Stop conditions**: [what stops it]
**Expiry behavior**: [what happens on expiry]
**Vendor note**: [Qualcomm-specific default or NV override if any]
```

### `failure_handling`

```markdown
## Failure Handling - Qualcomm {RAT} {MODULE} {PROCEDURE}

**completeness**: FULL | PARTIAL (reason if partial)

**Failure event**: [what triggers this handling]
**Expected action**:
1. [step 1]
2. [step 2]
...
**Retry behavior**: [automatic retry logic if any]
**Fallback**: [what happens if recovery fails]
**Vendor specific**: [Qualcomm implementation detail vs spec if any]
```

### `nv_item`

```markdown
## NV Item Reference - Qualcomm {MODULE} {context}

**completeness**: FULL | PARTIAL (reason if partial)

**NV Item**: {context}
**Name**: [full NV name]
**Module**: [affected module]
**Default value**: [Qualcomm default]
**Spec default**: [3GPP spec default if applicable]
**Effect**: [what this NV controls]
**Allowed range**: [min–max]
**RCA implication**: [how misconfiguration causes failures]
**How to read**: [QXDM command or QCAT path]
```

### `vendor_deviation`

```markdown
## Vendor Deviation - Qualcomm {RAT} {MODULE} {PROCEDURE}

**completeness**: FULL | PARTIAL (reason if partial)

**Spec requirement**:
- Reference: [TS XX.XXX §Y.Y]
- Text: [exact spec wording]

**Qualcomm implementation**:
- Behavior: [what Qualcomm actually does]
- Intentional: yes | no
- Reason: [why Qualcomm deviates if intentional]
- Applies to: [all QC | specific series | NV-overridable]

**RCA implication**:
- [observed behavior] is NOT a bug IF [condition]
- [observed behavior] IS a deviation IF [condition]
```

---

## RCA Phase Guidance

| RCA Phase | Intent(s) to Use |
|-----------|-----------------|
| PHASE 0 — Crash detected (CP_CRASH / MD_EX) | `crash_decode` — short-circuits all other phases |
| PHASE 1 — Reproduce failure (single-layer) | `log_keywords` + `procedure_flow` |
| PHASE 1 — Reproduce failure (multi-layer: VoLTE, SCG, RLF) | `log_keywords` + `procedure_flow` + `cross_layer_chain` |
| PHASE 2 — Classify cause | `known_patterns` (with observed_sequence) → if no match → `cause_analysis` |
| PHASE 3 — Decode unknown fields | `field_decode` |
| PHASE 4 — Confirm root cause vs spec | `failure_handling` + `vendor_deviation` (mandatory) |
| PHASE 4 — Timer suspected | + `timer_query` (context=timer name) |
| PHASE 4 — Config error suspected | + `nv_item` (context=NV name or number) |

When calling multiple intents for the same procedure in one phase, construct
separate spec-rag queries per intent — do not combine intents into one query.

**Phase 4 rule**: `vendor_deviation` is mandatory before any bug conclusion.
Observed behavior that differs from 3GPP spec may be intentional Qualcomm implementation.

---

## Error Handling

- If `log_keywords` returns fewer than 3 hex codes (`0xBxxx` format): query too generic.
  Refine by adding specific sub-procedure (e.g. `handover_failure` instead of `handover`).
  Also verify `completeness` field — PARTIAL means relevant chunks were outside top-15.

- If `log_keywords` completeness is PARTIAL after refine: run a second `log_keywords`
  query with a different sub-procedure suffix to recover missing event types.

- If output contains MTK-format event names (e.g. `LTE_RRC_OTA_PACKET`, `MD_EX`):
  corpus isolation failed. Verify spec-rag call used `corpus=qualcomm` and retry.

- If `crash_decode` returns no match: assert code not indexed.
  Note "unknown assert [code]" in RCA report. Do NOT analyze post-crash logs.
  Escalate to Qualcomm TSG with full crash dump.

- If `cross_layer_chain` completeness shows missing layers: run separate
  `log_keywords` per missing layer to fill the gap. Flag incomplete coverage in report.

- If `known_patterns` returns no match and observed_sequence was provided:
  proceed to manual analysis — cause_analysis + failure_handling + vendor_deviation.
  If observed_sequence was NOT provided: retry with observed_sequence before giving up.

- If `vendor_deviation` completeness is PARTIAL (deviation not documented):
  do not assume observed behavior is a bug. Note "deviation undocumented" in report.
  Escalate to Qualcomm TSG for confirmation before filing defect.

- If any intent returns no relevant chunks: procedure may not be indexed in corpus.
  Fall back to `3gpp-spec` for expected behavior. Note the RAG gap in report.
  Consider using `corpus=cp_common` if issue is in Samsung CP adaptation layer.
