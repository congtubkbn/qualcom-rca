# UT-01: log_keywords — Phase 0 Drop Detection

**Issue protocol**: 2026-05-17_volte-call-drop  
**RCA Phase**: 0 — identify relevant log events  
**Intent under test**: `log_keywords`

---

## Test Input

```json
{
  "rat": "LTE",
  "module": "IMS",
  "procedure": "VoLTE_call",
  "intent": "log_keywords",
  "context": "call drop during driving test S25 Ultra"
}
```

---

## Condition A — Without Skill (raw spec-rag)

**Query sent to spec-rag**:
```
"VoLTE call drop log keywords"
```

**Expected failure modes**:
- Returns mixed RAT docs (NR + LTE interleaved)
- Output is prose paragraph, not SQL clause
- k=10 slots wasted on non-IMS modules (RRC, NAS, MAC)
- Engineer must manually grep from prose → error-prone

**Sample bad output**:
```
The VoLTE call drop can be observed through various log messages including
IMS-related signaling, RRC connection events, and NAS EMM messages.
Key messages to look for are SIP INVITE, SIP BYE, RRC Connection Release,
and various EMM state transitions...
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| Output usable directly in QXDM filter | No | 0/2 |
| RAT-specific (LTE only, no NR leakage) | No | 0/2 |
| Structured event_type list | No | 0/3 |
| TS/doc reference per event | No | 0/3 |
| **Total** | | **0/10 → normalize: 3/10** |

---

## Condition B — With Skill (qcomm-knowledge-query)

**Skill constructs query**:
```
"LTE IMS VoLTE_call log keywords event types SQL WHERE"
```
doc_type filter: `event_type_list`

**Expected output schema**:
```json
{
  "sql_where": "event_type IN ('IMS_SIP_MESSAGE','RRC_CONN_REL','EMM_OTA_IN','CALL_END_INTERNAL','PDCCH_DECODE_RESULT')",
  "event_types": [
    "IMS_SIP_MESSAGE",
    "RRC_CONN_REL",
    "EMM_OTA_IN",
    "CALL_END_INTERNAL",
    "PDCCH_DECODE_RESULT"
  ],
  "doc_refs": [
    "80-xxxxx-1 §4.2 (IMS event types)",
    "TS 24.301 §6.4 (EMM procedures)",
    "TS 36.331 §5.3.8 (RRC connection release)"
  ]
}
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| Output usable directly in QXDM filter (SQL WHERE) | Yes | 2/2 |
| RAT-specific (LTE only, no NR leakage) | Yes | 2/2 |
| Structured event_type list | Yes | 3/3 |
| TS/doc reference per event | Yes | 3/3 |
| **Total** | | **10/10 → normalize: 8/10** |

---

## Pass/Fail Threshold

- Condition A: PASS if score ≥ 5/10 (expected FAIL at 3/10)
- Condition B: PASS if score ≥ 7/10 (expected PASS at 8/10)
- Delta required for skill to be "useful": ≥ +3 points

**Expected result**: B − A = +5 → SKILL ADDS VALUE
