# REST interface

Two binaries ship with NoteTrace: a long-running HTTP server and a one-shot CLI. Both wrap the same engine and emit the same JSON shape.

## Start the server

```bash
# Linux / macOS
./notetrace-server --listen 127.0.0.1:8080

# Windows (cmd)
notetrace-server.exe --listen 127.0.0.1:8080
```

Engine + data load once at startup (~5–10 s), then every request is in-process inference. Loopback-only — no auth, no TLS. For cross-machine access, put a reverse proxy (nginx, caddy, envoy) in front and terminate TLS there.

Run from the unzipped dist directory so `./data` is auto-discovered, or set `NOTETRACE_DATA_DIR` / pass `--data-dir`.

## Health check

```bash
curl -s http://127.0.0.1:8080/v1/health
# => {"status":"ok"}
```

## Find provenance

```bash
# Linux / macOS
curl -s -X POST http://127.0.0.1:8080/v1/notetrace \
  -H 'content-type: application/json' \
  -d '{"summary":"Patient with hypertension.",
       "source":"Doctor: history of high blood pressure? Patient: yes, hypertension."}'
```

```cmd
:: Windows (cmd)
curl -s -X POST http://127.0.0.1:8080/v1/notetrace -H "content-type: application/json" -d "{\"summary\":\"chest pain\",\"source\":\"patient reports chest pain\"}"
```

Output (truncated):

```json
{
  "items": [
    { "entity_text": "hypertension",
      "entity_span": {"start": 13, "end": 25},
      "cui": "38341003",
      "provenance": { "level": "verified", "match_kind": "exact",
                      "source_text": "high blood pressure",
                      "source_span": {"start": 19, "end": 38},
                      "confidence": 1.0, "checks": [] } }
  ],
  "stats": { "total": 1, "verified": 1, "plausible": 0, "conflicting": 0, "needs_review": 0 }
}
```

The request body has two required fields:

- **`summary`** — the AI-generated clinical note being checked
- **`source`** — the original transcript or source document

Field-by-field reference for the response is in [python.md](python.md) — same shape across both interfaces.

## One-shot JSON CLI

For ad-hoc use and low-volume integrations, `notetrace-json` reads a JSON request on stdin and writes the response on stdout. Fresh process per call (~5 s load-dominated). For bulk work, the server above is ~10× faster per note since the engine stays warm.

```bash
# Linux / macOS
echo '{"summary":"Pt has HTN.","source":"history of hypertension"}' | ./notetrace-json

# From file
./notetrace-json --input-file in.json --output-file out.json
```

Same output shape as the REST endpoint.

Run from the unzipped dist directory so `./data` is auto-discovered, or set `NOTETRACE_DATA_DIR` / pass `--data-dir`.

## When to use which

| Use case | Pick |
|---|---|
| One-off testing, shell scripts, ad-hoc verification | `notetrace-json` |
| Batch processing, web app backend, sustained QPS | `notetrace-server` |
| In-process Python pipelines | The Python wheel — see [python.md](python.md) |
