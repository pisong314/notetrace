# Examples — what NoteTrace catches

Every entity gets a traffic-light **`level`**:

- 🟢 **`verified`** — the source supports the claim.
- 🟡 **`plausible`** — supported, but broader than the source (the note generalised).
- 🔴 **`conflicting`** / **`needs_review`** — cannot find source or a flat disagreement (wrong medication dose/procedure side). A human must review.

---

## 1. Supported vs unsupported

**Source transcript**:
> Doctor: any past medical history?\
> Patient: I had a heart attack last year, and I've got hypertension.

**LLM-generated note**:
> PMHx: MI, hypertension, hypercholesterolemia.

**NoteTrace output** (excerpt):
```json
{
  "items": [
    { "entity_text": "MI",
      "provenance": { "level": "verified", "match_kind": "exact",
                      "source_text": "heart attack", "confidence": 1.0 } },
    { "entity_text": "hypertension",
      "provenance": { "level": "verified", "match_kind": "exact",
                      "source_text": "hypertension", "confidence": 1.0 } },
    { "entity_text": "hypercholesterolemia",
      "provenance": { "level": "needs_review", "hints": [] } }
  ]
}
```

**Takeaway**: *MI* and *hypertension* are both in the source → 🟢 `verified`. *MI* matched the lay term "heart attack": NoteTrace works on *meaning*, not just form. *Hypercholesterolemia* appears nowhere → 🔴 `needs_review`, the textbook hallucination.

---

## 2. Did the note add detail the source didn't?

A note can claim *more* detail than the source supports (risky) or *less* (harmless). NoteTrace tells them apart.

### Risky — note more specific than source

**Source transcript**:
> Doctor: Any past medical history?\
> Patient: I have diabetes.

**LLM-generated note**:
> PMHx: Type 2 Diabetes Mellitus.

**NoteTrace output** (excerpt):
```json
{
  "entity_text": "Type 2 Diabetes Mellitus",
  "provenance": { "level": "plausible", "match_kind": "source_broader",
                  "source_text": "diabetes", "confidence": 1.0 }
}
```

The patient said only *diabetes*; the note pinned it to *Type 2* — detail the source never gave (could be Type 1) → 🟡 `plausible`.

### Safe — note more general than source

**Source transcript**:
> Patient diagnosed with T2DM, now on metformin.

**LLM-generated note**:
> Past hx: diabetes.

**NoteTrace output** (excerpt):
```json
{
  "entity_text": "diabetes",
  "provenance": { "level": "verified", "match_kind": "source_narrower",
                  "source_text": "T2DM", "confidence": 1.0 }
}
```

The note generalised *T2DM* to *diabetes* — still true, nothing invented → 🟢 `verified`.

**Takeaway**: more specific than the record is the dangerous direction (the AI inventing detail); more general is harmless. A keyword search would treat both as the same hit on "diabetes".

---

## 3. No match, but not silent — `hints`

**Source transcript**:
> Doctor: any other concerns?\
> Patient: my urine has been a weird dark colour lately.

**LLM-generated note**:
> Haematuria.

**NoteTrace output** (excerpt):
```json
{
  "entity_text": "Haematuria",
  "provenance": {
    "level": "needs_review",
    "hints": [
      {
        "source_text": "Doctor: any other concerns? Patient: my urine has been a weird dark colour lately.",
        "source_span": { "start": 0, "end": 82 },
        "similarity": 0.58
      }
    ]
  }
}
```

**Takeaway**: *haematuria* means blood in the urine, but the source only mentions discolouration (which has many non-blood causes). The note may still be right depending on interpretation so NoteTrace reports only that **the source doesn't confirm it**: 🔴 `needs_review`. The `hints` list points to a possible source, so a human reviewer knows where to look.

---

## 4. Dosage conflict — right drug, wrong dose

**Source transcript**:
> patient takes metformin 500 mg twice a day

**LLM-generated note**:
> Metformin 1000 mg BD.

**NoteTrace output** (excerpt):
```json
{
  "entity_text": "Metformin",
  "provenance": {
    "level": "conflicting", "match_kind": "exact",
    "source_text": "metformin", "confidence": 1.0,
    "checks": [
      { "attribute": "dosage", "status": "conflicts",
        "summary_value": "1000mg", "source_value": "500 mg" }
    ]
  }
}
```

**Takeaway**: the drug is in the source, but the doses differ (1000 vs 500 mg) → 🔴 `conflicting`, so the traffic-light won't show green on a doubled dose.

---

## 5. Laterality conflict — right procedure, wrong side

**Source transcript**:
> Doctor: You will be having a left knee replacement. Let's book you in.

**LLM-generated note**:
> Pt booked for R TKR.

**NoteTrace output** (excerpt):
```json
{
  "entity_text": "TKR",
  "provenance": {
    "level": "conflicting", "match_kind": "source_broader",
    "source_text": "knee replacement", "confidence": 1.0,
    "checks": [
      { "attribute": "laterality", "status": "conflicts",
        "summary_value": "right", "source_value": "left" }
    ]
  }
}
```

**Takeaway**: the procedure is correct (acronym TKR = knee replacement) but the side is wrong → 🔴 `conflicting`.

---
