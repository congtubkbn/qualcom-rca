# UT-02: cause_analysis — Phase 2 Cause Classification

**Issue protocol**: 2026-05-17_volte-call-drop  
**RCA Phase**: 2 — classify root cause layer  
**Intent under test**: `cause_analysis`

---

## Test Input

```json
{
  "rat": "LTE",
  "module": "RRC",
  "procedure": "handover",
  "intent": "cause_analysis",
  "context": "VoLTE call drop during mobility, RRCConnectionRelease received, cause unknown"
}
```

---

## Condition A — Without Skill (raw spec-rag)

**Query sent to spec-rag**:
```
"VoLTE call drop cause LTE handover"
```

**Expected failure modes**:
- Returns generic 3GPP prose about release causes
- No cross-layer cause mapping (IMS ↔ RRC ↔ NAS)
- No Qualcomm-specific cause codes
- Engineer must manually build cause hypothesis table

**Sample bad output**:
```
VoLTE call drops can occur due to various causes. Radio Link Failure (RLF)
is triggered when the radio conditions deteriorate. The UE sends
RRCConnectionReestablishmentRequest after detecting RLF. IMS may also
generate a BYE message when the call is terminated...
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| Cause table structured (name, trigger, layer) | No | 0/3 |
| Cross-layer mapping (IMS ↔ RRC ↔ NAS) | No | 0/3 |
| Qualcomm-specific cause codes included | No | 0/2 |
| TS reference per cause | Partial | 1/2 |
| **Total** | | **1/10 → normalize: 4/10** |

---

## Condition B — With Skill (qcomm-knowledge-query)

**Skill constructs query**:
```
"LTE RRC handover cause_analysis cause codes failure reason"
```
doc_type filter: `cause_code`

**Expected output schema**:
```json
{
  "cause_table": [
    {
      "cause_name": "radioLinkFailure",
      "trigger": "T310 expiry after N310 OOS indications",
      "layer": "RRC",
      "ts_ref": "TS 36.331 §5.3.11"
    },
    {
      "cause_name": "handoverFailure",
      "trigger": "HO_FAIL_IND from target eNB or T304 expiry",
      "layer": "RRC",
      "ts_ref": "TS 36.331 §5.4.3"
    },
    {
      "cause_name": "imsCauseQ850_16",
      "trigger": "SIP BYE with Reason: Q.850;cause=16 (Normal Call Clearing)",
      "layer": "IMS",
      "ts_ref": "TS 24.229 §24.4"
    },
    {
      "cause_name": "emmCause_15",
      "trigger": "NAS detach or bearer modification failure",
      "layer": "NAS",
      "ts_ref": "TS 24.301 §9.9.3.9"
    }
  ]
}
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| Cause table structured (name, trigger, layer) | Yes | 3/3 |
| Cross-layer mapping (IMS ↔ RRC ↔ NAS) | Yes | 3/3 |
| Qualcomm-specific cause codes included | Yes | 2/2 |
| TS reference per cause | Yes | 2/2 |
| **Total** | | **10/10 → normalize: 9/10** |

---

## Pass/Fail Threshold

- Condition A: PASS if score ≥ 5/10 (expected FAIL at 4/10)
- Condition B: PASS if score ≥ 7/10 (expected PASS at 9/10)
- Delta required: ≥ +3

**Expected result**: B − A = +5 → SKILL ADDS VALUE
