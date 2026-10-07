# CupperStory

A 2D adventure puzzle game for a Unity course assignment.

**Status: not started.** This repository is a fresh Unity template. There are no scripts, no art assets, and no gameplay yet. What exists here is the agreed architecture, so four people can work in parallel without colliding.

## Requirements

The assignment brief asks for:

- A game menu where the player can read the instructions
- One or more playable levels
- A GameOver UI and a Win UI
- Enigmas for the player to solve
- Assets made by us, not purchased
- Builds on Desktop, Mobile and Web

Team rule: **n people requires at least n levels**, so four people requires four levels.

## Setup

Requirements are exact. A different Unity version will produce asset churn across the whole team.

- **Unity 6000.6.0f1** (verify in `ProjectSettings/ProjectVersion.txt`)
- The **Input System** package is the only active input backend. See "Input" below, this will bite you.
- **Everyone clones separately.** Do not share a folder over a network drive or a USB stick. Unity writes `Library/` constantly and a shared copy corrupts the asset database, which produces "multiple Unity instances" lock prompts and costs whole afternoons. `Library/` is gitignored for exactly this reason.

```
git clone <repo-url>
```

Open with Unity Hub, let the first import finish (several minutes, this is normal), then open `Assets/Scenes/SampleScene.unity`.

## Running tests

EditMode tests, from the command line:

```bash
~/Unity/Hub/Editor/6000.6.0f1/Editor/Unity -batchmode -nographics -quit \
  -projectPath "$PWD" \
  -runTests -testPlatform EditMode -testResults /tmp/results.xml
```

Single test, add `-testFilter "LevelRegistryTests"`. Results land in `/tmp/results.xml`. In the editor, use Window > General > Test Runner.

To force a compile check without the GUI, add `-quit` alone. This is the fastest way to catch C# errors before pushing.

## Architecture

One scene per level, and a single persistent `GameManager` that survives scene loads via `DontDestroyOnLoad`. The first instance to `Awake` survives; duplicates destroy themselves, so every scene can safely contain one.

Scenes:

| Scene | Purpose | Owner |
|---|---|---|
| `MainMenu` | Entry point, build index 0. Menu plus instructions. | Person 1 |
| `Levels/Level01` | First enigma | Person 1 |
| `Levels/Level02` | Second enigma | Person 2 |
| `Levels/Level03` | Third enigma | Person 3 |
| `Levels/Level04` | Fourth enigma | Person 4 |
| `Win` | Win screen, doubles as the final victory screen | Person 1 |
| `GameOver` | GameOver screen, shows the failure reason | Person 1 |

### The integration contract

This is the load-bearing idea of the whole design. **A level author writes exactly two calls and nothing else:**

```csharp
GameManager.Instance.ReportLevelWon();
GameManager.Instance.ReportLevelLost("The cup tipped over.");
```

No interface to implement. No base class. No scene references to wire up. Your level script decides when the enigma is solved and tells the flow; the flow decides what happens next.

That constraint is what makes four-person parallelism work. Nothing you write requires anyone else's file to change.

### Design decisions, and why

**No UnityEvent wiring.** The textbook Unity way to connect a level trigger to a scene object is Inspector-wiring a UnityEvent. Do not do this. Wiring a scene object into a prefab is stored as a prefab-instance override, and that override lives in the scene file. All four of us would then be editing one file, which is exactly the merge conflict this architecture exists to prevent. A direct static call replaces it.

**No assembly definitions (yet).** Skipped on purpose. In a single `Assembly-CSharp`, a syntax error only stalls whoever wrote it. With per-person asmdefs, forgetting `"references": ["Unity.InputSystem"]` or toggling Enable in the inspector breaks all four compiles on day one. Revisit this only if merge pain actually shows up, not before.

**No `Time.timeScale = 0`.** It breaks `Button` clicks on some Unity versions, and the win and lose screens are buttons. Freeze input with `PlayerMovement.SetInputEnabled(false)` instead.

**Multi-scene rather than one scene with level prefabs.** Both were designed and compared. Prefabs reduce the scene file to a single one, which helps merging, but cost five assembly definitions, a `Resources.Load` string convention, and a break from the convention the course teaches. Multi-scene lets each of us open our own level and hit Play without the others' work being finished, which is what "everyone presents their part" requires.

**Frame rate is set once.** `Application.targetFrameRate = 60` and `QualitySettings.vSyncCount = 0` are applied in `GameManager.Awake`. The brief's "responsive" means a stable framerate, not a responsive layout.

### File ownership

Non-negotiable. Two people editing one file is the failure mode the architecture is designed to prevent.

| Owner | Files |
|---|---|
| Person 1 | `Assets/Scripts/Core/`, `Assets/Scripts/UI/`, `Assets/Tests/`, the `MainMenu`, `Win`, and `GameOver` scenes, and `Level01` |
| Person 2 | `Assets/Scripts/Player/`, `Assets/Prefabs/Player.prefab`, `Level02` |
| Person 3 | `Level03` |
| Person 4 | `Level04`, and the Desktop/Mobile/Web builds |

Each level owner keeps everything under their own `LevelRoot` GameObject and in their own script namespace, for example `CupperStory.Level01`.

## Things that will cost you debugging time

- **The legacy `Input` class does not work.** `activeInputHandler` is set to `1`, meaning the Input System package only. `Input.GetAxis` and `Input.GetKey` are disabled. Use `UnityEngine.InputSystem`.
- **EventSystem needs `InputSystemUIInputModule`,** not `StandaloneInputModule`. With the Input System active, the legacy module does nothing.
- **The Win screen must read the level index in `Start`, not `Awake`.** `SceneManager.sceneLoaded` fires *after* `Awake`, so the index is not settled yet. Getting this wrong means the final level shows "Level complete" instead of "Victory".
- **Guard against double reporting.** Two puzzle objects resolving in the same frame would both call `ReportLevelWon()`. Use a frame lock.
- **Guard `QuitGame` with `#if !UNITY_WEBGL`.** `Application.Quit` is a no-op in a browser and logs an error on every click. WebGL is a required target.
- **Never modify an instance of `Player.prefab` inside a level scene.** It silently stores an override that resurrects stale values when Person 2 later fixes the prefab, and the bug comes back with no visible cause. Place your own objects beside it instead.
- **Gradle and Android signing are not configured yet.** The Android build engine is installed, but signing needs a keystore, which must stay out of git.

## Build targets

All four engines are already installed: Windows, Linux, WebGL, Android. Nothing needs installing.

Nothing has been built yet. When that changes, `Assets/Scenes/MainMenu.unity` must be build index 0.