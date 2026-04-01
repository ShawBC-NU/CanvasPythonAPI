---
name: canvus_api project structure and terminology
description: Core facts about the canvus_api Python library — module responsibilities, naming conventions, API quirks, and documentation gaps identified
type: project
---

The project is an async Python client library for the Canvus collaborative whiteboard platform. All API calls are async/await; `CanvusClient` must always be used as `async with CanvusClient(...) as client:`.

**Module responsibilities:**
- `client.py` — all API operations, retry logic, streaming (NDJSON via `subscribe()`)
- `models.py` — Pydantic models; widget hierarchy rooted at `BaseWidget`
- `exceptions.py` — typed exception hierarchy rooted at `CanvusAPIError`
- `geometry.py` — `Point`, `Size`, `Rectangle` dataclasses + spatial functions on widget objects
- `filters.py` — `Filter` class with 16+ operators for client-side filtering passed to `list_widgets()` / `list_canvases()`
- `search.py` — `CrossCanvasSearch` iterates canvases to find widgets across the server
- `export.py` — `WidgetExporter` / `WidgetImporter` for disk-based round-trip widget serialization
- `widget_operations.py` — `WidgetZoneManager`, `BatchWidgetOperations`, spatial grouping utilities; all are purely local (no API calls)

**Naming conventions and gotchas:**
- PDF widget_type string is `"Pdf"` (capital P, lowercase df) — not `"PDF"`
- Widget `location` and `size` are plain dicts: `{"x": float, "y": float}` and `{"width": float, "height": float}`
- `id` and `state` on widgets are server-managed; never include them in create payloads
- The `/canvases/{id}/widgets` endpoint returns heterogeneous types mapped to the generic `Widget` model; type-specific endpoints return `Note`, `Image`, etc.
- `CanvasBackground` is a pseudo-widget injected by the server; `Widget.model_validate()` has special-case handling for it
- Access tokens: `plain_token` is only returned on creation via `TokenResponse`; cannot be retrieved later
- Parenting offset formula: `new_relative_location = current_canvas_location - parent_location - 30`

**Documentation created (2026-04-02):**
- `/Users/sbc73526/CanvasPythonAPI/docs/getting-started.md` (15K)
- `/Users/sbc73526/CanvasPythonAPI/docs/api-reference.md` (34K)
- `/Users/sbc73526/CanvasPythonAPI/docs/architecture.md` (28K)

**Why:** First documentation for this codebase; no prior docs existed.
**How to apply:** Use these files as the canonical reference for the library's public API and internal design. Update them when client.py or other modules change.
