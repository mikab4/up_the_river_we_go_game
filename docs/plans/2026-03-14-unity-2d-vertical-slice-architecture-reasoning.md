# Unity 2D Vertical Slice Architecture Reasoning

## Purpose

This document explains why the architecture in [2026-03-14-unity-2d-vertical-slice-architecture-plan.md](/home/mikab4/projects/down_the_river_we_go/docs/plans/2026-03-14-unity-2d-vertical-slice-architecture-plan.md) was chosen, what tradeoffs it makes, and which alternative approaches were intentionally rejected.

The target is a solo-developed Unity 2D game with:
- a hub city
- one initial story mission
- progression-aware dialogue
- eventual chapter expansion
- a codebase that remains teachable while still being professionally structured

## Constraints That Drove The Design

The architecture was shaped by these constraints:

- The project is being built in Unity, so scenes, prefabs, `MonoBehaviour`s, and `ScriptableObject`s are first-class parts of the system.
- The game is small at first, so a large-team architecture would create too much ceremony.
- The code still needs strong boundaries because the project is meant to grow by adding more missions and chapters later.
- The developer already understands software architecture, so the structure should be disciplined rather than ad hoc.
- The developer is new to Unity, so the architecture cannot depend on too many Unity-specific frameworks or advanced editor tooling.

These constraints rule out both extremes:
- a loose prototype architecture with gameplay logic spread across scene scripts and managers
- a very pure enterprise-style layered architecture that fights Unity's normal workflow

## Why A Hybrid Architecture Was Chosen

The chosen architecture is a Unity-aware hybrid:

- `Domain` holds runtime state and rules
- `Content` holds authored game data
- `Application` orchestrates use cases
- `Infrastructure` wraps Unity/platform details
- `Gameplay` holds scene-local behavior
- `Presentation` holds UI logic
- `Bootstrap` handles lifetime wiring

This structure was selected because it preserves separation of concerns without pretending Unity is a generic application framework.

The most important design principle is:

> Keep gameplay local, keep progression centralized, keep content passive.

That principle supports future chapter expansion while keeping the first vertical slice understandable.

## Why Domain And Content Stay Separate

One alternative was to merge `Content` and `Domain` by putting more logic directly into `ScriptableObject` assets.

That was rejected.

Reason:
- It hides behavior inside assets
- It weakens testability
- It makes refactoring harder
- It encourages the project to spread core rules across inspector-configured objects instead of keeping them in code

The chosen model is:
- `ScriptableObject`s describe content
- services and domain-policy classes interpret that content

Example:
- `MissionDefinition` contains unlock requirements
- `MissionAvailabilityPolicy` decides whether those requirements pass against `ProgressionState`

This gives designers or future content work enough flexibility without moving the game's rules into assets.

## Why Application Services Exist

The `Application` layer exists to keep cross-scene orchestration out of scene objects.

For example:
- `MissionService` decides whether a mission can start, loads the mission scene, and applies mission rewards on completion
- `DialogueService` chooses the correct lines for an NPC based on progression
- `ShopService` decides what can be bought and what inventory is visible

Without these services, that logic would leak into:
- hub NPC scripts
- UI panels
- mission scenes
- scene transition code

That kind of leakage makes later chapter expansion harder, because global flow gets mixed into local scene behavior.

## Why Infrastructure Is Wrapped

Infrastructure wrappers are used for boundaries where Unity APIs or persistence details would otherwise leak upward.

Recommended wrappers:
- `ISceneLoader`
- `ISaveRepository`
- `IInputReader`
- `ICutscenePlayer`

This was chosen for three reasons:
- it protects higher-level systems from Unity-specific details
- it keeps responsibilities clearer
- it makes the architecture easier to test and refactor

The goal is not to wrap every Unity class. The goal is to wrap the boundaries that matter.

That is why there are no interfaces for scene-local classes like:
- `PlayerController`
- `EnemyBrain`
- `LeverSwitch`

