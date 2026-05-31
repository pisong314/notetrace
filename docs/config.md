# Configuration overrides

NoteTrace ships with sensible defaults baked in — boilerplate phrases to skip, section headers to ignore, a curated synonym table for shorthand. Two optional TOML files let you override these without rebuilding.

## File layout

Drop either file in `<NOTETRACE_DATA_DIR>/config/`:

```
<data_dir>/
├── config/
│   ├── filters.toml             # ← optional
│   └── custom_synonyms.toml     # ← optional
├── lookup/...
└── ner/...
```

Both files are picked up automatically when the engine loads. Examples shipping in the dist:

- `data/config/filters.toml.example`
- `data/config/custom_synonyms.toml.example`

Copy the `.example` file to drop the `.example` suffix to activate.

Explicit paths also work from the Python API (and a future REST flag):

```python
engine = Engine.load(
    "/path/to/data",
    filters_path="/etc/notetrace/filters.toml",
    synonyms_path="/etc/notetrace/custom_synonyms.toml",
)
```

## Replace-per-key semantics

Both files use **replace-per-key** overrides: every top-level key is optional, and if a key is present its value fully replaces the built-in default for that field. If a key is absent, the built-in default is unchanged.

There is no "extend" mode — to add to a default list, copy the default from the `.example` file and append your additions.

## `filters.toml` — what gets matched

Three optional keys. Each is a list of lowercase strings.

```toml
# Phrases that, if they appear as a summary entity, are skipped before any
# matching is attempted.
boilerplate_phrases = [
    "the patient", "patient reports", "no known", "follow up",
    "within normal limits", "unremarkable",
    # ...
]

# If a summary entity falls inside a line that starts with one of these
# (case-insensitive), it's skipped — section headers shouldn't be matched
# as content.
skip_section_prefixes = [
    "history of present illness", "chief complaint",
    "assessment and plan", "physical examination",
    # ...
]

# Only summary entities whose semantic type appears in this list are
# considered for matching.
allowed_semantic_types = ["disorder", "finding", "substance", "procedure"]
```

Tune these when you see false-positive noise (e.g. boilerplate appearing in your output) or when your notes use section headers NoteTrace doesn't know about.

## `custom_synonyms.toml` — terms NoteTrace should treat as equivalent

A supplemental synonym layer on top of NoteTrace's embedded UMLS table. Two uses:

1. **Non-clinical shorthand** that has no SNOMED concept — pharmacy/route abbreviations (`prn`, `bid`, `po`), chart notation (`dx`, `hx`, `s/p`), conversational shorthand (`w/`, `r/o`, `c/o`). These ship in the default.
2. **Domain-specific aliases** UMLS misses — internal codes, hospital-specific terms, regional variants.

```toml
[synonyms]
prn   = ["as needed"]
bid   = ["twice daily"]
"s/p" = ["status post"]

# Your additions:
"ed visit" = ["emergency department visit", "er visit"]
"my-hospital-code-1234" = ["acute appendicitis"]
```

Keys and values are matched lowercased. Multi-word values are also split on whitespace, so `"as needed"` makes both `"as needed"` and `needed` searchable as synonyms.

Clinical concept abbreviations like `HTN → hypertension` or `MI → myocardial infarction` are **not** in the default — the embedded NER and UMLS handle them on the CUI path. Add them back here if you want belt-and-suspenders coverage; the dropped entries are listed (commented out) in the shipped `custom_synonyms.toml.example` so you can uncomment what you want.

## Common scenarios

| You want to … | Edit |
|---|---|
| Suppress a noisy boilerplate phrase | `filters.toml` → `boilerplate_phrases` |
| Recognise your hospital's section headers | `filters.toml` → `skip_section_prefixes` |
| Restrict matching to drugs only | `filters.toml` → `allowed_semantic_types = ["substance"]` |
| Add a local abbreviation | `custom_synonyms.toml` → `[synonyms]` |
| Add a vendor-specific code as a synonym for a SNOMED concept | `custom_synonyms.toml` → `[synonyms]` |
