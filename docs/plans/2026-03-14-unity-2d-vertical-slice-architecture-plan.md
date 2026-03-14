# Unity 2D Vertical Slice Architecture Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a beginner-safe but professionally structured Unity 2D architecture for a vertical slice with a hub city, one mission, progression-aware dialogue, and save/load.

**Architecture:** Use a Unity-aware hybrid architecture with separate Domain, Content, Application, Infrastructure, Gameplay, Presentation, and Bootstrap layers. Keep authored content in `ScriptableObject` assets, keep game rules in domain/application code, add DTO-based persistence, and use project-level plus scene-level lifetimes.

**Tech Stack:** Unity 2D, Unity Input System, ScriptableObjects, local JSON persistence, Timeline for short cutscenes

---

## Summary

This plan defines the recommended hybrid architecture for the project. It is intentionally cleaner than a typical prototype, but avoids overengineering that would fight Unity's scene/prefab workflow.

The architecture keeps:
- Pure runtime state and rules in a `Domain` layer
- Passive authored content in a `Content` layer
- Use-case orchestration in an `Application` layer
- Unity/platform integrations in an `Infrastructure` layer
- Scene-local behavior in `Gameplay`
- UI flow in `Presentation`
- Lifetime wiring in `Bootstrap`

It also includes two Unity-specific refinements:
- DTO-based persistence instead of serializing domain objects directly
- Two lifetimes: `ProjectContext` for cross-scene services and `SceneContext` for per-scene wiring

## Revised Hybrid Architecture

### Layers

1. `Domain`
- Pure runtime state and rules
- No Unity API
- Examples:
  - `ProgressionState`
  - `InventoryState`
  - `ProgressFlag`
  - `UnlockConditionEvaluator`
  - `MissionAvailabilityPolicy`
  - `DialogueSelectionPolicy`
  - `PurchasePolicy`

2. `Content`
- Passive authored game definitions in `ScriptableObject`s
- No business logic beyond trivial helpers
- Examples:
  - `ChapterDefinition`
  - `MissionDefinition`
  - `DialogueDefinition`
  - `ShopInventoryDefinition`
  - `ItemDefinition`

3. `Application`
- Orchestrates use cases
- Reads `Content`, applies `Domain` rules, updates state
- Examples:
  - `ProgressionService`
  - `MissionService`
  - `DialogueService`
  - `ShopService`
  - `CutsceneService`

4. `Infrastructure`
- Unity/platform adapters and persistence
- Examples:
  - `JsonSaveRepository`
  - `ProgressionStateMapper`
  - `UnitySceneLoader`
  - `UnityInputReader`
  - `TimelineCutscenePlayer`

5. `Gameplay`
- Scene/prefab behavior
- Examples:
  - `PlayerController`
  - `PlayerCombat`
  - `EnemyBrain`
  - `MissionCompletionTrigger`
  - `LeverSwitch`
  - `PuzzleGoal`
  - `NpcDialogueInteractor`

6. `Presentation`
- UI and screen controllers
- Examples:
  - `DialoguePanelController`
  - `ShopPanelController`
  - `MissionBoardPanelController`
  - `HudController`

7. `Bootstrap`
- Lifetime wiring
- Split into:
  - `ProjectContext`
  - `SceneContext`

### Folder Structure

```text
Assets/
  Scenes/
    Boot.unity
    MainMenu.unity
    HubCity.unity
    Mission01_PollutedBank.unity

  Scripts/
    Domain/
      Progression/
        ProgressionState.cs
        InventoryState.cs
        ProgressFlag.cs
      Missions/
        MissionAvailabilityPolicy.cs
      Dialogue/
        DialogueSelectionPolicy.cs
      Shops/
        PurchasePolicy.cs
      Shared/
        UnlockConditionEvaluator.cs

    Content/
      Chapters/
        ChapterDefinition.cs
      Missions/
        MissionDefinition.cs
      Dialogue/
        DialogueDefinition.cs
      Shops/
        ShopInventoryDefinition.cs
      Items/
        ItemDefinition.cs

    Application/
      Progression/
        IProgressionService.cs
        ProgressionService.cs
      Missions/
        IMissionService.cs
        MissionService.cs
      Dialogue/
        IDialogueService.cs
        DialogueService.cs
      Shops/
        IShopService.cs
        ShopService.cs
      Cutscenes/
        ICutsceneService.cs
        CutsceneService.cs

    Infrastructure/
      Persistence/
        ISaveRepository.cs
        JsonSaveRepository.cs
        ProgressionSaveData.cs
        InventorySaveData.cs
        ProgressionStateMapper.cs
      Scenes/
        ISceneLoader.cs
        UnitySceneLoader.cs
      Input/
        IInputReader.cs
        UnityInputReader.cs
      Cutscenes/
        ICutscenePlayer.cs
        TimelineCutscenePlayer.cs

    Gameplay/
      Player/
        PlayerController.cs
        PlayerCombat.cs
        PlayerInteractor.cs
      AI/
        EnemyBrain.cs
        EnemyStateMachine.cs
      Missions/
        MissionCompletionTrigger.cs
      Interactables/
        LeverSwitch.cs
        GateController.cs
        PuzzleGoal.cs
      NPCs/
        NpcDialogueInteractor.cs
        ShopNpcInteractor.cs
        MissionBoardInteractor.cs

    Presentation/
      UI/
        DialoguePanelController.cs
        ShopPanelController.cs
        MissionBoardPanelController.cs
        HudController.cs

    Bootstrap/
      ProjectContext.cs
      SceneContext.cs
      ServiceRegistry.cs
```