Those are natural concrete Unity components.

## Why Persistence Uses DTOs

The save/load path uses DTOs instead of serializing `ProgressionState` directly.

Chosen structure:
- `ProgressionState` is the runtime model
- `ProgressionSaveData` is the persistence DTO
- `ProgressionStateMapper` converts between them

This was selected because Unity-style JSON serialization has practical limitations, and save formats often evolve differently from runtime state.

Benefits:
- save format is explicit
- runtime state can change without tightly coupling to disk format
- persistence stays in `Infrastructure` where it belongs

This is one of the most important structural choices in the architecture.

## Why The Bootstrap Is Split By Lifetime

Unity naturally has two lifetimes:
- application lifetime
- scene lifetime

So the architecture uses:
- `ProjectContext` for persistent services across scenes
- `SceneContext` for per-scene wiring

This was chosen because a single global bootstrap does not model Unity's object lifetime well.

`ProjectContext` should own long-lived systems like:
- progression
- save/load
- scene loading
- input access
- cutscene coordination

`SceneContext` should wire scene-local objects like:
- player references
- mission-local triggers
- UI panels
- NPC interactors

This keeps scene loading and destruction predictable without introducing a heavy dependency injection framework too early.

## Why Events Are Narrowly Used

The architecture intentionally avoids a global event-bus-first design.

Rejected approach:
- everything communicates through a message broker or reactive stream

Reason:
- too much indirection for a small project
- harder to debug while learning Unity
- command flows become harder to trace

Chosen approach:
- direct service calls for commands
- small local events for notifications

Good uses of events:
- progression changed
- inventory changed
- cutscene ended

Good uses of direct calls:
- start mission
- complete mission
- purchase item
- request dialogue

This keeps control flow legible.

## Why The Architecture Is Not Simpler

A much simpler architecture was considered:
- a few managers
- scene scripts talking directly to them
- less layering

That was rejected as the final target because the game is expected to expand with:
- more missions
- more chapters
- more dialogue states
- more shop states

With a simpler structure, global progression logic tends to leak into:
- hub NPC scripts
- mission triggers
- shop UIs
- cutscene scripts

That would make later chapter additions increasingly fragile.

## Why The Architecture Is Not More Complex

A larger-team style architecture was also considered:
- use-case classes for every interaction
- facades
- presenters and view models everywhere
- mappers between content assets and domain models
- broader eventing

That was rejected for v1 because:
- it adds too much boilerplate
- it slows down learning Unity
- it increases the amount of infrastructure before the game loop is proven
- it optimizes for team scaling that the project does not yet need

The current architecture is meant to be the smallest version that still has professional boundaries.

## What “Professional” Means In This Context

Professional does not mean maximal abstraction.

Here it means:
- stable ownership boundaries
- clear dependency direction
- runtime state is centralized
- content is structured for expansion
- infrastructure concerns are isolated
- Unity-native workflows remain intact

That is why this architecture is intentionally a hybrid rather than a pure clean architecture implementation.

## Core Architectural Rules

The most important rules are:

- global progression changes go only through `ProgressionService`
- mission start and mission completion go only through `MissionService`
- dialogue selection goes only through `DialogueService`
- shop inventory and purchasing go only through `ShopService`
- scene objects report local events; they do not define campaign progression
- content assets describe content; they do not execute business logic

If these rules are preserved, adding new chapters should mostly be a content-authoring task rather than a systems rewrite.

## Expansion Goal

The architecture is considered successful if future chapters can be added mostly by creating:
- new `ChapterDefinition` assets
- new `MissionDefinition` assets
- new scenes
- new dialogue entries
- new shop entries
- optional cutscene assets

The following systems should remain largely unchanged during that work:
- `ProgressionService`
- `MissionService`
- `DialogueService`
- `ShopService`
- `Save` infrastructure

That is the real expansion target, and it is the main reason this architecture was chosen.
