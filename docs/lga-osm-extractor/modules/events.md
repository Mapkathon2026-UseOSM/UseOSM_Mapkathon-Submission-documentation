# `lga_extractor/events.py`

## Purpose

A new, small module with an outsized effect on the rest of the codebase: it defines the plain-dict event schema the extraction pipeline now emits as it runs, plus two tiny helpers for consuming those events safely from a UI thread. This is what makes [`app.py`](../app.md)'s new live, per-stage progress interface possible, without making the pipeline itself (`boundary.py`, `layers.py`, `pipeline.py`) depend on Streamlit — or any UI framework — at all.

## Why this exists

Before this module, the pipeline was a black box from the caller's perspective: `extract_lga()` ran for several minutes and then returned (or raised) once, with no visibility into what stage it was in or whether progress was being made. The obvious fix — sprinkling `st.progress(...)` calls directly inside `layers.py` — would have tied the core extraction logic to Streamlit specifically, breaking every other use of this package (the CLI, a notebook, a future different UI).

The actual fix: every pipeline function that can take a while now accepts an optional `on_event` callback and calls it with a plain dict at each meaningful transition — a stage starting, retrying, completing, or failing. A UI supplies a callback that does something with those events; a script that doesn't care simply omits the argument, and `on_event` defaults to `None` everywhere, a guaranteed no-op regardless of whether Streamlit (or anything else) is even installed.

## Event shape

Every event is a dict with at least `"type"` and `"stage"` keys:

```python
{"type": "stage_started",   "stage": "boundary"}
{"type": "stage_completed", "stage": "boundary", "detail": "..."}
{"type": "stage_started",   "stage": "layer:roads"}
{"type": "retry",           "stage": "layer:roads", "attempt": 2, "message": "..."}
{"type": "stage_completed", "stage": "layer:roads", "detail": "2,431 features", "status": "success"}
{"type": "warning",         "stage": "layer:schools", "message": "..."}
{"type": "stage_failed",    "stage": "layer:health_facilities", "message": "..."}
{"type": "stage_started",   "stage": "cleaning"}
{"type": "stage_completed", "stage": "cleaning"}
{"type": "stage_started",   "stage": "export"}
{"type": "stage_completed", "stage": "export"}
{"type": "pipeline_completed", "summary": {...}}   # the same dict extract_lga() returns
```

Layer stages are namespaced `"layer:{layer_name}"` (e.g. `"layer:roads"`) specifically so a UI can group them separately from the top-level `"boundary"` / `"cleaning"` / `"export"` stages without needing to guess which names belong to which category.

## Functions

### `build_stage_order(tag_config: dict) -> list`

**What it does:** returns the full, ordered list of stage names for a given `tag_config` — `["boundary", "layer:roads", "layer:buildings", ..., "cleaning", "export"]` — with layer stages inserted between the two static boundaries (`STATIC_STAGES_BEFORE_LAYERS = ["boundary"]`, `STATIC_STAGES_AFTER_LAYERS = ["cleaning", "export"]`).