### Lifetime Model

`ProjectContext`
- Persists across scenes
- Owns:
  - `ProgressionService`
  - `ISaveRepository`
  - `ISceneLoader`
  - `IInputReader`
  - `ICutsceneService`

`SceneContext`
- Recreated per scene
- Wires:
  - player references
  - UI controllers
  - interactors
  - mission-local objects
  - enemy scene references

### Dependencies

```text
Presentation -> Application
Gameplay -> Application
Gameplay -> Content (config only)

Application -> Domain
Application -> Content
Application -> Infrastructure

Infrastructure -> Unity APIs
Content -> Unity serialization only
Domain -> no Unity dependency

Bootstrap -> Application
Bootstrap -> Infrastructure
Bootstrap -> Gameplay/Presentation wiring
```

### Main Class Dependencies

```text
MissionBoardInteractor -> IMissionService
NpcDialogueInteractor -> IDialogueService
ShopNpcInteractor -> IShopService
MissionCompletionTrigger -> IMissionService

MissionService -> IProgressionService
MissionService -> MissionDefinition
MissionService -> MissionAvailabilityPolicy
MissionService -> ISceneLoader
MissionService -> ICutsceneService

DialogueService -> IProgressionService
DialogueService -> DialogueDefinition
DialogueService -> DialogueSelectionPolicy

ShopService -> IProgressionService
ShopService -> ShopInventoryDefinition
ShopService -> PurchasePolicy

ProgressionService -> ProgressionState
ProgressionService -> ChapterDefinition
ProgressionService -> ISaveRepository

JsonSaveRepository -> ProgressionSaveData
ProgressionStateMapper -> ProgressionState
ProgressionStateMapper -> ProgressionSaveData
```

### Event Usage

Use events narrowly, not as a global bus.

Good event cases:
- progression changed
- inventory changed
- cutscene finished

Do not make commands event-driven by default.

Use direct calls for request/response flows:
- `MissionBoardInteractor -> MissionService.StartMission()`
- `ShopPanelController -> ShopService.PurchaseItem()`

### Explicit Non-Recommendations

Do not do these in v1:
- business logic inside `ScriptableObject` assets
- global message bus for everything
- UniRx as a foundation
- DI framework like `VContainer` before the project proves it needs it

## First 12 Classes To Implement

### Domain

1. `ProgressionState`
- Holds:
  - current chapter id
  - completed mission ids
  - unlocked flags
  - simple inventory

2. `UnlockConditionEvaluator`
- Given a `ProgressionState` and a set of requirements, returns true/false

### Content

3. `MissionDefinition`
- Mission id
- display name
- scene name
- unlock requirements
- completion flag
- reward flags
- return scene name

4. `DialogueDefinition`
- NPC id
- dialogue entries
- each entry has required/excluded flags and lines

### Application

5. `ProgressionService`
- Owns live `ProgressionState`
- Adds flags
- marks mission complete
- exposes read access for other systems

6. `MissionService`
- Checks if a mission is available
- starts mission
- completes mission
- applies mission rewards

7. `DialogueService`
- Selects the correct dialogue entry for an NPC from `DialogueDefinition`

### Infrastructure

8. `UnitySceneLoader`
- Loads scenes through Unity

9. `JsonSaveRepository`
- Saves/loads progression

10. `ProgressionStateMapper`
- Converts between `ProgressionState` and save DTO

### Gameplay

11. `PlayerController`
- Move and jump only at first

12. `MissionCompletionTrigger`
- Calls `MissionService.CompleteMission(...)` when player reaches the mission end

### Add Very Early If Needed

These are the next likely classes, but not day-one mandatory:
- `PlayerInteractor`
- `NpcDialogueInteractor`
- `DialoguePanelController`
- `IInputReader` / `UnityInputReader`
- `ProgressionSaveData`

## Recommended Initial Dependency Graph

```text
MissionCompletionTrigger -> MissionService
MissionService -> ProgressionService
MissionService -> MissionDefinition
MissionService -> UnitySceneLoader

DialogueService -> ProgressionService
DialogueService -> DialogueDefinition
DialogueService -> UnlockConditionEvaluator

ProgressionService -> ProgressionState
ProgressionService -> JsonSaveRepository
JsonSaveRepository -> ProgressionStateMapper
```

## First Playable Goal

With just this slice, the game should be able to:
- move in hub
- start a mission
- load mission scene
- reach mission end trigger
- complete mission
- return to hub
- have dialogue change based on progression
- save and load that progress

That is the first proof that the architecture works before adding shops, cutscenes, enemies, or chapter-expansion systems.

## Assumptions

- Placeholder art and UI are acceptable for the first implementation pass
- Controller support is deferred, but the Input System should still be used from the start
- The optional 3D opening cutscene remains out of scope until the 2D architecture is stable
- The architecture should stay Unity-native rather than trying to emulate a pure enterprise backend design
