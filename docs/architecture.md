# Architecture Guide

This document explains how `canvus_api` is structured internally, the design decisions behind each module, and the advanced features you can compose together to build sophisticated integrations.

## Table of Contents

- [Module Overview](#module-overview)
- [Data Flow](#data-flow)
- [The CanvusClient Internals](#the-canvusclient-internals)
- [Models (models.py)](#models-modelspy)
- [Exception Hierarchy (exceptions.py)](#exception-hierarchy-exceptionspy)
- [The Parenting System](#the-parenting-system)
- [Spatial Operations (geometry.py)](#spatial-operations-geometrypy)
- [Filtering (filters.py)](#filtering-filterspy)
- [Cross-Canvas Search (search.py)](#cross-canvas-search-searchpy)
- [Export and Import (export.py)](#export-and-import-exportpy)
- [Widget Zone Operations (widget_operations.py)](#widget-zone-operations-widget_operationspy)
- [Design Patterns and Extension Points](#design-patterns-and-extension-points)

---

## Module Overview

```
canvus_api/
├── client.py           # CanvusClient — all API calls, retry logic, streaming
├── models.py           # Pydantic models for every API entity
├── exceptions.py       # Exception hierarchy
├── geometry.py         # Spatial primitives (Point, Size, Rectangle) and widget spatial ops
├── filters.py          # Client-side filtering with 16+ operators
├── search.py           # CrossCanvasSearch — search widgets across multiple canvases
├── export.py           # WidgetExporter / WidgetImporter — round-trip serialization
└── widget_operations.py # WidgetZoneManager, BatchWidgetOperations, spatial grouping
```

**Dependencies between modules:**

```
client.py
    └── models.py
    └── exceptions.py
    └── filters.py

geometry.py
    └── models.py

filters.py
    └── geometry.py

search.py
    └── client.py
    └── geometry.py
    └── models.py

export.py
    └── client.py
    └── models.py
    └── exceptions.py

widget_operations.py
    └── models.py
    └── geometry.py
```

`client.py` is the root of the dependency tree. All higher-level modules (`search`, `export`, `widget_operations`) take a `CanvusClient` instance at construction time and use it for API access.

---

## Data Flow

Every API call follows this path:

```
Your code
    → CanvusClient method
        → _request() [builds URL, headers, retry loop]
            → aiohttp HTTP call
                → Server response
            → JSON decode
            → Pydantic model validation (if response_model provided)
        → Returns typed model or raw dict
    → Your code receives result
```

For streaming endpoints the path diverges:

```
Your code
    → client.subscribe() [appends subscribe=true query param]
        → _stream_request() [long-lived GET, no total timeout]
            → aiohttp async generator over response.content.readline()
            → Yields non-empty lines
        → JSON decode each line
        → Pydantic model validation per line
    → Your code receives typed updates via async for
```

---

## The CanvusClient Internals

### Authentication

The client sends `Private-Token: <api_key>` as an HTTP header on every request. There is no session cookie or OAuth flow required at the library level.

### URL Construction

All endpoints are prefixed with `/api/v1/`:

```python
def _build_url(self, endpoint: str) -> str:
    endpoint = endpoint.lstrip("/")
    return f"{self.base_url}/api/v1/{endpoint}"
```

So `client.get_canvas("abc")` calls `GET {base_url}/api/v1/canvases/abc`.

### Retry Logic

The `_request()` method implements a simple retry loop with exponential backoff:

1. Make the request.
2. If the response status is retryable (connection error, HTTP 408/429/5xx except 501), sleep for `retry_delay` seconds.
3. Multiply the delay by `retry_backoff` and try again.
4. After `max_retries + 1` total attempts, raise the last exception.

Non-retryable errors (401, 404, 501, other 4xx) are raised immediately.

### Per-Request Sessions

Each `_request()` call creates a new `aiohttp.ClientSession`. This avoids connection pool exhaustion during long-running processes but means there is a small overhead per call. The `session` attribute on the client is used only for multipart file uploads (images, videos, PDFs).

### Streaming Architecture

`_stream_request()` is a separate async generator that keeps a single HTTP connection open:

```python
async def _stream_request(self, method, endpoint, *, params=None, headers=None):
    # Sets total=None for unlimited duration, sock_read=60 for keepalive detection
    timeout = aiohttp.ClientTimeout(total=None, connect=30, sock_read=60)
    async with aiohttp.ClientSession() as session:
        async with session.request(method, url, ...) as response:
            while True:
                line = await response.content.readline()
                if not line:
                    break         # Connection closed by server
                if line.strip():
                    yield line    # Skip keepalive blank lines
```

`subscribe()` wraps this generator, appending `subscribe=true` to the query string (the Canvus server uses this parameter to switch the endpoint into streaming mode), then optionally parses each JSON line into a Pydantic model.

---

## Models (models.py)

All API entities are Pydantic `BaseModel` subclasses. The hierarchy for widget types:

```
BaseWidget
├── Note
├── Image
├── Browser
├── Video
├── PDF
├── Anchor
└── Widget  (generic, used for custom types and the /widgets endpoint)

Connector      (separate — has src/dst endpoints rather than location/size)
ConnectorEndpoint
```

### Key Fields on BaseWidget

| Field | Type | Notes |
|-------|------|-------|
| `id` | `str` | UUID v4, server-generated — never set this in create payloads |
| `state` | `str` | Server-managed lifecycle state (`"normal"`, `"deleted"`) |
| `widget_type` | `str` | String identifier, e.g. `"Note"`, `"Image"`, `"Pdf"` |
| `location` | `Dict[str, float]` | `{"x": ..., "y": ...}` in canvas coordinates |
| `size` | `Dict[str, float]` | `{"width": ..., "height": ...}` |
| `depth` | `float` | Z-order; higher values appear on top |
| `scale` | `float` | Scale factor, default `1.0` |
| `pinned` | `bool` | Pinned widgets cannot be moved by canvas users |
| `parent_id` | `str \| None` | ID of parent widget, or `None` for root-level |

> **Note on PDF widget_type:** The PDF model uses `widget_type = "Pdf"` (capital P, lowercase df). If you filter by type string, use `"Pdf"` exactly.

### The Widget Generic Model

The `/canvases/{id}/widgets` endpoint returns a heterogeneous list. The library maps all items to the `Widget` model, which has a `config: Dict[str, Any]` field to accommodate type-specific properties not captured by the typed subclasses.

When you need strong typing (e.g. `note.text`), use the type-specific endpoints:

```python
# Generic — returns Widget with config dict
all_widgets = await client.list_widgets(canvas_id)

# Type-specific — returns Note with typed .text field
notes = await client.list_notes(canvas_id)
```

### CanvasBackground Special Case

The `Widget.model_validate()` override handles the `CanvasBackground` pseudo-widget that the server includes in widget lists. It injects default `location`, `size`, and `id` values so that the model validator does not reject it.

---

## Exception Hierarchy (exceptions.py)

```
Exception
└── CanvusAPIError(message, status_code, response_text)
    ├── AuthenticationError     — HTTP 401
    ├── ResourceNotFoundError   — HTTP 404
    ├── RateLimitError          — HTTP 429
    ├── ServerError             — HTTP 5xx
    ├── TimeoutError            — HTTP 408 or connection timeout
    ├── ConnectionError         — Network-level failure
    ├── ValidationError         — Malformed payload or unexpected response
    ├── CanvasError             — Canvas-specific operation failure
    └── RetryableError
        └── TransientError      — Transient failure eligible for retry
```

Every exception exposes:
- `status_code: Optional[int]` — the HTTP status code, or `None` for network errors
- `response_text: Optional[str]` — the raw response body

The client's `_classify_error()` method maps HTTP status codes to the appropriate exception subclass before raising.

---

## The Parenting System

Widgets can form a tree hierarchy: any widget can be the parent of any other widget. The `parent_id` field on a widget stores the UUID of its parent, or `None` for root-level widgets.

### Why Positions Are Relative

When a widget has a parent, its `location` is interpreted relative to the parent's position. A child at `{"x": 50, "y": 50}` with a parent at `{"x": 200, "y": 200}` appears at canvas coordinates `{"x": 250, "y": 250}`.

This means that moving the parent automatically moves all its descendants, which is the primary use case for grouping related widgets.

### The Offset Formula

When you reparent a widget (changing its `parent_id`), you usually want it to stay in the same visual position on the canvas even though its coordinates are now relative to a new parent.

The client handles this automatically in every `update_*` method that accepts `parent_id`. The formula is:

```
new_relative_location = current_canvas_location - parent_canvas_location - 30
```

The constant `30` is an empirical offset that matches the visual origin of the parent container in the Canvus rendering engine.

Implementation in `_calculate_parent_offset()`:

```python
offset_x = current_x - parent_x - 30
offset_y = current_y - parent_y - 30
return {"x": offset_x, "y": offset_y}
```

This calculated offset is injected into the update payload, overriding any `location` value you supplied.

> **Tip:** When reparenting, do not include a `location` key in your payload. The client will calculate and set the correct one. If you include `location`, it will be overwritten.

### Circular Reference Detection

Before applying any `parent_id` change, the client calls `_check_circular_parenting()`, which:

1. Immediately raises `CanvusAPIError` if you try to set a widget as its own parent.
2. Walks up the ancestor chain of the proposed new parent, looking for the widget being reparented. If found, a cycle would be created, and the method raises `CanvusAPIError` before the HTTP request is sent.

This check requires fetching widget data from the server, so it performs an extra API call. If the widget tree cannot be fetched, the check is skipped and the reparent proceeds (to preserve backward compatibility with partial-access scenarios).

---

## Spatial Operations (geometry.py)

The geometry module provides primitives and functions for working with widget positions without needing to do coordinate math manually.

### Primitives

```python
from canvus_api.geometry import Point, Size, Rectangle

point = Point(x=100.0, y=200.0)
size  = Size(width=400.0, height=300.0)
rect  = Rectangle(x=100.0, y=200.0, width=400.0, height=300.0)
```

`Rectangle` exposes computed properties: `.left`, `.right`, `.top`, `.bottom`, `.center`, `.position`, `.size`.

### Rectangle Operations

| Function | Signature | Description |
|----------|-----------|-------------|
| `contains` | `(outer: Rectangle, inner: Rectangle) -> bool` | True if outer fully encloses inner |
| `touches` | `(rect1, rect2) -> bool` | True if rectangles touch or overlap (edges count) |
| `intersects` | `(rect1, rect2) -> bool` | True if rectangles have overlapping area (edges do not count) |
| `get_intersection` | `(rect1, rect2) -> Optional[Rectangle]` | Returns the overlap rectangle, or `None` |
| `get_union` | `(rect1, rect2) -> Rectangle` | Returns the smallest rectangle enclosing both |

### Widget-Level Operations

These functions work directly with widget model objects, extracting bounding boxes automatically:

| Function | Description |
|----------|-------------|
| `widget_bounding_box(widget)` | Returns the `Rectangle` for a widget's bounds. Handles `Connector` specially by computing bounds from endpoint locations. |
| `widget_contains(widget1, widget2)` | True if widget1 fully encloses widget2 |
| `widgets_touch(widget1, widget2)` | True if widgets touch or overlap |
| `widgets_intersect(widget1, widget2)` | True if widgets have overlapping area |
| `get_widget_intersection(widget1, widget2)` | Returns overlap rectangle or `None` |
| `get_widget_union(widget1, widget2)` | Returns bounding rectangle for both widgets |
| `distance_between_widgets(widget1, widget2)` | Minimum distance (0 if overlapping) |
| `find_widgets_in_area(widgets, area)` | Returns widgets from a list that intersect a rectangle |
| `find_widgets_containing_point(widgets, point)` | Returns widgets that contain a given point |
| `get_canvas_bounds(widgets)` | Returns the bounding rectangle of all widgets, or `None` |

### Practical Example

```python
from canvus_api.geometry import Rectangle, find_widgets_in_area, get_canvas_bounds

widgets = await client.list_widgets(canvas_id)

# Find all widgets in the top-left quadrant
search_area = Rectangle(x=0, y=0, width=1000, height=1000)
nearby = find_widgets_in_area(widgets, search_area)
print(f"Found {len(nearby)} widgets in area")

# Find the bounding box of all content
bounds = get_canvas_bounds(widgets)
if bounds:
    print(f"Content spans {bounds.width:.0f} x {bounds.height:.0f} pixels")
    print(f"  from ({bounds.left:.0f}, {bounds.top:.0f})")
    print(f"  to   ({bounds.right:.0f}, {bounds.bottom:.0f})")
```

---

## Filtering (filters.py)

The `Filter` class provides a chainable, client-side filtering system. It is used in `list_canvases()` and `list_widgets()` to narrow results after they are fetched from the server.

### Supported Operators

| Operator | `FilterOperator` value | Description |
|----------|------------------------|-------------|
| Equality | `equals`, `not_equals` | Exact match or non-match |
| Text | `contains`, `not_contains`, `starts_with`, `ends_with` | Substring or prefix/suffix |
| Comparison | `greater_than`, `less_than`, `greater_equal`, `less_equal` | Numeric comparison |
| Set | `in`, `not_in` | Value in or not in a list |
| Existence | `exists`, `not_exists` | Field is present/absent |
| Spatial | `spatial_intersects`, `spatial_contains`, `spatial_within` | Rectangle-based |
| Pattern | `wildcard_match` | `*` matches any sequence, `?` matches one character |

### Building Filters

Conditions are added with `add_condition(field, operator, value)`. The method returns `self` for chaining. Dot notation addresses nested fields.

```python
from canvus_api.filters import Filter, FilterOperator

# Find notes with yellow background
f = (Filter()
    .add_condition("widget_type", FilterOperator.EQUALS, "Note")
    .add_condition("background_color", FilterOperator.STARTS_WITH, "#ffff")
)
notes = await client.list_widgets(canvas_id, filter_obj=f)
```

### Spatial Filtering

```python
from canvus_api.filters import Filter
from canvus_api.geometry import Rectangle

# Find widgets in a specific region
region = Rectangle(x=0, y=0, width=500, height=500)
f = Filter().add_spatial_condition("intersects", region)
widgets = await client.list_widgets(canvas_id, filter_obj=f)
```

Three spatial operators are available:
- `"intersects"` — widget overlaps the search area
- `"contains"` — widget is inside the search area
- `"within"` — widget is fully contained within the search area

### Wildcard Filtering

```python
f = Filter().add_wildcard_condition("anchor_name", "Chapter *")
anchors = await client.list_widgets(canvas_id, filter_obj=f)
```

### Convenience Factory Functions

The module provides pre-built filters for common cases:

```python
from canvus_api.filters import (
    create_filter,
    create_spatial_filter,
    create_widget_type_filter,
    create_text_filter,
    create_wildcard_filter,
    combine_filters,
)

# Spatial filter
f = create_spatial_filter(Rectangle(x=0, y=0, width=1000, height=1000))

# Type filter
f = create_widget_type_filter(["Note", "Browser"])

# Text search across title, text, description
f = create_text_filter("project deadline")

# Wildcard on title field
f = create_wildcard_filter("Sprint *", field="title")

# Combine (AND logic)
combined = combine_filters(create_text_filter("urgent"), create_widget_type_filter("Note"))
```

> **Note:** `combine_filters` currently implements AND logic by concatenating all conditions. OR logic across multiple filters is not natively supported; you would need to run separate queries and merge the results.

### Filter Logic

All conditions in a single `Filter` object must match for an item to be included (AND logic). The `matches(item)` method converts a Pydantic model to a dict and evaluates each condition in order, short-circuiting on the first non-match.

---

## Cross-Canvas Search (search.py)

`CrossCanvasSearch` is a higher-level abstraction built on top of `CanvusClient`. It iterates over multiple canvases, collects their widgets, and applies filters to find matching content anywhere on the server.

### Class Interface

```python
from canvus_api.search import CrossCanvasSearch

searcher = CrossCanvasSearch(client)
```

### Finding Widgets

```python
# By text content
results = await searcher.find_widgets_by_text("budget review")

# By type
results = await searcher.find_widgets_by_type("Browser")

# By spatial area (same rectangle across all canvases)
from canvus_api.geometry import Rectangle
area = Rectangle(x=0, y=0, width=2000, height=2000)
results = await searcher.find_widgets_in_area(area)

# By property value (supports dot notation)
results = await searcher.find_widgets_by_property("location.x", 100)

# Complex query (dict or JSON string)
results = await searcher.find_widgets_across_canvases(
    query={"widget_type": "Note", "text": "*urgent*"},
    canvas_ids=["canvas-1", "canvas-2"],  # Limit scope; None = all canvases
    widget_types=["Note"],
    spatial_filter=area,
    max_results=50,
    include_deleted=False,
)
```

### SearchResult Fields

Each result is a `SearchResult` dataclass:

| Field | Type | Description |
|-------|------|-------------|
| `canvas_id` | `str` | Canvas containing the widget |
| `canvas_name` | `str` | Display name of the canvas |
| `widget_id` | `str` | Widget ID |
| `widget_type` | `str` | Widget type string |
| `widget` | `BaseWidget` | Full typed widget object |
| `match_score` | `float` | 0.0–1.0 relevance score |
| `match_reason` | `str` | Human-readable explanation |
| `drill_down_path` | `str` (property) | `"{canvas_id}:{widget_id}"` — unique address |

```python
results = await searcher.find_widgets_by_text("launch checklist")
for r in results:
    print(f"{r.canvas_name} → {r.widget_type} [{r.drill_down_path}]")
    print(f"  Score: {r.match_score:.2f}  Reason: {r.match_reason}")
```

### Module-Level Convenience Functions

The module also exports free functions that create a `CrossCanvasSearch` internally:

```python
from canvus_api.search import (
    find_widgets_across_canvases,
    find_widgets_by_text,
    find_widgets_by_type,
    find_widgets_in_area,
)

results = await find_widgets_by_text(client, "deadline")
```

### Performance Considerations

`CrossCanvasSearch` fetches all widgets from every canvas in scope. On a server with many large canvases this can be slow. Limit the search scope with `canvas_ids` and `max_results` when possible:

```python
# Only search canvases the user is actively working with
results = await searcher.find_widgets_across_canvases(
    "kickoff",
    canvas_ids=["canvas-a", "canvas-b"],
    max_results=20,
)
```

Errors fetching individual canvases are logged and skipped, so a partial access failure does not abort the entire search.

---

## Export and Import (export.py)

`WidgetExporter` and `WidgetImporter` enable round-trip serialization: exporting widgets to a directory on disk and re-importing them into any canvas.

### Export Architecture

An export creates a directory with the following structure:

```
canvus_export_{canvas_id}_{timestamp}/
├── manifest.json          # Index of everything exported
├── widgets/
│   ├── {widget_id}.json   # One file per widget with full data
│   └── ...
├── assets/                # Downloaded binary files (images, videos, PDFs)
│   ├── image_{id}.jpg
│   └── ...
└── metadata/              # Reserved for future use
```

The `manifest.json` records the export timestamp, canvas metadata, and configuration.

### ExportConfig

```python
from canvus_api.export import ExportConfig

config = ExportConfig(
    include_assets=True,           # Download image/video/PDF files
    include_spatial_data=True,     # Include position and size data
    include_metadata=True,         # Include widget metadata
    asset_format="original",       # "original", "compressed", or "web"
    export_path="/tmp/exports",    # Base directory
    overwrite_existing=False,      # Fail if directory already exists
)
```

### Exporting Widgets

```python
from canvus_api.export import WidgetExporter, ExportConfig

config = ExportConfig(export_path="/backup")
exporter = WidgetExporter(client, config)

# Export all widgets from a canvas
export_path = await exporter.export_widgets_to_folder(canvas_id)

# Export specific widgets only
export_path = await exporter.export_widgets_to_folder(
    canvas_id,
    widget_ids=["uuid-1", "uuid-2"],
    folder_path="/tmp/partial_export",
)

print(f"Export saved to: {export_path}")
```

Using the convenience function:

```python
from canvus_api.export import export_widgets_to_folder

path = await export_widgets_to_folder(
    client, canvas_id, folder_path="/tmp/backup"
)
```

### ImportConfig

```python
from canvus_api.export import ImportConfig

config = ImportConfig(
    import_assets=True,            # Re-upload asset files
    restore_spatial_data=True,     # Restore positions and parent relationships
    restore_metadata=True,
    target_canvas_id="canvas-xyz", # Default target canvas
    spatial_offset={"x": 100, "y": 0},  # Shift all positions on import
    preserve_ids=False,            # Assign new IDs (the server always does this)
)
```

### Importing Widgets

```python
from canvus_api.export import WidgetImporter, ImportConfig

config = ImportConfig(target_canvas_id=new_canvas_id)
importer = WidgetImporter(client, config)

result = await importer.import_widgets_from_folder("/backup/canvus_export_abc_20260101_120000")
print(f"Imported {result['imported_count']} widgets")
print(f"ID mapping: {result['id_mapping']}")  # {old_id: new_id}
```

### Import Behavior by Widget Type

| Widget Type | Behavior |
|-------------|----------|
| `Note`, `Browser`, `Anchor`, `Connector` | Re-created via the type-specific create endpoint |
| `Image`, `Video`, `PDF` | Re-uploaded using the asset file from the export directory. If the asset file is missing, the widget is skipped with a warning. |
| All others | Created via the generic `create_widget` endpoint |

Fields that the server manages (`id`, `canvas_id`, `created_at`, etc.) are stripped from the import payload before the create call.

---

## Widget Zone Operations (widget_operations.py)

This module provides spatial grouping and batch operation utilities that work locally on lists of widget objects — no API calls required.

### WidgetZoneManager

A zone is a named rectangular region of the canvas. `WidgetZoneManager` creates `WidgetZone` objects and queries which widgets fall inside them.

```python
from canvus_api.widget_operations import WidgetZoneManager

manager = WidgetZoneManager()
widgets = await client.list_widgets(canvas_id)

# Create a zone that encompasses a group of widgets
intro_zone = manager.create_zone_from_widgets(
    [widget_a, widget_b, widget_c],
    name="Introduction Section",
    description="Slides 1-5",
    padding=20.0,  # Add 20px around the group bounding box
)

# Find widgets inside the zone
inside = manager.widgets_in_zone(widgets, intro_zone)

# Find widgets that touch or overlap the zone boundary
touching = manager.widgets_touching_zone(widgets, intro_zone)
```

### BatchWidgetOperations

Generates update payloads for applying the same operation to many widgets. The generated payloads must be applied individually with `client.update_widget()` or the type-specific update methods.

```python
from canvus_api.widget_operations import BatchWidgetOperations

batch = BatchWidgetOperations()
widgets_to_move = [note_a, note_b, image_c]

# Generate move operations (does not call the API)
ops = batch.move_widgets(widgets_to_move, offset_x=100, offset_y=0)
for op in ops:
    await client.update_widget(canvas_id, op["widget_id"], op["payload"])

# Generate resize operations
ops = batch.resize_widgets(widgets_to_move, scale_factor=1.5)
for op in ops:
    await client.update_widget(canvas_id, op["widget_id"], op["payload"])
```

For `Connector` objects, `move_widgets` adjusts the `src` and `dst` endpoint locations instead of a top-level `location` field. `resize_widgets` scales the `line_width`.

### Spatial Grouping Functions

```python
from canvus_api.widget_operations import (
    create_spatial_group,
    find_widget_clusters,
    calculate_widget_density,
)

widgets = await client.list_widgets(canvas_id)

# Group widgets by proximity (widgets within 10px of each other are grouped)
groups = create_spatial_group(widgets, tolerance=10.0)
print(f"Found {len(groups)} spatial groups")

# Find clusters with at least 3 widgets within 20px of each other
clusters = find_widget_clusters(widgets, min_cluster_size=3, tolerance=20.0)

# Calculate widget density in a region
from canvus_api.geometry import Rectangle
region = Rectangle(x=0, y=0, width=1920, height=1080)
density = calculate_widget_density(widgets, region)
print(f"Density: {density:.6f} widgets per square pixel")
```

### SpatialTolerance Configuration

Both `WidgetZoneManager` and `BatchWidgetOperations` accept a `SpatialTolerance` dataclass that controls precision thresholds:

```python
from canvus_api.widget_operations import WidgetZoneManager, SpatialTolerance

tolerance = SpatialTolerance(
    position_tolerance=5.0,   # Position snap threshold
    size_tolerance=2.0,       # Size comparison threshold
    overlap_tolerance=1.0,    # Minimum overlap area for intersection
    distance_tolerance=10.0,  # Proximity threshold for grouping
)
manager = WidgetZoneManager(tolerance=tolerance)
```

---

## Design Patterns and Extension Points

### Composing Higher-Level Tools

All of `search.py`, `export.py`, and `widget_operations.py` accept a `CanvusClient` at construction time and work with standard model objects. You can compose them:

```python
from canvus_api.search import CrossCanvasSearch
from canvus_api.export import export_widgets_to_folder
from canvus_api.geometry import Rectangle

async with CanvusClient(base_url=URL, api_key=TOKEN) as client:
    searcher = CrossCanvasSearch(client)

    # Find all notes in a region across all canvases
    area = Rectangle(x=0, y=0, width=2000, height=2000)
    results = await searcher.find_widgets_in_area(area, widget_types=["Note"])

    # Group them by canvas
    canvases_found = {r.canvas_id for r in results}

    # Export each canvas for backup
    for canvas_id in canvases_found:
        path = await export_widgets_to_folder(
            client, canvas_id, folder_path="/tmp/backup"
        )
        print(f"Backed up {canvas_id} to {path}")
```

### Extending with Custom Filtering

`Filter.matches()` accepts any `Dict[str, Any]`. You can apply it to arbitrary data beyond widgets and canvases:

```python
from canvus_api.filters import Filter

f = Filter().add_condition("status", "equals", "active")

my_records = [{"id": 1, "status": "active"}, {"id": 2, "status": "inactive"}]
active = [r for r in my_records if f.matches(r)]
```

### Real-Time + Spatial Composition

Combine streaming with spatial queries for reactive applications:

```python
from canvus_api.geometry import Rectangle, widget_bounding_box, intersects

hotzone = Rectangle(x=500, y=500, width=400, height=300)

async for update in client.subscribe(f"canvases/{canvas_id}/widgets", response_model=Widget):
    try:
        bounds = widget_bounding_box(update)
        if intersects(bounds, hotzone):
            print(f"Widget {update.id} entered or moved within the hot zone")
    except ValueError:
        pass  # Connector or widget without bounds
```

### Stateless vs. Stateful Modules

- `geometry.py`, `filters.py`, and `widget_operations.py` are **stateless** — they operate on data passed as arguments and return results without making API calls or storing state between calls.
- `search.py` and `export.py` are **stateful** — `CrossCanvasSearch` holds a reference to the client, and `WidgetExporter`/`WidgetImporter` accumulate manifest data across multiple widget export calls.
- `CanvusClient` itself is **stateful** — it holds the connection configuration and session handle.
