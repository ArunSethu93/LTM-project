# Drawing QA & Conflict Detection Agent

Ingests engineering/construction drawings (DXF, PDF, PNG/JPG), converts them into a
structured entity + relationship graph, answers grounded questions about them, and
compares two revisions to surface changes and conflicts — every answer carries
page/bounding-box evidence and a confidence score.

This repository is an **architecture-first scaffold**: every interface is real and the
end-to-end flow runs today with zero ML dependencies (stub adapters + a synthetic
sample). Heavy models are opt-in through configuration.

## Flow

```
                 ┌──────────── ingestion ────────────┐
 drawing  ──▶    │ DXF  → CADParser (ezdxf)          │
 (pdf/png/dxf)   │ PDF  → PyMuPDF render + text layer│──▶ TextSpans + raw geometry
                 │ IMG  → OCREngine (Paddle/stub)    │
                 └───────────────────────────────────┘
                                    │
              EntityExtractor (tag regex, prose-vs-label rule)
                                    │
              RelationshipLinker (labels, connectivity, references)
                                    │
                 DrawingDocument  ──▶  networkx graph
                          │                     │
            ┌─────────────┘                     └──────────────┐
   QAPipeline                                        RevisionComparator
   intent → retrieve → deterministic answer           align → diff → conflict rules
          → optional VLM refinement                          │
          → Answer(text, confidence, evidence)        ComparisonReport(changes,
                                                       conflicts, evidence)
```

The VLM never sees a drawing without structured context, and the deterministic answer
is always computed first — so an unavailable or weak model degrades the *phrasing*,
not the correctness.

## Layout

```
app/
  domain/        BBox, TextSpan, Entity, Relationship, DrawingDocument, Evidence, Answer,
                 EntityChange, Conflict, ComparisonReport      (no library-specific code)
  interfaces/    OCREngine · CADParser · VisionLanguageModel · LanguageModel ·
                 SymbolDetector · DocumentStore                (abstract contracts)
  adapters/      concrete implementations behind those contracts
    ocr/         stub · paddle · tesseract
    cad/         dxf_ezdxf
    vision/      stub · opencv symbol detector
    vlm/         stub · hf_qwen2_5_vl · openai_compatible
    llm/         stub · openai_compatible
    storage/     local JSON store
    catalog.py   name → implementation map (add a line to add an adapter)
    factory.py   settings → concrete component
  ingestion/     loader, pdf_reader, raster_reader, pipeline
  extraction/    tag_patterns, entity_extractor, linker
  graph/         builder (networkx), queries (locate / neighbours / integrity)
  qa/            intents, retriever, formatter, confidence, crops, pipeline
  compare/       aligner, differ, conflicts (rule registry), report
  services/      drawing / qa / revision services + Container (composition root)
  api/           FastAPI routes + request/response schemas
  core/          config, registry, prompts, logging, errors
config/          default.yaml, local.example.yaml, prompts/*.txt
frontend/        Streamlit operator UI (HTTP only)
samples/         synthetic two-revision P&ID (DXF) + raster scan + generator
tests/           24 tests covering config, extraction, ingestion, graph, QA, diff, API
scripts/         dqa_cli.py
```

## Run it locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt

python samples/generate_samples.py     # synthetic rev A / rev B + a raster scan
pytest                                 # 24 tests, no ML deps needed

python scripts/dqa_cli.py demo         # full flow in the terminal

uvicorn app.main:app --reload          # API  → http://localhost:8000/docs
streamlit run frontend/streamlit_app.py  # UI → http://localhost:8501
```

Quick API check:

```bash
curl -F "file=@samples/plant_rev_a.dxf" -F "revision=A" localhost:8000/api/v1/drawings
curl -X POST localhost:8000/api/v1/qa/ask \
  -H 'content-type: application/json' \
  -d '{"document_id":"<id>","question":"What is connected to P-101?"}'
```

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/health` | status + which adapters are active |
| GET | `/api/v1/config` | effective settings + adapter registry |
| POST | `/api/v1/drawings` | upload + ingest (multipart) |
| POST | `/api/v1/drawings/from-path` | ingest a server-side file |
| GET | `/api/v1/drawings` | list |
| GET | `/api/v1/drawings/{id}` | full structured document |
| GET | `/api/v1/drawings/{id}/entities` | entities + relationships |
| GET | `/api/v1/drawings/{id}/graph` | node/edge JSON |
| GET | `/api/v1/drawings/{id}/pages/{i}/image` | page raster |
| POST | `/api/v1/qa/ask` | grounded answer + confidence + evidence |
| POST | `/api/v1/revisions/compare` | changes + conflicts report |

## Swapping components

Nothing model-specific lives in business logic. To switch, edit `config/local.yaml`:

```yaml
ocr:    { engine: paddle }              # stub | paddle | tesseract
vision: { symbol_detector: opencv }     # stub | opencv
vlm:    { provider: hf_qwen2_5_vl, model_name: "Qwen/Qwen2.5-VL-7B-Instruct", device: cuda }
llm:    { provider: openai_compatible, api_base: "http://localhost:11434/v1" }
```

then `pip install -r requirements-ml.txt`. Any key is also settable by env var:
`DQA__VLM__PROVIDER=stub`, `DQA__QA__TOP_K=12`.

To add a **new** adapter: implement the interface in `app/adapters/<kind>/`, expose a
`build(settings)` factory, add one line to `app/adapters/catalog.py`. To add a new
**conflict rule**: write a function in `app/compare/conflicts.py`, decorate it with
`@rule("name")`, list the name under `compare.enabled_rules`.

## Sample data

`samples/generate_samples.py` builds a tiny P&ID in two revisions:

* **Rev A** — `TK-100 → P-101 → V-101 → TK-200`, with `FT-301` on the discharge, plus a
  note referencing `FT-301`.
* **Rev B** — `V-101` renamed to `V-102`, `P-101` moved 30 units, `V-103` added,
  `FT-301` deleted but **still referenced by the stale note**.

Comparing them produces: 1 added, 1 removed, 1 moved, 1 renamed, connection changes, and
conflicts including `removed_referenced_entity` (HIGH) and `dangling_reference` (MEDIUM).

## Known limitations (deliberate, for the next iteration)

* Symbol detection is a stub by default; the OpenCV baseline is contour heuristics, not
  a trained detector. Real P&ID symbol recognition needs a fine-tuned detector or
  template matching against a symbol library.
* Connectivity for rasters is not implemented — only CAD line endpoints are traced.
  Raster line following (Hough/skeletonisation) is the gap.
* Tag extraction is regex-based and discipline-agnostic; multi-part tags, revision
  clouds, title blocks and sheet cross-references are not modelled yet.
* Alignment between revisions is tag-first then geometric; drawings with different
  origins/scales need a registration step before comparison is meaningful.
* Storage is JSON on disk — fine for one machine, not for concurrent users.
