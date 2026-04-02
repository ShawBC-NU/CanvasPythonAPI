# API Reference

Complete method signatures for all operations exposed by `CanvusClient`. Methods are grouped by resource category. Every method is `async` and must be awaited inside an `async with CanvusClient(...) as client:` block.

---

## What's New in Canvus 3.5

This section summarizes API changes introduced in Canvus 3.5. Items marked with *(New in Canvus 3.5)* throughout this document indicate features from this release.

### New Widget Types

| Widget Type | Description | Endpoint |
|-------------|-------------|----------|
| **Table** | Grid-based container with cells for tabular layouts | `/canvases/{id}/tables` |
| **TableCell** | Individual cell within a Table widget | `/canvases/{id}/tables/{tableId}/cells` |
| **IpVideo** | Live network video streams (RTSP, RTMP) | `/canvases/{id}/ip-videos` |
| **RdpConnection** | Remote Desktop Protocol sessions | `/canvases/{id}/rdp-connections` |

### New Attributes

| Widget | Attribute | Type | Description |
|--------|-----------|------|-------------|
| Video | `muted` | `bool` | Whether audio is muted (default `false`) |
| Video | `duration` | `float` | Duration in seconds (read-only, populated by client decoder) |

### New Parameters

| Parameter | Applies To | Description |
|-----------|------------|-------------|
| `auto_raise` | All PATCH endpoints | When `true`, sets widget depth above all siblings |

### Breaking Changes

1. **Depth validation enforced** — PATCH requests setting `depth` below 1.0 now return `400 Bad Request`
2. **Image/Video resize enforces aspect ratio** — Size changes use longest-edge scaling; read the response for actual applied size
3. **Video pause preserves position** — Pausing without explicit `playback_position` now computes correct current position

---

## Table of Contents

