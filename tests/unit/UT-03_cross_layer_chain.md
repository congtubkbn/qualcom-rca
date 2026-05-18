# UT-03: cross_layer_chain — Phase 2 Multi-Layer Event Chain

**Issue protocol**: 2026-05-17_volte-call-drop  
**RCA Phase**: 2 — trace failure across protocol layers  
**Intent under test**: `cross_layer_chain`  
**Known V1 weakness**: W2 — single-module query misses IMS, NAS, MAC layers

---

## Test Input

```json
{
  "rat": "LTE",
  "module": "RRC",
  "procedure": "VoLTE_call_drop",
  "intent": "cross_layer_chain",
  "context": "IMS SIP BYE → RRC release → MAC HARQ failure during HO"
}
```

---

## Condition A — Without Skill (raw spec-rag)

**Query sent to spec-rag**:
```
"IMS RRC MAC call drop chain LTE handover failure"
```

**Expected failure modes**:
- Single unfiltered query: k=10 results spread randomly across layers
- No temporal ordering of events
- No parent-child causality (which layer triggered which)
- Engineer must manually correlate timestamps across 3-4 log layers

**Sample bad output**:
```
The call drop during handover involves multiple protocol layers.
At the IMS layer, SIP BYE may be triggered. At the RRC layer,
RRCConnectionRelease can occur. MAC layer may show HARQ failures
indicating poor radio conditions...
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| IMS layer covered | Maybe | 1/2 |
| RRC layer covered | Maybe | 1/2 |
| MAC/PHY layer covered | No | 0/2 |
| Events ordered with causality arrows | No | 0/2 |
| Missing-layer indicator present | No | 0/2 |
| **Total** | | **2/10 → normalize: 2/10** |

---

## Condition B1 — With Skill V1 (qcomm-knowledge-query, known degraded)

**Skill constructs query** (single module — V1 weakness W2):
```
"LTE RRC VoLTE_call_drop cross layer chain event sequence"
```

**Expected output (V1 — RRC-only, degraded)**:
```json
{
  "chain": [
    {"step": 1, "layer": "RRC", "event": "T310_expiry", "trigger": "PDCCH loss N310 times", "ts_ref": "TS 36.331 §5.3.11"},
    {"step": 2, "layer": "RRC", "event": "RRCConnectionRelease", "trigger": "eNB decision post-RLF", "ts_ref": "TS 36.331 §5.3.8"}
  ],
  "missing_layers": ["IMS", "NAS", "MAC", "PHY"]
}
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| IMS layer covered | No | 0/2 |
| RRC layer covered | Yes | 2/2 |
| MAC/PHY layer covered | No | 0/2 |
| Events ordered with causality arrows | Partial | 1/2 |
| Missing-layer indicator present | Yes | 2/2 |
| **Total** | | **5/10** |

---

## Condition B2 — With Skill V2 (proposed fix: parallel per-layer queries)

**V2 fix**: splits into 3 parallel spec-rag calls:
- Query 1: `"LTE IMS VoLTE_call_drop cross layer chain"`
- Query 2: `"LTE RRC VoLTE_call_drop cross layer chain"`
- Query 3: `"LTE MAC PHY VoLTE_call_drop cross layer chain"`

Then merges + orders by event timestamp anchors.

**Expected output (V2 — full chain)**:
```json
{
  "chain": [
    {"step": 1, "layer": "PHY", "event": "PDCCH_DECODE_FAIL", "trigger": "weak signal during HO gap", "ts_ref": "TS 36.213"},
    {"step": 2, "layer": "RRC", "event": "OOS_indication_N310", "trigger": "PHY reports OOS", "ts_ref": "TS 36.331 §5.3.11"},
    {"step": 3, "layer": "RRC", "event": "T310_start", "trigger": "N310 OOS count reached", "ts_ref": "TS 36.331 §5.3.11"},
    {"step": 4, "layer": "RRC", "event": "T310_expiry_RLF", "trigger": "T310 expired, no IS indication", "ts_ref": "TS 36.331 §5.3.11"},
    {"step": 5, "layer": "IMS", "event": "IMS_local_release", "trigger": "RRC reports RLF to IMS", "ts_ref": "TS 24.229 §5.1.6"},
    {"step": 6, "layer": "IMS", "event": "SIP_BYE_sent", "trigger": "IMS local release", "ts_ref": "TS 24.229"}
  ],
  "missing_layers": []
}
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| IMS layer covered | Yes | 2/2 |
| RRC layer covered | Yes | 2/2 |
| MAC/PHY layer covered | Yes | 2/2 |
| Events ordered with causality arrows | Yes | 2/2 |
| Missing-layer indicator present | Yes (empty = complete) | 2/2 |
| **Total** | | **10/10 → normalize: 9/10** |

---

## Comparison Table

| Condition | Score /10 | Multi-layer | Ordered |
|-----------|-----------|-------------|---------|
| A: Without skill | 2 | No | No |
| B1: Skill V1 | 5 | Partial | Partial |
| B2: Skill V2 | 9 | Yes | Yes |

## Pass/Fail Threshold

- Condition A: PASS if ≥ 5 (expected FAIL at 2)
- Condition B1: PASS if ≥ 7 (expected FAIL at 5 — **V1 weakness confirmed**)
- Condition B2: PASS if ≥ 7 (expected PASS at 9)

**Action**: UT-03 B1 failure confirms need for V2 upgrade (W2 fix priority = HIGH).
