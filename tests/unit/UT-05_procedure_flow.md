# UT-05: procedure_flow — Phase 4 Spec Conformance Check

**Issue protocol**: 2026-05-17_volte-call-drop  
**RCA Phase**: 4 — compare observed vs expected 3GPP flow  
**Intent under test**: `procedure_flow`

---

## Test Input

```json
{
  "rat": "LTE",
  "module": "IMS",
  "procedure": "VoLTE_call_drop",
  "intent": "procedure_flow",
  "context": "MO VoLTE call drop due to RLF during driving, expected vs observed sequence"
}
```

---

## Condition A — Without Skill (raw spec-rag)

**Query sent to spec-rag**:
```
"VoLTE call drop procedure flow 3GPP LTE"
```

**Expected failure modes**:
- Returns generic VoLTE call setup flow, not drop-specific flow
- MT and MO flows mixed in same result
- No step-number anchors for log timestamp correlation
- Engineer must manually identify where observed log deviates from spec

**Sample bad output**:
```
VoLTE call procedure involves SIP signaling over IMS. When a call drop occurs,
the UE may send a BYE message to terminate the session. The network responds
with 200 OK. In case of Radio Link Failure, the IMS session may be terminated
locally before the SIP BYE is sent...
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| Drop-specific flow (not generic setup) | No | 0/2 |
| MO-specific (not mixed MT/MO) | No | 0/2 |
| Step-numbered for log correlation | No | 0/2 |
| Entity labels (UE, eNB, IMS_Core, P-CSCF) | No | 0/2 |
| TS reference per step | No | 0/2 |
| **Total** | | **0/10 → normalize: 3/10** |

---

## Condition B — With Skill (qcomm-knowledge-query)

**Skill constructs query**:
```
"LTE IMS VoLTE_call_drop procedure flow step sequence"
```

**Expected output schema**:
```json
{
  "flow_type": "MO_VoLTE_call_drop_via_RLF",
  "flow": [
    {
      "step": 1,
      "entity": "PHY→RRC",
      "message": "OOS indication (PDCCH decode failure)",
      "condition": "Repeated N310 times",
      "ts_ref": "TS 36.213, TS 36.331 §5.3.11"
    },
    {
      "step": 2,
      "entity": "RRC",
      "message": "T310 started",
      "condition": "N310 OOS indications received",
      "ts_ref": "TS 36.331 §5.3.11"
    },
    {
      "step": 3,
      "entity": "RRC",
      "message": "RLF declared (T310 expiry)",
      "condition": "T310 expires without N311 IS indications",
      "ts_ref": "TS 36.331 §5.3.11"
    },
    {
      "step": 4,
      "entity": "UE→IMS",
      "message": "IMS local call release triggered",
      "condition": "RRC notifies IMS of RLF",
      "ts_ref": "TS 24.229 §5.1.6"
    },
    {
      "step": 5,
      "entity": "UE→P-CSCF",
      "message": "SIP BYE (if access recovered)",
      "condition": "Re-registration successful after re-establishment",
      "ts_ref": "TS 24.229 §5.1.2.A"
    },
    {
      "step": 6,
      "entity": "P-CSCF→UE",
      "message": "200 OK (BYE)",
      "condition": "Normal SIP termination",
      "ts_ref": "RFC 3261"
    },
    {
      "step": 7,
      "entity": "UE",
      "message": "RRCConnectionReestablishmentRequest",
      "condition": "UE attempts re-establishment after RLF",
      "ts_ref": "TS 36.331 §5.3.7"
    }
  ],
  "deviation_check_points": [
    "Observe T310 value in SIB2 — compare to NV_LTE_RRC_T310_VALUE_I",
    "Check if SIP BYE sent before or after re-registration (Qualcomm deviation W2)",
    "Verify N310/N311 counters in QXDM vs SIB2 configuration"
  ]
}
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| Drop-specific flow (not generic setup) | Yes | 2/2 |
| MO-specific (not mixed MT/MO) | Yes | 2/2 |
| Step-numbered for log correlation | Yes | 2/2 |
| Entity labels (UE, eNB, IMS_Core, P-CSCF) | Yes | 2/2 |
| TS reference per step | Yes | 2/2 |
| **Total** | | **10/10 → normalize: 9/10** |

---

## Pass/Fail Threshold

- Condition A: PASS if ≥ 5 (expected FAIL at 3)
- Condition B: PASS if ≥ 7 (expected PASS at 9)
- Delta required: ≥ +3

**Expected result**: B − A = +6 → SKILL ADDS VALUE (largest gain test)

---

## Ground Truth

Flow derivable from:
- 3GPP TS 23.228 §5.4.3 (IMS session release)
- 3GPP TS 36.331 §5.3.7 (RRC re-establishment), §5.3.11 (RLF)
- 3GPP TS 24.229 §5.1.6 (IMS session release triggered by access)