- [Client Initialization](#client-initialization)
- [Server Operations](#server-operations)
- [Canvas Operations](#canvas-operations)
- [Folder Operations](#folder-operations)
- [Widget Operations (Generic)](#widget-operations-generic)
- [Notes](#notes)
- [Images](#images)
- [Browsers](#browsers)
- [Videos](#videos)
- [PDFs](#pdfs)
- [Anchors](#anchors)
- [Connectors](#connectors)
- [Tables](#tables) *(New in Canvus 3.5)*
- [IP Videos](#ip-videos) *(New in Canvus 3.5)*
- [RDP Connections](#rdp-connections) *(New in Canvus 3.5)*
- [Canvas Background](#canvas-background)
- [Color Presets](#color-presets)
- [Canvas Permissions](#canvas-permissions)
- [Demo Canvas Operations](#demo-canvas-operations)
- [Uploads Folder](#uploads-folder)
- [Video Inputs](#video-inputs)
- [Video Outputs](#video-outputs)
- [Streaming and Subscriptions](#streaming-and-subscriptions)
- [User Management](#user-management)
- [Authentication and Sessions](#authentication-and-sessions)
- [Access Tokens](#access-tokens)
- [Group Management](#group-management)
- [Workspace Operations](#workspace-operations)
- [Client Management](#client-management)
- [License Management](#license-management)
- [Audit Log](#audit-log)
- [Mipmap Operations](#mipmap-operations)

---

## Client Initialization

```python
CanvusClient(
    base_url: str,
    api_key: str,
    verify_ssl: bool = True,
    max_retries: int = 3,
    retry_delay: float = 1.0,
    retry_backoff: float = 2.0,
    timeout: float = 30.0,
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `base_url` | `str` | required | Base URL of the Canvus server, e.g. `"https://canvus.example.com"`. Trailing slashes are stripped automatically. |
| `api_key` | `str` | required | API key (Private Token) for authentication. Sent as `Private-Token` header on every request. |
| `verify_ssl` | `bool` | `True` | Whether to verify SSL certificates. Set to `False` only for development against servers with self-signed certificates. |
| `max_retries` | `int` | `3` | Maximum number of additional attempts after a transient failure. Retryable conditions: connection errors, HTTP 408, 429, and 5xx (except 501). |
| `retry_delay` | `float` | `1.0` | Seconds to wait before the first retry. |
| `retry_backoff` | `float` | `2.0` | Multiplier applied to the delay after each retry (exponential backoff). |
| `timeout` | `float` | `30.0` | Per-request timeout in seconds. Streaming connections use a separate, unlimited total timeout with a 60-second socket-read timeout. |

**Usage pattern:**

```python
async with CanvusClient(base_url=URL, api_key=TOKEN) as client:
    # All method calls go here
    ...
```

---

## Server Operations

### `get_server_info() -> ServerInfo`

Returns version, API support list, server ID, and Go runtime version.

```python
info = await client.get_server_info()
# info.version, info.api (list of supported API versions), info.server_id, info.go
```

### `get_server_config() -> ServerConfig`

Returns the current server configuration, including feature flags, authentication settings, and external URL.

```python
config = await client.get_server_config()
```

### `update_server_config(payload: Dict[str, Any]) -> ServerConfig`

Applies a partial update to the server configuration. Only the fields present in `payload` are changed.

```python
config = await client.update_server_config({
    "server_name": "Production Canvus",
    "external_url": "https://canvus.example.com",
})
```

### `send_test_email() -> Dict[str, Any]`

Sends a test email to verify that email delivery is configured correctly.

```python
result = await client.send_test_email()
```

---

## Canvas Operations

### `list_canvases(params=None, filter_obj=None) -> List[Canvas]`

Returns all canvases accessible to the authenticated user.

| Parameter | Type | Description |
|-----------|------|-------------|
| `params` | `Dict[str, Any] \| str \| None` | Query parameters forwarded to the API, e.g. `{"folder_id": "abc"}`. Accepts a JSON string or a dict. |
| `filter_obj` | `Filter \| None` | Client-side filter applied after the API response is received. See [Filtering](architecture.md#filtering). |

```python
canvases = await client.list_canvases()
# With a filter object:
from canvus_api.filters import Filter
f = Filter().add_condition("name", "starts_with", "Project")
canvases = await client.list_canvases(filter_obj=f)
```

### `get_canvas(canvas_id: str) -> Canvas`

Returns a single canvas by ID.

```python
canvas = await client.get_canvas("canvas-uuid")
```

### `create_canvas(payload: Dict[str, Any]) -> Canvas`

Creates a new canvas.

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | `str` | yes | Display name of the canvas |
| `folder_id` | `str` | yes | ID of the parent folder |
| `description` | `str` | no | Optional description |
| `mode` | `str` | no | `"normal"` (default) or `"demo"` |

```python
canvas = await client.create_canvas({"name": "Sprint 12", "folder_id": root_id})
```

### `update_canvas(canvas_id: str, payload: Dict[str, Any]) -> Canvas`

Applies a partial update to a canvas. Supports the same keys as `create_canvas`.

```python
canvas = await client.update_canvas(canvas_id, {"name": "Sprint 12 — Completed"})
```

### `move_canvas(canvas_id: str, folder_id: str) -> Canvas`

Moves a canvas to a different folder.

```python
canvas = await client.move_canvas(canvas_id, destination_folder_id)
```

### `copy_canvas(canvas_id: str, payload: Dict[str, Any]) -> Canvas`

Creates a full copy of a canvas.

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | `str` | yes | Name for the copy |
| `folder_id` | `str` | yes | Destination folder ID |

```python
copy = await client.copy_canvas(canvas_id, {"name": "Copy", "folder_id": folder_id})
```

### `get_canvas_permissions(canvas_id: str) -> Dict[str, Any]`

Returns the permission configuration for a canvas.

### `get_canvas_preview(canvas_id: str) -> bytes`

Returns binary PNG preview image of the canvas.

```python
data = await client.get_canvas_preview(canvas_id)
with open("preview.png", "wb") as f:
    f.write(data)
```

### `delete_canvas(canvas_id: str) -> None`

Permanently deletes a canvas and all its widgets.

```python
await client.delete_canvas(canvas_id)
```

---

## Folder Operations

### `list_folders(params=None) -> List[CanvasFolder]`

Returns all canvas folders.

### `get_folder(folder_id: str) -> CanvasFolder`

Returns a single folder by ID.

### `create_folder(payload: JsonData) -> CanvasFolder`

Creates a new folder.

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | `str` | yes | Folder display name |
| `folder_id` | `str` | yes | ID of the parent folder |

```python
folder = await client.create_folder({"name": "Q1 Projects", "folder_id": root_id})
```

### `update_folder(folder_id: str, payload: JsonData) -> CanvasFolder`

Applies a partial update to a folder.

### `move_folder(folder_id: str, payload: JsonData) -> CanvasFolder`

Moves a folder to a different parent.

```python
folder = await client.move_folder(folder_id, {"folder_id": new_parent_id})
```

### `copy_folder(folder_id: str, payload: Dict[str, Any]) -> CanvasFolder`

Creates a copy of a folder.

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `folder_id` | `str` | yes | Destination parent folder ID |
| `name` | `str` | no | Name for the copy |

### `delete_folder(folder_id: str) -> None`

Deletes a folder. The folder must be empty.

### `delete_folder_children(folder_id: str) -> None`

Deletes all children (canvases and sub-folders) of a folder.

---

## Widget Operations (Generic)

The generic widget endpoints operate on any widget type and return `Widget` objects. Use type-specific methods (Notes, Images, etc.) when you need strongly-typed models.

### `list_widgets(canvas_id: str, filter_obj=None) -> List[Widget]`

Returns all widgets in a canvas.

```python
widgets = await client.list_widgets(canvas_id)

# With a client-side filter:
from canvus_api.filters import Filter
f = Filter().add_condition("widget_type", "equals", "Note")
notes = await client.list_widgets(canvas_id, filter_obj=f)
```

### `get_widget(canvas_id: str, widget_id: str) -> Widget`

Returns a single widget by ID.

### `create_widget(canvas_id: str, payload: Dict[str, Any]) -> Widget`

Creates a new widget of any type. Use type-specific methods for images, videos, and PDFs (which require multipart uploads).

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `widget_type` | `str` | yes | Type identifier, e.g. `"Note"` |
| `location` | `{"x": float, "y": float}` | yes | Canvas position |
| `size` | `{"width": float, "height": float}` | yes | Widget dimensions |
| `config` | `Dict[str, Any]` | no | Type-specific configuration |
| `depth` | `float` | no | Z-order depth |
| `scale` | `float` | no | Scale factor (default `1.0`) |
| `pinned` | `bool` | no | Whether the widget is locked in place |
| `parent_id` | `str` | no | ID of the parent widget |

### `update_widget(canvas_id: str, widget_id: str, payload: Dict[str, Any]) -> Widget`

Applies a partial update to any widget. When `parent_id` is changed, the client automatically validates the hierarchy and adjusts `location` to preserve the widget's visual position.

**Common update parameters** *(auto_raise is New in Canvus 3.5)*:

| Key | Type | Description |
|-----|------|-------------|
| `auto_raise` | `bool` | When `true`, sets depth to `maxSiblingDepth + 1.0`, matching UI bring-to-front behavior |

```python
# Move a widget
widget = await client.update_widget(canvas_id, widget_id, {
    "location": {"x": 500, "y": 300},
})

# Reparent a widget (location is recalculated automatically)
widget = await client.update_widget(canvas_id, widget_id, {
    "parent_id": parent_widget_id,
})

# Bring widget to front (New in Canvus 3.5)
widget = await client.update_widget(canvas_id, widget_id, {
    "auto_raise": True,
})
```

> **Canvus 3.5 breaking change:** `depth` values below 1.0 now return `400 Bad Request`. Previously, values like -1 or 0 made widgets render behind the canvas background, permanently inaccessible from the UI.

### `delete_widget(canvas_id: str, widget_id: str) -> None`

Deletes a widget from the canvas.

### `list_widget_annotations(canvas_id: str) -> List[Dict[str, Any]]`

Returns annotation data for all widgets in a canvas.

### `clone_widgets(canvas_id: str, payload: Dict[str, Any]) -> List[Widget]` *(New in Canvus 3.5 - Stub)*

> **Note:** This endpoint is registered but returns `501 Not Implemented`. Full server-side cloning is planned for a future release.

Reserved for cross-canvas widget cloning.

---

## Notes

### `list_notes(canvas_id: str) -> List[Note]`

Returns all notes in a canvas as strongly-typed `Note` objects.

### `get_note(canvas_id: str, note_id: str) -> Note`

Returns a single note by ID.

### `create_note(canvas_id: str, payload: Dict[str, Any]) -> Note`

Creates a new note widget.

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `text` | `str` | yes | — | Main text content |
| `location` | `{"x": float, "y": float}` | yes | — | Canvas position |
| `size` | `{"width": float, "height": float}` | yes | — | Note dimensions |
| `title` | `str` | no | `None` | Optional header text |
| `text_color` | `str` | no | `"#000000ff"` | RGBA hex color for text |
| `background_color` | `str` | no | `"#ffffffff"` | RGBA hex color for background |
| `auto_text_color` | `bool` | no | `True` | Automatically adjust text color for readability |
| `depth` | `float` | no | `0.0` | Z-order depth |
| `scale` | `float` | no | `1.0` | Scale factor |
| `pinned` | `bool` | no | `False` | Whether the note is pinned |
| `parent_id` | `str` | no | `None` | Parent widget ID |

```python
note = await client.create_note(canvas_id, {
    "text": "Review this section",
    "location": {"x": 100, "y": 100},
    "size": {"width": 300, "height": 150},
    "background_color": "#ffffa0ff",
})
```

### `update_note(canvas_id: str, note_id: str, payload: Dict[str, Any]) -> Note`

Applies a partial update. Supports all fields from `create_note`. When `parent_id` changes, location is adjusted automatically.

### `delete_note(canvas_id: str, note_id: str) -> None`

---

## Images

### `list_images(canvas_id: str) -> List[Image]`

### `get_image(canvas_id: str, image_id: str) -> Image`

### `create_image(canvas_id: str, file_path: str, payload=None) -> Image`

Uploads an image file using multipart form data.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `canvas_id` | `str` | yes | Target canvas |
| `file_path` | `str` | yes | Local filesystem path to the image file |
| `payload` | `Dict \| str \| None` | no | Optional metadata: `location`, `size`, `title` |

```python
image = await client.create_image(
    canvas_id,
    "/path/to/diagram.png",
    {"location": {"x": 200, "y": 200}, "size": {"width": 800, "height": 600}},
)
```

> **Note:** This method requires the client to be inside the `async with` context manager because it uses the session for file streaming.

### `update_image(canvas_id: str, image_id: str, payload: JsonData) -> Image`

Applies a partial update. When `parent_id` changes, location is adjusted automatically.

> **Canvus 3.5 breaking change:** Changing `size` on Image widgets now enforces the current aspect ratio. The requested size is treated as a bounding box using longest-edge scaling. Read the response body to get the actual applied size.

### `delete_image(canvas_id: str, image_id: str) -> None`

### `download_image(canvas_id: str, image_id: str) -> bytes`

Returns the raw binary content of the image file.

---

## Browsers

### `list_browsers(canvas_id: str) -> List[Browser]`

### `get_browser(canvas_id: str, browser_id: str) -> Browser`

### `create_browser(canvas_id: str, payload: Dict[str, Any]) -> Browser`

Creates a browser widget that embeds a live web page.

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `url` | `str` | yes | — | URL for the embedded page |
| `location` | `{"x": float, "y": float}` | yes | — | Canvas position |
| `size` | `{"width": float, "height": float}` | yes | — | Browser dimensions |
| `title` | `str` | no | `None` | Display title |
| `transparent_mode` | `bool` | no | `False` | Render page with transparent background |
| `pinned` | `bool` | no | `False` | |
| `parent_id` | `str` | no | `None` | |

```python
browser = await client.create_browser(canvas_id, {
    "url": "https://grafana.example.com/d/overview",
    "location": {"x": 0, "y": 0},
    "size": {"width": 1200, "height": 800},
})
```

### `update_browser(canvas_id: str, browser_id: str, payload: Dict[str, Any]) -> Browser`

### `delete_browser(canvas_id: str, browser_id: str) -> None`

---

## Videos

### `list_videos(canvas_id: str) -> List[Video]`

### `get_video(canvas_id: str, video_id: str) -> Video`

### `create_video(canvas_id: str, file_path: str, payload=None) -> Video`

Uploads a video file using multipart form data.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `canvas_id` | `str` | yes | Target canvas |
| `file_path` | `str` | yes | Local path to the video file |
| `payload` | `Dict \| str \| None` | no | Optional metadata: `location`, `size`, `title` |

```python
video = await client.create_video(
    canvas_id,
    "/path/to/demo.mp4",
    {"location": {"x": 300, "y": 300}, "size": {"width": 640, "height": 360}},
)
```

### `update_video(canvas_id: str, video_id: str, payload: JsonData) -> Video`

> **Canvus 3.5 breaking change:** Changing `size` on Video widgets now enforces the current aspect ratio. The requested size is treated as a bounding box using longest-edge scaling. Read the response body to get the actual applied size.

Supports all standard widget fields plus:

| Key | Type | Description |
|-----|------|-------------|
| `playback_state` | `str` | `"STOPPED"`, `"PLAYING"`, or `"PAUSED"` |
| `playback_position` | `float` | Position in seconds |
| `muted` | `bool` | Whether audio is muted *(New in Canvus 3.5)* |

**Read-only attributes** *(New in Canvus 3.5)*:

| Key | Type | Description |
|-----|------|-------------|
| `duration` | `float` | Video duration in seconds (populated by desktop client decoder, 0.0 if not yet decoded) |

```python
await client.update_video(canvas_id, video.id, {"playback_state": "PLAYING"})

# Mute the video (New in Canvus 3.5)
await client.update_video(canvas_id, video.id, {"muted": True})
```

> **Canvus 3.5 behavior change:** When `playback_state` is set to `"PAUSED"` without providing `playback_position`, the server now computes the correct current position. Previously, pausing without an explicit position caused the video to jump back.

### `delete_video(canvas_id: str, video_id: str) -> None`

### `download_video(canvas_id: str, video_id: str) -> bytes`

---

## PDFs

### `list_pdfs(canvas_id: str) -> List[PDF]`

### `get_pdf(canvas_id: str, pdf_id: str) -> PDF`

### `create_pdf(canvas_id: str, file_path: str, payload=None) -> PDF`

Uploads a PDF file using multipart form data.

```python
pdf = await client.create_pdf(
    canvas_id,
    "/path/to/spec.pdf",
    {"location": {"x": 500, "y": 100}, "size": {"width": 600, "height": 800}},
)
```

### `update_pdf(canvas_id: str, pdf_id: str, payload: JsonData) -> PDF`

Supports all standard widget fields plus:

| Key | Type | Description |
|-----|------|-------------|
| `index` | `int` | Zero-based page number currently displayed |

```python
await client.update_pdf(canvas_id, pdf.id, {"index": 4})  # Show page 5
```

### `delete_pdf(canvas_id: str, pdf_id: str) -> None`

### `download_pdf(canvas_id: str, pdf_id: str) -> bytes`

---

## Anchors

### `list_anchors(canvas_id: str) -> List[Anchor]`

### `get_anchor(canvas_id: str, anchor_id: str) -> Anchor`

### `create_anchor(canvas_id: str, payload: JsonData) -> Anchor`

Creates a named navigation anchor.

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `anchor_name` | `str` | yes | `"New anchor"` | Display name |
| `anchor_index` | `int` | no | `0` | Position in the ordered anchor list |
| `location` | `{"x": float, "y": float}` | yes | — | Canvas position |
| `size` | `{"width": float, "height": float}` | yes | — | Anchor dimensions |

```python
anchor = await client.create_anchor(canvas_id, {
    "anchor_name": "Executive Summary",
    "anchor_index": 0,
    "location": {"x": 0, "y": 0},
    "size": {"width": 400, "height": 200},
})
```

### `update_anchor(canvas_id: str, anchor_id: str, payload: JsonData) -> Anchor`

When `parent_id` changes, location is adjusted automatically.

### `delete_anchor(canvas_id: str, anchor_id: str) -> None`

---

## Connectors

### `list_connectors(canvas_id: str) -> List[Connector]`

### `get_connector(canvas_id: str, connector_id: str) -> Connector`

### `create_connector(canvas_id: str, payload: Dict[str, Any]) -> Connector`

Creates a visual connection between two widgets.

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `src` | `ConnectorEndpoint` | yes | — | Source endpoint |
| `dst` | `ConnectorEndpoint` | yes | — | Destination endpoint |
| `line_color` | `str` | no | `"#e7e7f2ff"` | RGBA hex line color |
| `line_width` | `float` | no | `5.0` | Line thickness in pixels |
| `type` | `str` | no | `"curve"` | Line style (only `"curve"` supported) |

**ConnectorEndpoint fields:**

| Key | Type | Description |
|-----|------|-------------|
| `id` | `str` | ID of the widget to connect to |
| `rel_location` | `{"x": float, "y": float}` | Relative position on the widget (0.0–1.0) |
| `auto_location` | `bool` | Let the server choose the attachment point |
| `tip` | `str` | `"none"` or `"solid-equilateral-triangle"` |

```python
connector = await client.create_connector(canvas_id, {
    "src": {
        "id": widget_a_id,
        "rel_location": {"x": 1.0, "y": 0.5},  # Right-center
        "auto_location": False,
        "tip": "none",
    },
    "dst": {
        "id": widget_b_id,
        "rel_location": {"x": 0.0, "y": 0.5},  # Left-center
        "auto_location": False,
        "tip": "solid-equilateral-triangle",
    },
    "line_color": "#ff6600ff",
    "line_width": 4.0,
})
```

### `update_connector(canvas_id: str, connector_id: str, payload: Dict[str, Any]) -> Connector`

Circular parenting check is applied. Note that connectors do not have a `location` attribute, so position offsetting does not apply on reparent.

### `delete_connector(canvas_id: str, connector_id: str) -> None`

---

## Tables

> **New in Canvus 3.5.** Table widgets were added as part of the Canvus 3.5 API refactor.

Tables are grid-based container widgets on the canvas. Each table is a structured grid of cells where you can organize content into rows and columns. When you create a table, the server automatically creates the cell widgets to fill the grid.

### `list_tables(canvas_id: str) -> List[Table]`

Returns all tables on a canvas.

### `get_table(canvas_id: str, table_id: str) -> Table`

Returns a single table by ID.

### `create_table(canvas_id: str, payload: Dict[str, Any]) -> Table`

Creates a new table widget. The server automatically creates the grid cells based on `grid_size`.

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `location` | `{"x": float, "y": float}` | yes | — | Canvas position |
| `size` | `{"width": float, "height": float}` | no | server default | Table dimensions |
| `grid_size` | `{"rows": int, "columns": int}` | no | `{"rows": 2, "columns": 2}` | Grid dimensions |
| `title` | `str` | no | `None` | Display title |
| `depth` | `float` | no | `1.0` | Z-order depth (must be >= 1.0) |
| `pinned` | `bool` | no | `False` | Whether the table is pinned |

```python
table = await client.create_table(canvas_id, {
    "location": {"x": 100, "y": 100},
    "size": {"width": 600, "height": 400},
    "grid_size": {"rows": 3, "columns": 4},
    "title": "Comparison Matrix",
})
```

### `update_table(canvas_id: str, table_id: str, payload: Dict[str, Any]) -> Table`

Updates table properties. The `grid_size` cannot be changed after creation.

| Key | Type | Description |
|-----|------|-------------|
| `title` | `str` | Display title |
| `location` | `{"x": float, "y": float}` | Canvas position |
| `size` | `{"width": float, "height": float}` | Table dimensions |
| `pinned` | `bool` | Whether the table is pinned |
| `auto_raise` | `bool` | When `true`, sets depth above all siblings |

### `delete_table(canvas_id: str, table_id: str) -> None`

Permanently removes a table and all its cells. Any connectors attached to the table are also deleted.

### `list_table_cells(canvas_id: str, table_id: str) -> List[TableCell]`

Returns all cells belonging to a specific table. Each cell includes `row` and `column` position.

```python
cells = await client.list_table_cells(canvas_id, table_id)
for cell in cells:
    print(f"Cell at row {cell.row}, column {cell.column}")
```

### `list_all_table_cells(canvas_id: str) -> List[TableCell]`

Returns all table cells across all tables on a canvas in a single flat list.

---

## IP Videos

> **New in Canvus 3.5.** IP Video widgets were added as part of the Canvus 3.5 API refactor.

IP Video widgets embed live network video streams on the canvas. They display content from IP cameras, RTSP/RTMP streams, or any other network video source. The Canvus desktop client handles decoding and rendering — the server stores the widget's metadata and stream configuration.

### `list_ip_videos(canvas_id: str) -> List[IpVideo]`

Returns all IP Video widgets on a canvas.

### `get_ip_video(canvas_id: str, widget_id: str) -> IpVideo`

Returns a single IP Video widget by ID.

### `create_ip_video(canvas_id: str, payload: Dict[str, Any]) -> IpVideo`

Creates a new IP Video widget.

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `source` | `str` | yes | — | Stream URL (RTSP, RTMP, etc.) |
| `location` | `{"x": float, "y": float}` | yes | — | Canvas position |
| `size` | `{"width": float, "height": float}` | yes | — | Widget dimensions |
| `title` | `str` | no | `None` | Display title |
| `depth` | `float` | no | `1.0` | Z-order depth (must be >= 1.0) |
| `pinned` | `bool` | no | `False` | Whether the widget is pinned |

```python
ip_video = await client.create_ip_video(canvas_id, {
    "source": "rtsp://192.168.1.100:554/stream1",
    "location": {"x": 200, "y": 200},
    "size": {"width": 640, "height": 480},
    "title": "Lobby Camera",
})
```

### `update_ip_video(canvas_id: str, widget_id: str, payload: Dict[str, Any]) -> IpVideo`

Updates IP Video properties.

| Key | Type | Description |
|-----|------|-------------|
| `source` | `str` | New stream URL |
| `title` | `str` | Display title |
| `auto_raise` | `bool` | When `true`, sets depth above all siblings |

### `delete_ip_video(canvas_id: str, widget_id: str) -> None`

Permanently removes an IP Video widget. Any connectors attached to this widget are also deleted.

> **Note:** The `source` URL must be reachable from the Canvus desktop client, not the server. The `host_id` field (read-only) identifies which connected client is responsible for decoding the stream.

---

## RDP Connections

> **New in Canvus 3.5.** RDP Connection widgets were added as part of the Canvus 3.5 API refactor.

RDP Connection widgets embed live Remote Desktop Protocol sessions on the canvas. They allow Canvus users to interact with remote Windows (or other RDP-capable) machines directly from the collaborative canvas surface.

### `list_rdp_connections(canvas_id: str) -> List[RdpConnection]`

Returns all RDP Connection widgets on a canvas.

### `get_rdp_connection(canvas_id: str, widget_id: str) -> RdpConnection`

Returns a single RDP Connection widget by ID.

### `create_rdp_connection(canvas_id: str, payload: Dict[str, Any]) -> RdpConnection`

Creates a new RDP Connection widget.

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `connection_name` | `str` | yes | — | Hostname or IP address of remote machine |
| `location` | `{"x": float, "y": float}` | yes | — | Canvas position |
| `size` | `{"width": float, "height": float}` | yes | — | Widget dimensions |
| `title` | `str` | no | `None` | Display title |
| `depth` | `float` | no | `1.0` | Z-order depth (must be >= 1.0) |
| `pinned` | `bool` | no | `False` | Whether the widget is pinned |

```python
rdp = await client.create_rdp_connection(canvas_id, {
    "connection_name": "workstation.local",
    "location": {"x": 300, "y": 300},
    "size": {"width": 1280, "height": 720},
    "title": "Engineering Workstation",
})
```

### `update_rdp_connection(canvas_id: str, widget_id: str, payload: Dict[str, Any]) -> RdpConnection`

Updates RDP Connection properties.

| Key | Type | Description |
|-----|------|-------------|
| `connection_name` | `str` | Hostname or IP address |
| `title` | `str` | Display title |
| `auto_raise` | `bool` | When `true`, sets depth above all siblings |

### `delete_rdp_connection(canvas_id: str, widget_id: str) -> None`

Permanently removes an RDP Connection widget. The remote desktop session (if active) is terminated.

> **Note:** RDP authentication is handled by the Canvus desktop client at connection time. The API does not store or transmit RDP credentials. The `host_id` and `content_id` fields are read-only and managed by the system.

---

## Canvas Background

### `get_canvas_background(canvas_id: str) -> Dict[str, Any]`

Returns background configuration including type, color, and scale.

### `set_canvas_background(canvas_id: str, payload: Dict[str, Any]) -> Dict[str, Any]`

Updates background settings.

| Key | Type | Description |
|-----|------|-------------|
| `type` | `str` | Background type, e.g. `"color"` or `"image"` |
| `color` | `str` | RGBA hex color string |
| `opacity` | `float` | Opacity 0.0–1.0 |
| `scale` | `float` | Background scale factor |

```python
await client.set_canvas_background(canvas_id, {
    "type": "color",
    "color": "#1a1a2eff",
})
```

### `set_canvas_background_image(canvas_id: str, file_path: str) -> Dict[str, Any]`

Uploads a local image file and sets it as the canvas background.

```python
await client.set_canvas_background_image(canvas_id, "/path/to/background.jpg")
```

---

## Color Presets

### `get_color_presets(canvas_id: str) -> Dict[str, Any]`

Returns the custom color palette for a canvas.

### `update_color_presets(canvas_id: str, payload: Dict[str, Any]) -> Dict[str, Any]`

Updates the color palette.

---

## Canvas Permissions

### `get_canvas_permissions(canvas_id: str) -> Dict[str, Any]`

Returns permission overrides including per-user and per-group access levels.

### `set_canvas_permissions(canvas_id: str, payload: Dict[str, Any]) -> Dict[str, Any]`

Sets permission overrides on a canvas.

### `get_folder_permissions(folder_id: str) -> Dict[str, Any]`

Returns permission overrides for a folder, including `editors_can_share`, user permissions, and group permissions.

### `set_folder_permissions(folder_id: str, payload: Dict[str, Any]) -> Dict[str, Any]`

Sets permission overrides on a folder.

---

## Demo Canvas Operations

Demo mode allows a canvas to be reset to a saved state. This is useful for kiosk displays and repeatable demonstrations.

### `set_canvas_mode(canvas_id: str, is_demo: bool) -> Canvas`

Switches a canvas between normal and demo mode.

```python
await client.set_canvas_mode(canvas_id, is_demo=True)
```

### `save_demo_state(canvas_id: str) -> None`

Saves the current layout of a demo canvas. This becomes the state it resets to.

```python
await client.save_demo_state(canvas_id)
```

### `restore_demo_state(canvas_id: str) -> None`

Resets a demo canvas to its last saved state, discarding any changes made since.

```python
await client.restore_demo_state(canvas_id)
```

---

## Uploads Folder

The uploads folder is a staging area on the canvas. Uploaded items appear there before being placed.

### `upload_note(canvas_id: str, payload: JsonData) -> Dict[str, Any]`

Creates a note in the uploads folder.

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `upload_type` | `str` | yes | Must be `"note"` |
| `text` | `str` | no | Note text content |
| `title` | `str` | no | Note title |
| `background_color` | `str` | no | RGBA hex color |

### `upload_file(canvas_id: str, file_path: str, payload=None) -> Dict[str, Any]`

Uploads any supported file (image, video, PDF) to the uploads folder. File type is auto-detected by the server.

| Parameter | Type | Description |
|-----------|------|-------------|
| `file_path` | `str` | Local filesystem path |
| `payload` | `Dict \| None` | Optional: `title`, `original_filename`. If `upload_type` is present, it must be `"asset"` or omitted. |

```python
result = await client.upload_file(canvas_id, "/path/to/report.pdf")
```

---

## Video Inputs

Video inputs are live video streams placed on the canvas.

### `list_canvas_video_inputs(canvas_id: str) -> List[Dict[str, Any]]`

### `create_video_input(canvas_id: str, payload: Dict[str, Any]) -> Dict[str, Any]`

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | `str` | yes | Display name |
| `source` | `str` | yes | Video source identifier |
| `location` | `{"x": float, "y": float}` | yes | Canvas position |
| `size` | `{"width": float, "height": float}` | yes | Widget dimensions |
| `config` | `Dict` | no | Source-specific configuration |

### `delete_video_input(canvas_id: str, input_id: str) -> None`

---

## Video Outputs

Video outputs send canvas regions to external display hardware.

### `update_video_output(canvas_id: str, output_id: str, payload: Dict[str, Any]) -> Dict[str, Any]`

Updates a video output on a canvas.

| Key | Type | Description |
|-----|------|-------------|
| `name` | `str` | Output name |
| `enabled` | `bool` | Enable/disable output |
| `resolution` | `str` | Resolution string, e.g. `"1920x1080"` |
| `refresh_rate` | `int` | Refresh rate in Hz |
| `source` | `str` | Video source identifier |

---

## Streaming and Subscriptions

### `subscribe(endpoint, *, response_model=None, params=None, callback=None) -> AsyncGenerator`

Opens a long-lived NDJSON stream and yields updates as they arrive. The connection has no total timeout; it stays open until the caller breaks out of the loop or cancels the task.

| Parameter | Type | Description |
|-----------|------|-------------|
| `endpoint` | `str` | API path to subscribe to, e.g. `"canvases/{id}/widgets"` |
| `response_model` | `Type[T] \| None` | Pydantic model to parse each update into |
| `params` | `Dict \| None` | Additional query parameters |
| `callback` | `Callable \| None` | Function called for each update before yielding |

The method automatically appends `subscribe=true` to the query parameters.

```python
async for update in client.subscribe(
    f"canvases/{canvas_id}/widgets",
    response_model=Widget,
):
    print(f"{update.id}: {update.widget_type} at {update.location}")
```

```python
# Subscribe to all canvases
async for update in client.subscribe("canvases"):
    print(update)
```

### `subscribe_annotations(canvas_id, callback=None) -> AsyncGenerator[Dict[str, Any], None]`

Subscribes to real-time annotation updates for a canvas.

```python
async for annotation in client.subscribe_annotations(canvas_id):
    print(annotation)
```

---

## User Management

### `list_users() -> List[User]`

Returns all registered users. Requires admin privileges.

### `get_user(user_id: int) -> User`

Returns a single user by integer ID.

### `get_current_user() -> User`

Returns information about the currently authenticated user by validating the client's API key.

```python
me = await client.get_current_user()
print(f"Logged in as {me.name} ({me.email})")
```

### `create_user(payload: Dict[str, Any]) -> User`

Creates a new user. Requires admin privileges.

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `email` | `str` | yes | Email address |
| `name` | `str` | yes | Display name |
| `password` | `str` | no | Initial password |
| `admin` | `bool` | no | Grant admin role |
| `approved` | `bool` | no | Approve immediately (default `True`) |
| `blocked` | `bool` | no | Block immediately (default `False`) |

### `update_user(user_id: int, payload: Dict[str, Any]) -> User`

Applies a partial update to a user profile. Regular users can only update their own profile and only `email` and `name`. Admins can update all fields.

### `delete_user(user_id: int) -> None`

Permanently deletes a user. Requires admin privileges.

### `approve_user(user_id: int) -> User`

Approves a pending user registration. Requires admin privileges.

### `block_user(user_id: int) -> User`

Blocks a user from signing in.

### `unblock_user(user_id: int) -> User`

Unblocks a user. Requires admin privileges.

---

## Authentication and Sessions

### `login(email=None, password=None, token=None) -> Dict[str, Any]`

Authenticates and returns a session token plus user information.

```python
# Password login
result = await client.login(email="user@example.com", password="secret")
token = result["token"]

# Token validation
result = await client.login(token="existing-token")
```

### `login_saml() -> Dict[str, Any]`

Initiates SAML authentication. Returns redirect and SAML-specific data.

### `logout(token=None) -> None`

Invalidates a session token. If `token` is omitted, the client's own token is used.

### `register_user(payload: Dict[str, Any]) -> Dict[str, Any]`

Registers a new user account without requiring authentication. Requires the server to allow open registration.

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `email` | `str` | yes | |
| `name` | `str` | yes | |
| `password` | `str` | yes | |

### `confirm_email(token: str) -> Dict[str, Any]`

Confirms a user's email using the token from the confirmation email.

### `change_password(user_id: int, current_password: str, new_password: str) -> User`

### `request_password_reset(email: str) -> None`

Sends a password reset email.

### `validate_reset_token(token: str) -> Dict[str, Any]`

Checks whether a password reset token is valid without consuming it.

### `reset_password(token: str, new_password: str) -> Dict[str, Any]`

Sets a new password using a token from a reset email.

---

## Access Tokens

Access tokens are long-lived API keys associated with a user account. The plain token value is only returned when the token is first created.

### `list_tokens(user_id: int) -> List[AccessToken]`

Lists access tokens for a user. Token values are not returned. Regular users can only list their own tokens; admins can list any user's tokens.

### `get_token(user_id: int, token_id: str) -> AccessToken`

Returns metadata for a single token (not the token value).

### `create_token(user_id: int, description: str) -> TokenResponse`

Creates a new access token. The `plain_token` field of the response contains the actual token string — save it immediately, as it cannot be retrieved again.

```python
token_response = await client.create_token(user_id=42, description="CI Pipeline Key")
print(f"Token: {token_response.plain_token}")  # Save this now
```

### `update_token(user_id: int, token_id: str, description: str) -> AccessToken`

Updates the description of an existing token.

### `delete_token(user_id: int, token_id: str) -> None`

Revokes and deletes an access token. The token can no longer be used for authentication.

---

## Group Management

Groups allow permissions to be assigned to sets of users.

### `list_groups() -> List[Dict[str, Any]]`

### `get_group(group_id: str) -> Dict[str, Any]`

### `create_group(payload: Dict[str, Any]) -> Dict[str, Any]`

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | `str` | yes | Group name |
| `description` | `str` | no | Group description |

```python
group = await client.create_group({"name": "Engineering", "description": "Eng team"})
```

### `delete_group(group_id: str) -> None`

### `list_group_members(group_id: str) -> List[Dict[str, Any]]`

### `add_user_to_group(group_id: str, user_id: str) -> Dict[str, Any]`

Adds a user to a group.

```python
await client.add_user_to_group(group_id="eng-group", user_id="42")
```

### `remove_user_from_group(group_id: str, user_id: str) -> None`

---

## Workspace Operations

A workspace is a client application's viewport into a canvas. Each connected client can have multiple workspaces.

### `list_workspaces(client_id: str) -> List[Workspace]`

Returns all workspaces for a connected client.

### `get_workspace(client_id: str, workspace_index: int) -> Workspace`

Returns a specific workspace by index.

### `update_workspace(client_id: str, workspace_index: int, payload: Dict[str, Any]) -> Workspace`

Updates workspace parameters.

| Key | Type | Description |
|-----|------|-------------|
| `info_panel_visible` | `bool` | Whether the info panel is shown |
| `pinned` | `bool` | Whether the workspace is pinned |
| `view_rectangle` | `{"x": float, "y": float, "width": float, "height": float}` | The visible region in canvas coordinates |

```python
# Scroll the workspace to show a specific area
await client.update_workspace(client_id, 0, {
    "view_rectangle": {"x": 0, "y": 0, "width": 1920, "height": 1080}
})
```

---

## Client Management

A "client" in the Canvus context is a connected Canvus application instance (e.g. a Canvus display node).

### `list_clients() -> List[Dict[str, Any]]`

Returns all currently connected clients.

### `get_client(client_id: str) -> Dict[str, Any]`

Returns a specific client by ID.

### `list_client_video_inputs(client_id: str) -> List[Dict[str, Any]]`

Returns video input devices available on a specific client.

### `list_client_video_outputs(client_id: str) -> List[Dict[str, Any]]`

Returns video output devices on a specific client.

### `set_video_output_source(client_id: str, index: int, payload: Dict[str, Any]) -> Dict[str, Any]`

Configures the source for a specific video output.

| Key | Type | Description |
|-----|------|-------------|
| `source` | `str` | Video source identifier |
| `enabled` | `bool` | Enable/disable the output |
| `resolution` | `str` | e.g. `"1920x1080"` |
| `refresh_rate` | `int` | Hz |

---

## License Management

### `get_license_info() -> Dict[str, Any]`

Returns license status, edition, feature flags, and limits.

```python
license = await client.get_license_info()
print(license["status"], license.get("expiry_date"))
```

### `request_offline_activation(key: str) -> Dict[str, Any]`

Generates an offline activation request for a license key.

```python
request_data = await client.request_offline_activation("AAAA-BBBB-CCCC-DDDD")
```

### `install_offline_license(license_data: str) -> Dict[str, Any]`

Installs a license from offline activation data.

```python
result = await client.install_offline_license(activation_response_string)
```

---

## Audit Log

### `get_audit_log(filters=None) -> Dict[str, Any]`

Returns paginated audit events sorted newest-first.

| Filter Key | Type | Description |
|------------|------|-------------|
| `created_after` | `str` | ISO 8601 timestamp |
| `created_before` | `str` | ISO 8601 timestamp |
| `target_type` | `str` | Resource type filter |
| `target_id` | `str` | Specific resource ID |
| `author_id` | `str` | User who triggered the event |
| `per_page` | `int` | Page size |
| `cursor` | `int` | Pagination offset from `Link` header |

```python
events = await client.get_audit_log({
    "author_id": "42",
    "created_after": "2026-01-01T00:00:00Z",
})
```

### `export_audit_log_csv(filters=None) -> bytes`

Exports audit log events as CSV binary data. Accepts the same filters as `get_audit_log`.

```python
csv_bytes = await client.export_audit_log_csv()
with open("audit_log.csv", "wb") as f:
    f.write(csv_bytes)
```

---

## Mipmap Operations

Mipmaps are pre-rendered resolution levels of image assets used for efficient WebGL rendering.

### `get_mipmap_info(public_hash_hex: str, canvas_id: str, page=None) -> Dict[str, Any]`

Returns resolution, max mipmap level, and page count for an asset.

| Parameter | Type | Description |
|-----------|------|-------------|
| `public_hash_hex` | `str` | Asset hash from `Image.hash` or `PDF.hash` |
| `canvas_id` | `str` | Canvas ID for access control |
| `page` | `int \| None` | Page number for multi-page assets (zero-based) |

```python
info = await client.get_mipmap_info(image.hash, canvas_id)
print(info["resolution"], info["max_level"])
```

### `get_mipmap_level_image(public_hash_hex: str, level: int, canvas_id: str, page=None) -> bytes`

Returns a specific mipmap level as WebP image bytes. Level 0 is the original resolution.

```python
# Get a thumbnail (level 3 = significantly downscaled)
thumbnail = await client.get_mipmap_level_image(image.hash, 3, canvas_id)
with open("thumbnail.webp", "wb") as f:
    f.write(thumbnail)
```
