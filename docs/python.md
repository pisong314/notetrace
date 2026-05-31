# Python API

In-process bindings for NoteTrace. Best when your pipeline is already Python and you want zero IPC overhead.

The README covers download + the JSON CLI. This page focuses on the Python wheel once it's installed.

## Install

Pre-built wheel ships under `wheels/` in the dist. Requires Python 3.10+.

```bash
python3 -m pip install wheels/notetrace-0.1.0-cp310-abi3-*.whl
export NOTETRACE_DATA_DIR=/path/to/dist/data
```

One wheel works across CPython 3.10 / 3.11 / 3.12 / 3.13 (abi3).

## Load the engine

```python
from notetrace import Engine

engine = Engine.load()    # reads NOTETRACE_DATA_DIR
```

`Engine.load()` also accepts an explicit path — `Engine.load("/path/to/data")` — if you'd rather not use the env var.

Loading takes ~5–10 seconds (NER vocab + SNOMED archives + Word2Vec FST). Hold on to the instance for the lifetime of your process; per-call cost is fast.

## Run NoteTrace

```python
report = engine.find_provenance(
    summary="Patient with hypertension. Started Metformin 1g BD for Type 2 diabetes.",
    source =(
        "Doctor: any history of high blood pressure?\n"
        "Patient: yes, hypertension. Also diabetes — I'm on metformin 500 mg twice a day."
    ),
)
```

`find_provenance` returns a dict with the same shape the REST endpoint and JSON CLI emit:

```python
report = {
    "items": [...],   # one entry per medical entity in the summary
    "stats": {...},   # totals by level
}
```

## Output shape

Each item has a `provenance` block. For a **matched** entity (`verified` / `plausible` / `conflicting`):

| Field | Type | Meaning |
|---|---|---|
| `level`       | `"verified"` / `"plausible"` / `"conflicting"` / `"needs_review"` | Traffic-light label |
| `match_kind`  | `"exact"` / `"variant"` / `"source_narrower"` / `"source_broader"` / `"partial"` | How the source relates to the note — see [examples.md](examples.md) |
| `source_text` | `str` | The span in the source that supported the entity |
| `source_span` | `{"start": int, "end": int}` | Char offsets of that source span |
| `confidence`  | `float` (0.0–1.0) | Match confidence |
| `checks`      | list of `{attribute, status, summary_value, source_value}` | Structured verifiers (dosage, laterality); `status: "agrees" \| "conflicts"`. A `conflicts` check sets `level` to `"conflicting"`. |

For a **`needs_review`** entity (nothing in the source supported it), the block is instead `{"level": "needs_review", "hints": [...]}`, where each hint is `{source_text, source_span, similarity}` — the nearest leads, or `[]` if none. (No `match_kind` / `source_text` / `confidence` at the top level in this case.)

The wrapping item also has `entity_text`, `entity_span`, and `cui` for the summary-side entity.

Iterate the items:

```python
for item in report["items"]:
    prov = item["provenance"]
    print(f"{item['entity_text']:30s} → level={prov['level']:<13s} match_kind={prov.get('match_kind')}")
```

Output (for the example above):

```
hypertension                   → level=verified     match_kind=exact
Metformin                      → level=conflicting  match_kind=exact          # dosage 1g ≠ 500 mg
Type 2 diabetes                → level=plausible    match_kind=source_broader
```

## Filter by level

For batch use cases the `level` field is usually all you need:

```python
unsupported = [it for it in report["items"] if it["provenance"]["level"] == "needs_review"]
```

That gives you the list of summary entities that NoteTrace couldn't find any support for in the source — i.e. the hallucination candidates.

For finer-grained policy, combine `level` with `match_kind`:

```python
# Catch "summary specified beyond what the source justifies"
specifications = [
    it for it in report["items"]
    if it["provenance"].get("match_kind") == "source_broader"
]
```

See [examples.md](examples.md) for what each `match_kind` value means in context.

## Custom config

Filter overrides (boilerplate phrases, section headers, allowed semantic types) and custom synonym tables can be loaded at startup:

```python
engine = Engine.load(
    "/path/to/data",
    filters_path="/etc/notetrace/filters.toml",
    synonyms_path="/etc/notetrace/custom_synonyms.toml",
)
```

If you drop the files under `<NOTETRACE_DATA_DIR>/config/` they're auto-discovered — no extra args needed. Full reference: [config.md](config.md).
