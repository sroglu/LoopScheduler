# LoopScheduler

## Purpose
A deterministic multi-phase frame scheduler injected into Unity's PlayerLoop. Replaces scattered
`Update()` methods with ordered `LoopPhase` phases plus owner-scoped, re-entrancy-safe callbacks and
a per-phase Profiler marker.

## Assemblies

| Assembly | Path | Notes |
|---|---|---|
| `PFound.LoopScheduler` | `Runtime/PFound.LoopScheduler.asmdef` | `autoReferenced: false`; references the engine for `PlayerLoop` / `ProfilerMarker` / `Application.onBeforeRender` |

## Dependencies
- Engine only: `UnityEngine.LowLevel.PlayerLoop`, `Unity.Profiling.ProfilerMarker`,
  `Application.onBeforeRender`, `RuntimeInitializeOnLoadMethod`.
- No other PFound module, no third-party package, no scripting define.

## Key Types

### `PFound.LoopScheduler`
- **`LoopScheduler`** (static) — the self-installing injector and the register/deregister surface.
- **`LoopPhase`** (enum) — the ordered frame phases; declaration order = execution order.
- **`PhaseCallbacks`** (internal) — engine-free, re-entrancy-safe ordered callback list for one
  phase (deferred add/remove, one-shot entries, owner pruning). Pure C#, testable without Unity.
- **`LoopSchedulerTools`** (static) — `IsCustomLoop(this PlayerLoopSystem)` extension to spot the
  injected pipeline when inspecting the PlayerLoop.

## Public API

`LoopScheduler` (static):
```csharp
void RegisterUpdateLoop(Action callback, UnityEngine.Object owner);   // persistent, on Update phase
void DeregisterUpdateLoop(Action callback);
void RegisterBeforeRenderLoop(Action callback, UnityEngine.Object owner);  // persistent, pre-render hook
void DeregisterBeforeRenderLoop(Action callback);
void InvokeOnceAt(LoopPhase phase, Action callback);                  // one-shot at next occurrence of phase
```

`LoopSchedulerTools`:
```csharp
bool IsCustomLoop(this PlayerLoopSystem system);   // true if this is the injected pipeline
```

## Model

- **Phases** (`LoopPhase`, declaration order = execution order): `Input`, `Selection`, `EarlyUpdate`,
  `Update`, `LateUpdate`, `PostUpdate`, `Camera`, `Bindings`, `PreRender`, `BeforeRender`, `Render`,
  `PostRender`, `Scene`, `Network`, `EndOfFrame`. Callbacks within a phase run in registration order.
- **One early pass + a pre-render hook.** Every logic phase runs in a single pass injected at the
  FRONT of the PlayerLoop each frame (before the engine's own subsystems). `BeforeRender` alone is
  driven by the engine's `Application.onBeforeRender`.
- **Re-entrancy safe.** Registrations/removals requested while a phase is invoking are deferred to
  after the pass; a callback added mid-invoke waits for the next frame.
- **Owner-scoped auto-cleanup.** A persistent callback tagged with a `UnityEngine.Object` owner is
  pruned automatically once that owner is destroyed (fake-null), before the next invoke of its phase.
  A `null` owner means caller-managed lifetime and is never pruned — you `Deregister…` it yourself.
- **Per-phase profiling.** Each phase invoke is wrapped in a `ProfilerMarker` named
  `LoopScheduler.<Phase>`.

## Setup / wiring

**No placement, no instantiation — just call `Register*`.** The scheduler installs itself: a
`[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]` entry point injects the phase pipeline into the
PlayerLoop and subscribes the pre-render hook before the first scene loads. You never `new` it, place
it in a scene, or add it to the PlayerLoop by hand.

```csharp
// from anywhere, once your object exists:
LoopScheduler.RegisterUpdateLoop(Tick, this);      // 'this' MonoBehaviour owns the callback
LoopScheduler.InvokeOnceAt(LoopPhase.EndOfFrame, FlushOnce);
// no manual deregister needed when 'this' is destroyed — it is auto-pruned.
```

Pass a real `owner` for anything with a bounded lifetime so it is dropped when the owner dies; pass
`null` only when you will `Deregister…` it yourself. Main-thread only.

## File Structure
```
LoopScheduler/
  README.md
  MODULE.md
  Runtime/
    PFound.LoopScheduler.asmdef
    LoopScheduler.cs        # static self-installing injector + register/deregister surface
    LoopPhase.cs            # ordered frame-phase enum
    PhaseCallbacks.cs       # internal, engine-free re-entrancy-safe callback list
    LoopSchedulerTools.cs   # IsCustomLoop PlayerLoopSystem extension
```

## Downstream Dependents
- **`PFound.Render.BatchRendering`** and **`PFound.Render.RenderContext`** reference the assembly to
  drive their per-frame work off the scheduler's phases rather than their own `Update()`.

## Limitations / Known Gaps
- **Main-thread only.** The callback lists are not thread-safe; register/invoke from the main thread.
- **Register surface is Update + BeforeRender only.** The other `LoopPhase` values are reachable for
  one-shots via `InvokeOnceAt`, but there is no persistent-register helper for each phase — add one
  if a consumer needs a persistent callback on, say, `LateUpdate`.
- **No ordering key within a phase.** Callbacks run in registration order; there is no priority
  parameter. Register in the order you need, or split across phases.
- **Global PlayerLoop mutation.** Injection front-loads a subsystem into the PlayerLoop for the whole
  player; there is no per-scene or opt-out install.
