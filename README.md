# PFound.LoopScheduler

Deterministic multi-phase frame scheduler injected into Unity's PlayerLoop. Replaces scattered
`Update()` methods with ordered `LoopPhase` phases plus owner-scoped, re-entrancy-safe callbacks and
a per-phase Profiler marker. The scheduler is a **static, self-installing** injector — there is
nothing to place in a scene.

## Model

- **Phases** (`LoopPhase`, declaration order = execution order): `Input`, `Selection`, `EarlyUpdate`,
  `Update`, `LateUpdate`, `PostUpdate`, `Camera`, `Bindings`, `PreRender`, `BeforeRender`, `Render`,
  `PostRender`, `Scene`, `Network`, `EndOfFrame`. Callbacks within a phase run in registration order.
- **One early pass + a pre-render hook.** Every logic phase runs in a single pass injected at the
  front of the PlayerLoop each frame; `BeforeRender` alone is driven by the engine's
  `Application.onBeforeRender`.
- **Re-entrancy safe.** Registrations/removals requested while a phase is invoking are deferred to
  after the pass; a callback added mid-invoke waits for the next frame.
- **Owner-scoped auto-cleanup.** A persistent callback tagged with a `UnityEngine.Object` owner is
  pruned automatically once that owner is destroyed (fake-null). A `null` owner means
  caller-managed lifetime and is never pruned.

## Public API (`LoopScheduler`, static)

- `RegisterUpdateLoop(Action callback, UnityEngine.Object owner)` / `DeregisterUpdateLoop(callback)`
  — persistent callback on the `Update` phase.
- `RegisterBeforeRenderLoop(Action callback, UnityEngine.Object owner)` /
  `DeregisterBeforeRenderLoop(callback)` — persistent callback on the pre-render hook.
- `InvokeOnceAt(LoopPhase phase, Action callback)` — one-shot at the next occurrence of `phase`.
- `LoopSchedulerTools.IsCustomLoop(this PlayerLoopSystem)` — extension to spot the injected pipeline
  when inspecting the PlayerLoop.

## Setup / wiring

**No setup.** The scheduler installs itself: a `[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]`
entry point injects the phase pipeline into the PlayerLoop and subscribes the pre-render hook before
the first scene loads. You never instantiate it, place it in a scene, or add it to the PlayerLoop by
hand — just call the static register methods.

```csharp
// from anywhere, once your object exists:
LoopScheduler.RegisterUpdateLoop(Tick, this);      // 'this' MonoBehaviour owns the callback
LoopScheduler.InvokeOnceAt(LoopPhase.EndOfFrame, FlushOnce);
// no manual deregister needed when 'this' is destroyed — it is auto-pruned.
```

Pass a real `owner` for anything with a bounded lifetime so it is dropped when the owner dies; pass
`null` only when you will `Deregister…` it yourself. Main-thread only.

## Layout

`Runtime/` — `LoopScheduler` (static injector), `LoopPhase`, `PhaseCallbacks` (engine-free,
re-entrancy-safe callback list), `LoopSchedulerTools`. Assembly `PFound.LoopScheduler`
(`autoReferenced:false`; references the engine for `PlayerLoop`/`ProfilerMarker`).
