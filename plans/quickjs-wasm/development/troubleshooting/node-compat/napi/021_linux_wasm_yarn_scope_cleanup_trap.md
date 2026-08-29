# Linux/wasm Yarn scope-cleanup trap

| | | Remarks |
| --- | --- | --- |
| **Status** | 🟢 | Resolved 2026-08-28: N-API user finalizers are deferred out of QuickJS's cycle sweep. |
| **Severity** | High | The trap stopped Yarn mid-fetch in the browser guest. |

## Observed failure

The pinned Actual checkout's Yarn fetch step stops after four progress bars in
both Safari and Chromium. Chromium's browser console shows that the Edge.js
Linux/wasm process trapped rather than waiting for disk I/O:

```text
error running user module: RuntimeError: unreachable
    at edge.malloc_usable_size
    at edge.js__malloc_usable_size
    at edge.js_free_shape0
    at edge.free_gc_object
    at edge.js_free_value_rt
    at edge.JS_FreeValue
    at edge.napi_value__::~napi_value__()
    at edge.napi_allocator__<napi_value__, napi_scope__, 64ul>::block__::close()
    at edge.napi_allocator__<napi_value__, napi_scope__, 64ul>::close()
    at edge.napi_scope__::~napi_scope__()
```

This is a runtime memory/lifetime failure surfaced during N-API handle-scope
cleanup. It is not evidence of a blocked OPFS request. The top frame alone does
not prove that `malloc_usable_size` is defective: musl's implementation will
also trap when QuickJS passes it a stale or corrupted allocation pointer.

## Diagnosis

QuickJS's cycle collector (`gc_free_cycles`) ran N-API user finalizers inline
during `JS_GC_PHASE_REMOVE_CYCLES`. A TLSWrap finalizer resolved a weak
`napi_ref` through `napi_get_reference_value`; the non-empty guard passed and
`dup_inner()` dup'd the condemned object back into a live handle scope. The
resurrection sent `free_object` down its `gc_zero_ref_count_list` branch, so
the scope's later close freed the same object a second time; `p->shape` was
already NULL, the allocation pointer read `0xfffffffc`, and mallocng rejected
it in `get_meta`, surfacing as `unreachable` in `malloc_usable_size`.
Instrumentation tied every detection to Node's TLS socket wrap — the property
set Yarn's HTTPS fetching exercises — with exact pointer correlation between
objects wrapped during the sweep and objects freed under a live handle.

## Fix

`distro/edgejs/napi-finalizer-deferral.patch` (shipped as edgejs apk r9)
removes user code from the sweep:

- QuickJS gains a GC epilogue handler (`JS_SetGCEpilogueHandler`,
  `JS_GetGCPhase`, `JSGCPhaseEnum` in `quickjs.h`); `JS_RunGC` fires it after
  `gc_free_cycles`, and `__JS_FreeRuntime` fires it once more at teardown.
- `napi_external__::finalizer` and `free_external_array_buffer_data` queue
  their hints during the sweep instead of running user code; queued entries
  carry no JSValues, so nothing resurrected can outlive the sweep.
- The epilogue drains the queue (`invoke_finalizer`, then
  `destroy_with_runtime`), looping until empty behind a `draining` flag so a
  nested collection cannot recurse; teardown erases the queue.

The narrow `napi_get_reference_value` mitigation was not needed: the sweep
clears the weak ref before the deferred finalizer runs, so it resolves null
through the ordinary empty path.

## Validation

- A fresh unperturbed `yarn install --immutable --mode=skip-build` of the
  pinned Actual checkout completed in the guest in 916,899 ms with exit 0 and
  `node_modules/.yarn-state.yml` present.
- The instrumented fixed build scored 3/3 runs with `wrap_during_gc=0`,
  `live_handle=0`, `double_free=0`, `over_release=0`, against baseline
  detections of 2, 7, 1, 0, 0, 0 across six pre-fix runs (score by detector
  counts, not trap rate).
- The sandbox-built binary contains `JS_GetGCPhase`,
  `JS_SetGCEpilogueHandler`, and `drain_pending_finalizers`.
- The focused Edge/WAMR checks (`async-io`, `buffer-memory`,
  `linked-addon`, `webassembly`, `webassembly-teardown`, `wamr-interpreter`)
  pass on the fixed tree.

