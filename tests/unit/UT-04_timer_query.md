# UT-04: timer_query — Phase 3 Timer Decode

**Issue protocol**: 2026-05-17_volte-call-drop  
**RCA Phase**: 3 — decode specific fields/timers  
**Intent under test**: `timer_query`

---

## Test Input

```json
{
  "rat": "LTE",
  "module": "RRC",
  "procedure": "RLF_detection",
  "intent": "timer_query",
  "context": "T310 timer role during handover attempt, driving test weak coverage"
}
```

---

## Condition A — Without Skill (raw spec-rag)

**Query sent to spec-rag**:
```
"T310 timer LTE RLF"
```

**Expected failure modes**:
- May return NR T310 (TS 38.331) mixed with LTE T310 (TS 36.331) — same name, different spec
- Output is prose paragraph, no structured start/stop conditions
- No expiry action chain documented
- No NV configurability info (Qualcomm-specific)

**Sample bad output**:
```
T310 is a timer used in LTE for Radio Link Failure detection. It is started
when the UE detects out-of-sync conditions from the physical layer. The timer
value is configurable by the network. When T310 expires, the UE considers
that a Radio Link Failure has occurred...
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| RAT-correct spec (LTE TS 36.331, not NR) | 50% | 1/2 |
| start_condition explicit and structured | No | 0/2 |
| stop_condition explicit and structured | No | 0/2 |
| expiry_action with next procedure | No | 0/2 |
| NV configurability (Qualcomm) noted | No | 0/2 |
| **Total** | | **1/10 → normalize: 4/10** |

---

## Condition B — With Skill (qcomm-knowledge-query)

**Skill constructs query**:
```
"LTE RRC RLF_detection timer query start stop expiry"
```
doc_type filter: `timer`

**Expected output schema**:
```json
{
  "timers": [
    {
      "name": "T310",
      "rat": "LTE",
      "start_condition": "N310 consecutive out-of-sync indications received from PHY layer",
      "stop_condition": "N311 consecutive in-sync indications, OR handover command received, OR RRCConnectionRelease received",
      "expiry_action": "Declare RLF → initiate RRC connection re-establishment (RRCConnectionReestablishmentRequest)",
      "default_value_ms": 1000,
      "configurable_by_nw": true,
      "configurable_by_nv": true,
      "nv_item": "NV_LTE_RRC_T310_VALUE_I",
      "ts_ref": "TS 36.331 §5.3.11, Table 7.3-1"
    },
    {
      "name": "N310",
      "rat": "LTE",
      "description": "Max consecutive OOS indications before starting T310",
      "default_value": 1,
      "ts_ref": "TS 36.331 §7.3"
    },
    {
      "name": "N311",
      "rat": "LTE",
      "description": "Consecutive IS indications needed to stop T310",
      "default_value": 1,
      "ts_ref": "TS 36.331 §7.3"
    }
  ]
}
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| RAT-correct spec (LTE TS 36.331, not NR) | Yes | 2/2 |
| start_condition explicit and structured | Yes | 2/2 |
| stop_condition explicit and structured | Yes | 2/2 |
| expiry_action with next procedure | Yes | 2/2 |
| NV configurability (Qualcomm) noted | Yes | 2/2 |
| **Total** | | **10/10 → normalize: 9/10** |

---

## Pass/Fail Threshold

- Condition A: PASS if ≥ 5 (expected FAIL at 4)
- Condition B: PASS if ≥ 7 (expected PASS at 9)
- Delta required: ≥ +3

**Expected result**: B − A = +5 → SKILL ADDS VALUE

---

## Ground Truth Verification

Timer values verifiable against TS 36.331 Table 7.3-1:
- T310 default: {ms0, ms50, ms100, ms200, ms500, ms1000, ms2000} — UE configured via SIB2
- N310 default: {n1, n2, n4, n6, n8, n10, n20}
- N311 default: {n1, n2, n3, n4, n5, n6, n8, n10}
