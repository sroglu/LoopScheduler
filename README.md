# PFound.LoopScheduler

A deterministic multi-phase frame scheduler injected into Unity's PlayerLoop — ordered `LoopPhase`
phases with owner-scoped, re-entrancy-safe callbacks and per-phase Profiler markers. Static and
self-installing: nothing to place in a scene.

## Quick reference

```csharp
// no setup — the scheduler installs itself before the first scene loads.
LoopScheduler.RegisterUpdateLoop(Tick, this);                 // persistent; auto-pruned when 'this' dies
LoopScheduler.RegisterBeforeRenderLoop(SyncCamera, this);     // persistent, pre-render hook
LoopScheduler.InvokeOnceAt(LoopPhase.EndOfFrame, FlushOnce);  // one-shot
LoopScheduler.DeregisterUpdateLoop(Tick);                     // only needed if owner was null
```

## Dependencies

Engine only (`PlayerLoop` / `ProfilerMarker` / `Application.onBeforeRender`). `autoReferenced:false`.

## Docs

Deep reference: [MODULE.md](MODULE.md) — phase order, re-entrancy/ownership model, full API, and the
"just call `Register*`" wiring note.
