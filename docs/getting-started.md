# Getting Started with canvus_api

This guide walks you through everything you need to go from zero to a working integration with the Canvus API. By the end you will have installed the library, made your first API call, created widgets, uploaded files, and set up real-time streaming.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Your First API Call](#your-first-api-call)
- [Working with Canvases](#working-with-canvases)
- [Creating Widgets](#creating-widgets)
- [Uploading Files](#uploading-files)
- [Real-Time Streaming](#real-time-streaming)
- [Error Handling](#error-handling)
- [Next Steps](#next-steps)

---

## Prerequisites

- Python 3.8 or higher
- A running Canvus server (on-premises or hosted)
- An API key (also called a Private Token) from your Canvus account settings

To generate an API key, sign in to the Canvus web interface, go to your profile settings, and create a new access token. Copy the token value — it is only shown once.

---

## Installation

The library depends on three packages. Install them with pip:

```bash
pip install aiohttp pydantic aiofiles
```

Then clone or copy the `canvus_api` package into your project, or install it from the repository root:

```bash
# From the repository root
pip install -e .
```

Verify the import works:

```python
from canvus_api import CanvusClient
```

---

## Your First API Call

Every interaction with the library follows the same pattern: create a `CanvusClient` inside an `async with` block, then call methods on it.

```python
import asyncio
from canvus_api import CanvusClient

async def main():
    async with CanvusClient(
        base_url="https://canvus.example.com",  # Your Canvus server URL
        api_key="your-private-token-here",       # API key from account settings
    ) as client:
        # Check that the server is reachable and the token works
        info = await client.get_server_info()
        print(f"Connected to Canvus {info.version}")

asyncio.run(main())
```

The `async with` block handles opening and closing the HTTP session automatically. You should never call `CanvusClient` without this context manager.

### Connection Options

The constructor accepts several options for customizing connection behavior:

```python
async with CanvusClient(
    base_url="https://canvus.example.com",
    api_key="your-private-token-here",
    verify_ssl=True,       # Set False only for self-signed certs in dev
    max_retries=3,         # Retry transient failures up to 3 times
    retry_delay=1.0,       # Wait 1 second before first retry
    retry_backoff=2.0,     # Double the delay after each retry
    timeout=30.0,          # Abort requests that take longer than 30 seconds
) as client:
    ...
```

> **Note:** Disabling SSL verification (`verify_ssl=False`) is intended only for local development against servers with self-signed certificates. Always use `verify_ssl=True` in production.

---

## Working with Canvases

A canvas is the top-level workspace where all widgets live. Canvases are organized into folders.

### Listing Canvases

```python
async with CanvusClient(base_url=URL, api_key=TOKEN) as client:
    canvases = await client.list_canvases()

    for canvas in canvases:
        print(f"{canvas.id}  {canvas.name}  ({canvas.state})")
```

### Getting a Specific Canvas

```python
canvas = await client.get_canvas("canvas-uuid-here")
print(canvas.name, canvas.mode)  # mode is "normal" or "demo"
```

### Creating a Canvas

```python
new_canvas = await client.create_canvas({
    "name": "My Project Canvas",
    "folder_id": "root-folder-id",  # ID of the parent folder
})
print(f"Created canvas: {new_canvas.id}")
```

### Updating a Canvas

```python
updated = await client.update_canvas(canvas_id, {
    "name": "Updated Name",
    "description": "A description of the project",
})
```

### Copying and Moving Canvases

```python
# Copy to the same folder with a new name
copy = await client.copy_canvas(canvas_id, {
    "name": "Canvas Copy",
    "folder_id": target_folder_id,
})

# Move to a different folder
moved = await client.move_canvas(canvas_id, new_folder_id)
```

### Deleting a Canvas

```python
await client.delete_canvas(canvas_id)
```

---

## Creating Widgets

Widgets are the objects on a canvas: notes, images, videos, PDFs, browsers, anchors, connectors, and more. Each widget type has dedicated methods.

> **Canvus 3.5** introduced three new widget types: **Tables** (grid-based containers), **IP Videos** (live network streams), and **RDP Connections** (remote desktop sessions). See the [API Reference](api-reference.md#whats-new-in-canvus-35) for details.

### Coordinates and Sizes

All positions and dimensions use canvas-coordinate space as plain Python dictionaries:

```python
location = {"x": 100.0, "y": 200.0}
size     = {"width": 400.0, "height": 300.0}
```

### Creating a Note

Notes are the simplest widget type — a box containing styled text.

```python
note = await client.create_note(canvas_id, {
    "text": "Hello from the API!",
    "title": "My First Note",           # Optional header
    "location": {"x": 100, "y": 100},
    "size": {"width": 300, "height": 200},
    "background_color": "#ffff99ff",    # RGBA hex string
    "text_color": "#000000ff",
})
print(f"Note created: {note.id}")
```

### Updating a Note

```python
updated_note = await client.update_note(canvas_id, note.id, {
    "text": "Updated content",
    "background_color": "#99ff99ff",
})
```

### Deleting a Note

```python
await client.delete_note(canvas_id, note.id)
```

### Creating a Browser Widget

A browser widget embeds a live web page on the canvas.

```python
browser = await client.create_browser(canvas_id, {
    "url": "https://www.example.com",
    "title": "Live Dashboard",
    "location": {"x": 500, "y": 100},
    "size": {"width": 800, "height": 600},
})
```

### Creating an Anchor

Anchors are named navigation points — they mark important regions of the canvas.

```python
anchor = await client.create_anchor(canvas_id, {
    "anchor_name": "Introduction",
    "anchor_index": 0,                  # Order in the anchor list
    "location": {"x": 0, "y": 0},
    "size": {"width": 200, "height": 100},
})
```

### Creating a Connector

Connectors draw a line between two existing widgets.

```python
connector = await client.create_connector(canvas_id, {
    "src": {
        "id": source_widget_id,
        "rel_location": {"x": 0.5, "y": 1.0},  # Bottom-center of source
        "auto_location": False,
        "tip": "none",
    },
    "dst": {
        "id": dest_widget_id,
        "rel_location": {"x": 0.5, "y": 0.0},  # Top-center of destination
        "auto_location": False,
        "tip": "solid-equilateral-triangle",    # Arrow tip on destination
    },
    "line_color": "#3366ffff",
    "line_width": 3.0,
    "type": "curve",
})
```

### Widget Parenting

Any widget can be made a child of another widget. When you set `parent_id`, the client automatically:

1. Checks that the relationship would not create a circular hierarchy (A → B → A).
2. Adjusts the widget's `location` so its visual position on the canvas stays the same.

```python
# Make note_b a child of note_a
updated = await client.update_note(canvas_id, note_b.id, {
    "parent_id": note_a.id,
    # Do NOT supply location here — the client calculates the correct
    # relative offset automatically using the formula:
    #   child_location = current_location - parent_location - 30
})
```

To detach a widget from its parent, set `parent_id` to `null`:

```python
await client.update_note(canvas_id, note_b.id, {"parent_id": None})
```

> **Warning:** Never set a widget as its own parent, or make a widget an ancestor of one of its own descendants. The client raises `CanvusAPIError` before even sending the request if it detects a circular reference.

---

## Uploading Files

Images, videos, and PDFs are created by uploading a file from your local filesystem. The client sends a multipart form request containing the file.

### Uploading an Image

```python
image = await client.create_image(
    canvas_id,
    "/path/to/photo.png",          # Local file path
    {
        "location": {"x": 200, "y": 200},
        "size": {"width": 600, "height": 400},
        "title": "Project Photo",
    },
)
print(f"Image uploaded with hash: {image.hash}")
```

### Uploading a Video

```python
video = await client.create_video(
    canvas_id,
    "/path/to/recording.mp4",
    {
        "location": {"x": 200, "y": 700},
        "size": {"width": 640, "height": 360},
    },
)
```

### Uploading a PDF

```python
pdf = await client.create_pdf(
    canvas_id,
    "/path/to/document.pdf",
    {
        "location": {"x": 900, "y": 200},
        "size": {"width": 500, "height": 700},
        "title": "Specification Document",
    },
)
```

### Navigating PDF Pages

The `index` field on a PDF widget controls which page is displayed (zero-based):

```python
await client.update_pdf(canvas_id, pdf.id, {"index": 2})  # Show page 3
```

### Downloading File Content

You can download the binary content of any file-based widget:

```python
image_bytes = await client.download_image(canvas_id, image.id)
with open("local_copy.png", "wb") as f:
    f.write(image_bytes)

video_bytes = await client.download_video(canvas_id, video.id)
pdf_bytes   = await client.download_pdf(canvas_id, pdf.id)
```

### Generic File Upload to the Uploads Folder

The uploads folder is a staging area that accepts any supported file type. The server auto-detects the file type:

```python
result = await client.upload_file(
    canvas_id,
    "/path/to/any_supported_file.png",
    {"title": "My Upload"},  # Optional metadata
)
```

---

## Real-Time Streaming

Canvus supports server-sent NDJSON streams. Use `subscribe()` to receive live updates whenever widgets change on a canvas.

### Subscribing to Widget Updates

```python
import asyncio
from canvus_api import CanvusClient
from canvus_api.models import Widget

async def watch_canvas(canvas_id: str):
    async with CanvusClient(base_url=URL, api_key=TOKEN) as client:
        # The subscribe() method is an async generator; iterate over it
        async for update in client.subscribe(
            f"canvases/{canvas_id}/widgets",
            response_model=Widget,          # Optional: parse into typed objects
        ):
            print(f"Widget updated: {update.id} ({update.widget_type})")

asyncio.run(watch_canvas("your-canvas-id"))
```

The stream stays open indefinitely. The underlying connection uses no total timeout so it will not time out during normal idle periods; only network failures or explicit cancellation will end the loop.

### Using a Callback Instead of a Loop

```python
def on_widget_update(widget: Widget) -> None:
    print(f"Change detected on {widget.id}")

async for _ in client.subscribe(
    f"canvases/{canvas_id}/widgets",
    response_model=Widget,
    callback=on_widget_update,
):
    pass  # Callback handles the work; the loop just keeps the generator alive
```

### Subscribing to Other Resources

The `endpoint` argument accepts any valid API path. Common subscription targets:

```python
# All canvases
async for update in client.subscribe("canvases"):
    ...

# Widget annotations for a canvas
async for update in client.subscribe_annotations(canvas_id):
    print(update)
```

### Cancelling a Subscription

Wrap the subscription in a task and cancel it when done:

```python
import asyncio

async def main():
    async with CanvusClient(base_url=URL, api_key=TOKEN) as client:
        task = asyncio.create_task(watch_canvas(client, canvas_id))

        # Do other work...
        await asyncio.sleep(60)

        # Stop watching
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass
```

---

## Error Handling

The library raises structured exceptions so you can handle specific failure modes.

### Exception Hierarchy

```
CanvusAPIError          # Base class for all library errors
├── AuthenticationError # HTTP 401 — invalid or missing API key
├── ResourceNotFoundError # HTTP 404 — canvas or widget does not exist
├── RateLimitError      # HTTP 429 — too many requests
├── ServerError         # HTTP 5xx — the server encountered an error
├── TimeoutError        # Connection timeout or HTTP 408
├── ConnectionError     # Network-level connection failure
└── ValidationError     # Malformed request or unexpected response shape
```

### Basic Error Handling

```python
from canvus_api.exceptions import (
    AuthenticationError,
    ResourceNotFoundError,
    RateLimitError,
    CanvusAPIError,
)

async def safe_get_canvas(client, canvas_id):
    try:
        return await client.get_canvas(canvas_id)

    except AuthenticationError:
        print("API key is invalid or has been revoked.")
        raise

    except ResourceNotFoundError:
        print(f"Canvas {canvas_id} does not exist.")
        return None

    except RateLimitError as e:
        print(f"Rate limit hit. Retry after a moment. Status: {e.status_code}")
        raise

    except CanvusAPIError as e:
        # Catch-all for any other API error
        print(f"API error {e.status_code}: {e}")
        raise
```

### Retry Behavior

The client automatically retries requests that fail with transient errors (connection failures, HTTP 5xx, HTTP 408, HTTP 429). The number of retries and the backoff timing are set in the constructor:

```python
async with CanvusClient(
    base_url=URL,
    api_key=TOKEN,
    max_retries=5,       # Try up to 5 times after the initial attempt
    retry_delay=0.5,     # Start with 0.5s delay
    retry_backoff=2.0,   # Double after each retry: 0.5s, 1s, 2s, 4s, 8s
) as client:
    ...
```

Non-retryable errors (HTTP 401, 404, etc.) are raised immediately on the first failure.

### Inspecting Error Details

Every `CanvusAPIError` instance carries the HTTP status code and raw response text:

```python
try:
    await client.delete_canvas("bad-id")
except CanvusAPIError as e:
    print(e.status_code)    # e.g. 404
    print(e.response_text)  # Raw text from the server
```

---

## Next Steps

Now that you can connect, create widgets, upload files, and stream updates, explore the deeper capabilities of the library:

- **[API Reference](api-reference.md)** — Complete method signatures for every operation, organized by resource category.
- **[Architecture Guide](architecture.md)** — How the modules fit together, the parenting system, spatial operations, filtering, cross-canvas search, and export/import.

### Common Patterns to Explore

**Filtering widgets client-side:**

```python
from canvus_api.filters import Filter

# Find all notes whose text contains "TODO"
f = Filter().add_condition("text", "contains", "TODO")
notes = await client.list_widgets(canvas_id, filter_obj=f)
```

**Searching across all canvases:**

```python
from canvus_api.search import CrossCanvasSearch

searcher = CrossCanvasSearch(client)
results = await searcher.find_widgets_by_text("project kickoff")
for r in results:
    print(f"{r.canvas_name} / {r.widget_id}: {r.match_reason}")
```

**Exporting a canvas for backup:**

```python
from canvus_api.export import export_widgets_to_folder

path = await export_widgets_to_folder(client, canvas_id, folder_path="/tmp/backup")
print(f"Exported to {path}")
```