**Why it exists:** so a UI can pre-render a fixed checklist of every stage *before* extraction starts, rather than only discovering stages as events for them happen to arrive. This matters specifically because `tag_config` is user-extensible (see [`layers.py`](layers.md)'s `DEFAULT_TAG_CONFIG`) — a caller who adds a seventh layer needs the progress checklist to have seven layer rows without any code change to this function.

### `ThreadSafeEventQueue`

**What it does:** a minimal thread-safe sink for events — a thin wrapper around `queue.Queue` exposing exactly the `on_event(event)` call signature `extract_lga()`/`extract_layers()` expect (the class implements `__call__`, so an instance of it *is* a valid `on_event` callback), plus `empty()` and `drain()` for a consumer to pull events back out.

**The pattern this enables**, straight from the class's own docstring:
```python
events = ThreadSafeEventQueue()
thread = threading.Thread(target=extract_lga, kwargs={..., "on_event": events})
thread.start()
while thread.is_alive() or not events.empty():
    for event in events.drain():
        ...update UI state from event, on the main thread...
```

**Why this specific pattern is necessary, not just convenient:** `extract_lga()` is a single, blocking call that can take several minutes — layer extraction alone runs concurrently across worker threads (see [`layers.py`](layers.md)'s `ThreadPoolExecutor`). The only way to update a UI *while* it runs, rather than only before and after, is to run extraction on a background thread and have something else draining events on the main/UI thread — and Streamlit's own APIs are explicitly not safe to call directly from a background thread. `ThreadSafeEventQueue` is the connective tissue: safe to call `on_event()` (i.e. call the instance) from *any* thread (including the worker threads inside `layers.extract_layers()`'s thread pool), but `drain()` should only ever be called from the thread that owns the UI.

**`drain()`'s specific implementation detail:** pops and returns every event currently queued *without blocking* (`queue.Queue.get_nowait()` in a loop until `queue.Empty`), so a UI polling loop can call it repeatedly without ever stalling waiting for a new event — it always returns immediately, with an empty list if nothing new has arrived yet.

### `_emit(on_event, event: dict)` (private)

**What it does:** calls `on_event(event)` if `on_event` is not `None`, swallowing *any* exception the callback itself raises.

**Why swallowing the exception is the correct choice here, not a code smell:** a bug in a UI's event-handling code (a `KeyError` on an unexpected event shape, a Streamlit call from the wrong thread, anything) must never be allowed to abort a live, multi-minute extraction that was otherwise succeeding. Per the function's own docstring: that would be "a strictly worse regression than not having a progress UI at all." This is a deliberate, narrow exception to the general principle of not swallowing errors silently — justified specifically because the event system is a pure side channel with no bearing on extraction correctness; losing a progress update is a UI annoyance, aborting a real extraction because of it would be a functional regression.

## Thread safety — the single most important thing to understand about this module

Stated directly in the module's own docstring, and worth repeating here because it's easy to miss: layer extraction runs multiple layers concurrently in a `ThreadPoolExecutor` (see [`layers.py`](layers.md)), so `on_event` **will** be called from multiple worker threads at once for `"layer:*"` stages, not just from the main thread. A callback that only appends to a `queue.Queue` (thread-safe by design — exactly what `ThreadSafeEventQueue` does) or increments a `threading.Lock`-protected counter is safe to pass directly as `on_event`. A callback that touches Streamlit UI elements directly is **not** safe to pass straight into `extract_lga()` — this is precisely why `app.py` doesn't pass a Streamlit-touching function as `on_event` at all; it passes a `ThreadSafeEventQueue` instance and does the actual UI updates in a polling loop on the main thread instead.

## Gotchas

- **This module has zero dependency on Streamlit or any UI framework** — `import queue` is the only import in the whole file. A future consumer building a different UI (a CLI progress bar, a web API with server-sent events, a Jupyter widget) can reuse this exact event schema and `ThreadSafeEventQueue` without pulling in anything Streamlit-specific.
- **Layer-stage events genuinely interleave** across threads (`MAX_CONCURRENT_LAYER_QUERIES = 2` from `layers.py`, so up to two layers' events can arrive in an unpredictable interleaved order) — a consumer should not assume events for one layer complete before another layer's events begin arriving.
- **`_emit`'s silent exception-swallowing** means a bug in your own `on_event` callback will fail completely silently from the pipeline's perspective — if your progress UI isn't updating and you suspect a bug in your callback itself, don't expect the pipeline to surface it; you'll need to debug the callback in isolation.

## Related

- [`layers.py`](layers.md) — the module with the richest event emission (started/retry/completed/failed per layer, from worker threads).
- [`pipeline.py`](pipeline.md) — emits the top-level stage events (boundary/cleaning/export) and the final `pipeline_completed` event.
- [`app.md`](../app.md) — the sole current consumer, using `ThreadSafeEventQueue` to drive a live, per-stage progress interface.
