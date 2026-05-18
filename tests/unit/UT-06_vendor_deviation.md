# UT-06: vendor_deviation — Phase 4 Qualcomm-Specific Deviation

**Issue protocol**: 2026-05-17_volte-call-drop  
**RCA Phase**: 4 — confirm deviation from 3GPP standard  
**Intent under test**: `vendor_deviation`

---

## Test Input

```json
{
  "rat": "LTE",
  "module": "IMS",
  "procedure": "VoLTE_call",
  "intent": "vendor_deviation",
  "context": "SIP BYE timing on RLF — Qualcomm sends BYE immediately vs standard deferred behavior"
}
```

---

## Condition A — Without Skill (raw spec-rag)

**Query sent to spec-rag**:
```
"Qualcomm VoLTE SIP BYE vendor deviation LTE RLF"
```

**Expected failure modes**:
- spec-rag corpus typically contains 3GPP specs, not vendor deviation docs
- Returns standard SIP BYE behavior (TS 24.229), not Qualcomm delta
- No CR reference (only Qualcomm internal CRs exist outside 3GPP)
- No NV item link (Qualcomm proprietary)
- Engineer has no way to know deviation exists without prior experience

**Sample bad output**:
```
According to 3GPP TS 24.229, when a UE detects a radio link failure,
it should attempt re-registration before sending a SIP BYE. The IMS
stack will handle the session termination according to the standard
procedure...
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| Qualcomm-specific behavior delta identified | No | 0/3 |
| Standard behavior (3GPP baseline) stated | Yes | 1/2 |
| CR reference (internal Qualcomm CR) | No | 0/2 |
| NV item controlling the deviation | No | 0/2 |
| Actionable engineer guidance | No | 0/1 |
| **Total** | | **1/10 → normalize: 1/10** |

---

## Condition B — With Skill (qcomm-knowledge-query)

**Skill constructs query**:
```
"LTE IMS VoLTE_call vendor deviation Qualcomm vs standard specification"
```
doc_type filter: `vendor_deviation` (Qualcomm corpus)

**Expected output schema**:
```json
{
  "deviations": [
    {
      "feature": "SIP BYE on RLF",
      "standard_behavior": "UE defers SIP BYE until after successful re-registration (TS 24.229 §5.1.6.A)",
      "qualcomm_behavior": "UE sends SIP BYE immediately upon RLF detection, before attempting re-establishment",
      "rationale": "Faster IMS resource release on network side; avoids dangling sessions",
      "cr_reference": "CR-QCOM-IMS-2341",
      "nv_item": "NV_IMS_BYE_ON_RLF_IMMEDIATE_I",
      "nv_default": 1,
      "nv_effect": "Set to 0 to match standard deferred behavior",
      "ts_ref": "TS 24.229 §5.1.6"
    },
    {
      "feature": "IMS re-registration on HO",
      "standard_behavior": "Re-registration triggered after RRC re-establishment (TS 24.229 §5.1.2.A)",
      "qualcomm_behavior": "Pre-emptive re-registration initiated during HO preparation phase",
      "cr_reference": "CR-QCOM-IMS-1876",
      "nv_item": "NV_IMS_PREEMPTIVE_REREG_HO_I",
      "ts_ref": "TS 24.229 §5.1.2"
    }
  ]
}
```

**Criteria scoring**:
| Criterion | Pass? | Score |
|-----------|-------|-------|
| Qualcomm-specific behavior delta identified | Yes | 3/3 |
| Standard behavior (3GPP baseline) stated | Yes | 2/2 |
| CR reference (internal Qualcomm CR) | Yes | 2/2 |
| NV item controlling the deviation | Yes | 2/2 |
| Actionable engineer guidance | Yes | 1/1 |
| **Total** | | **10/10 → normalize: 8/10** |

---

## Pass/Fail Threshold

- Condition A: PASS if ≥ 5 (expected FAIL at 1)
- Condition B: PASS if ≥ 7 (expected PASS at 8)
- Delta required: ≥ +3

**Expected result**: B − A = +7 → SKILL ADDS VALUE (largest absolute gain)

---

## Note on Corpus Dependency

This test is most sensitive to **corpus completeness** (SKILL.md V1 weakness W8: hardcoded to Qualcomm CP common layer, may exclude CP-specific IMS deviation docs). If spec-rag corpus doesn't contain vendor deviation docs, Condition B score will degrade toward 3/10. V2 fix: configurable corpus path.
