# UpscalerSwitcher Plugin for Unreal Engine

`UpscalerSwitcher` lets a game switch between NVIDIA DLSS and AMD FSR at runtime and remember the
player's choice across sessions, without taking a compile-time dependency on either vendor plugin.

## 🔧 Features

- Switch the upscaler — DLSS, FSR or none — instantly at runtime, in any direction.
- Per-quality-mode control for both upscalers.
- DLSS Frame Generation, including multi frame generation modes on hardware that supports them.
- FSR Frame Generation.
- The player's choice is saved and reapplied automatically on the next launch.
- An FPS reading that stays correct while frame generation is running.
- Every function is Blueprint-callable — a Blueprint-only project needs no C++.
- Compiles and runs whether or not the DLSS and FSR plugins are installed.

## Requirements

|                |                                                              |
| -------------- | ------------------------------------------------------------ |
| Engine         | 5.2 – 5.8                                                    |
| Platform       | Win64, D3D12                                                 |
| Vendor plugins | Both optional. The plugin degrades to whatever is installed. |
| Project type   | Blueprint-only is fine.                                      |

## 📦 Installation

1. Copy the `UpscalerSwitcher` plugin into your project's `Plugins` directory.
2. Enable it in your `.uproject`:

    ```json
    "Plugins": [
        {
            "Name": "UpscalerSwitcher",
            "Enabled": true
        }
    ]
    ```

3. **Set your GameUserSettings class to `UpscalerGameUserSettings`** in
   *Project Settings → Engine → General Settings → GameUserSettingsClass*, or directly in
   `DefaultEngine.ini`:

    ```ini
    [/Script/EngineSettings.GameSessionSettings]
    GameUserSettingsClassName=/Script/UpscalerSwitcher.UpscalerGameUserSettings
    ```

    This is not optional. Almost every function in `UUpscalerSwitcherUtils` reads the game user
    settings object, and without this override the cast returns null and every call logs a warning and
    does nothing. **If the plugin appears to do nothing at all, check this first.**

4. Restart the editor. A C++ project compiles the plugin on the way in; a Blueprint-only project uses
   the binaries that ship with it.
5. Install the vendor plugins you want to support. Neither is required.
    - 🔷 DLSS: <https://developer.nvidia.com/rtx/dlss> — enable the `DLSS`, `Streamline`,
      `StreamlineCore` and `StreamlineDLSSG` plugins. The last one is what provides Frame Generation.
    - 🔶 FSR: <https://gpuopen.com/learn/ue-fsr/> — enable the `FSR`
      plugin (named `FSR3` on the 3.x generation).
6. If you call NVIDIA's Blueprint library from your own C++, add `DLSSBlueprint` to your `Build.cs`:

    ```csharp
    PrivateDependencyModuleNames.AddRange(new string[] { "DLSSBlueprint" });
    ```

## Quick start (Blueprint)

No C++ is required to use any of this. Every function below is a Blueprint node; right-click in any
graph and look under the **Upscaler Switcher** category, or type the function name into the search box
(the palette spells them with spaces — `ApplyFSR` is *Apply FSR*, `GetCurrentFPS` is *Get Current FPS*).

The three pieces of a working settings menu:

**1. Apply the saved choice when the game starts.** Set your GameInstance class to
`UpscalerGameInstance` in *Project Settings → Project → Maps & Modes → GameInstanceClass*, or reparent
your existing GameInstance Blueprint to it. That is the whole step — it applies the saved upscaler for
you at the right moment. (Doing it in your own GameInstance instead is
[described below](#applying-the-saved-upscaler-on-game-start), and the timing is fussier than it looks.)

**2. Read the current state to populate the menu.** All of these are pure nodes:

| To fill in | Call |
| --- | --- |
| Which upscaler is selected | `GetSavedUpscaler` |
| The saved quality mode | *Get Upscaler Game User Settings* → `GetDLSSQualityMode` / `GetFSRQualityMode` |
| A DLSS frame generation dropdown | `GetSupportedDLSSFrameGenModes` (see the warning below) |
| Whether to grey out DLSS frame generation | `GetDLSSFrameGenSupported` |
| Whether to say "restart required" | `GetFrameGenRestartRequired` |

**3. Write the player's choice back.** `ApplyDLSS`, `ApplyFSR` and `DisableUpscaling` switch upscaler;
`SetDLSSQuality`, `SetFSRQuality`, `SetDLSSFramegen`, `SetFSRFramegen` and `SetDLSSFrameGenMode` change
one setting each. All of them save automatically.

> ⚠️ **Populating a DLSS frame generation dropdown: never cast the combo box index to the enum.**
> `GetSupportedDLSSFrameGenModes` returns a **subset** of `UDLSSGMode_Custom`, not a prefix of it. On a
> 40-series card it is `{Off, Auto, On2X}`, so index 2 is `On2X` while enum *value* 2 is `Dynamic`.
> Round-trip through the array itself — **Array Get** with the selected index to read a choice, **Array
> Find** to preselect the current one.
>
> Use this node instead of Streamline's own *Get Supported DLSS-FG Modes*. That node lives in a module
> which only loads at `PostEngineInit`, so a Blueprint referencing it fails to *compile on load* in a
> packaged game if the widget is reachable from a class default object — you get
> `Could not find a function named "GetSupportedDLSSGModes"` and a cascade of "Input is not an Enum"
> errors that never appear in PIE. `GetDLSSFrameGenMode` is the safe replacement for *Get DLSS-FG Mode*
> for the same reason.

## Choosing an upscaler (C++)

```cpp
#include "UpscalerSwitcherUtils.h"
#include "UpscalerGameUserSettings.h"

UUpscalerGameUserSettings* Settings = UUpscalerGameUserSettings::GetUpscalerGameUserSettings();

// Enable FSR. Any other upscaler is turned off first.
UUpscalerSwitcherUtils::ApplyFSR(FFSRModeInformation(Settings->GetFSRQualityMode(),
                                                     Settings->GetFSRFrameGenEnabled()));

// Enable DLSS. FSR is turned off first.
UUpscalerSwitcherUtils::ApplyDLSS(FDLSSModeInformation(Settings->GetDLSSQualityMode(),
                                                       Settings->GetDLSSOptimalScreenPercentage(),
                                                       Settings->GetDLSSFrameGenEnabled(),
                                                       Settings->GetDLSSFrameGenMode()));

// Back to native resolution.
UUpscalerSwitcherUtils::DisableUpscaling();

// Reapply whatever is saved. This is the single entry point for restoring state.
UUpscalerSwitcherUtils::ApplySavedUpscaler();
```

The first three save the choice by default; pass `bSaveConfig = false` to apply without persisting.
`ApplySavedUpscaler()` takes no arguments — it only reads.

### Quality settings

```cpp
UUpscalerSwitcherUtils::SetFSRQuality(EFFXFSR3QualityMode_Custom::Performance);
UUpscalerSwitcherUtils::SetDLSSQuality(UDLSSMode_Custom::Performance, OptimalScreenPercentage);
```

Both save the new value and reapply it immediately, but **only when that upscaler is the current
one**. Setting FSR's quality while DLSS is active saves the preference and logs why it was not
applied; it takes effect the next time FSR is selected. The same holds for the frame generation
setters below.

DLSS needs an optimal screen percentage for the mode you pick. Ask NVIDIA's library for it:

```cpp
#include "DLSSLibrary.h" // needs DLSSBlueprint in your Build.cs

bool bIsSupported, bIsFixedScreenPercentage;
float OptimalScreenPercentage, MinScreenPercentage, MaxScreenPercentage, OptimalSharpness;
UDLSSLibrary::GetDLSSModeInformation(
    Settings->GetDLSSQualityMode(), Settings->GetScreenResolution(),
    bIsSupported, OptimalScreenPercentage, bIsFixedScreenPercentage,
    MinScreenPercentage, MaxScreenPercentage, OptimalSharpness);

UUpscalerSwitcherUtils::SetDLSSQuality(Settings->GetDLSSQualityMode(), OptimalScreenPercentage);
```

### Applying several changes at once

Every setter takes two optional flags:

| Flag | Default | Effect |
| --- | --- | --- |
| `bSaveConfig` | `true` | Write `GameUserSettings.ini` now. |
| `bApplyUpscaling` | `true` | Re-apply the upscaler now, so the change takes effect. |

A settings menu that writes each control as the player touches it will re-apply once per control. To
batch instead, pass `bApplyUpscaling = false` on each setter and call `ApplySavedUpscaler()` once when
the player presses Apply:

```cpp
UUpscalerSwitcherUtils::SetDLSSQuality(Quality, OptimalScreenPercentage, true, /*bApplyUpscaling*/ false);
UUpscalerSwitcherUtils::SetDLSSFrameGenMode(Mode,                        true, /*bApplyUpscaling*/ false);
UUpscalerSwitcherUtils::ApplySavedUpscaler();
```

Applying more than once is safe either way — every console-variable write is read-compare-write, so a
value that has not changed is not written.

## Frame generation

Frame generation behaves differently from upscaling, and a settings menu has to account for it. Four
things to know before you build one.

### It does not run in the editor, or in PIE

Both vendors disable frame generation outright whenever `GIsEditor` is true — that is their decision,
not this plugin's. **Test frame generation in a packaged build, or with *Play → Standalone Game*.** In
PIE the toggles save correctly and do nothing visible, which is the most common "it's broken" report
and is not a fault.

### Only one vendor's frame generation runs at a time

Upscaling switches instantly and freely. Frame generation has one extra rule behind it, and it is worth
knowing because it shapes what your menu should say.

Frame generation belongs to whichever vendor created the game's DXGI swapchain, and Unreal uses exactly
one swapchain provider — the first registered one that supports the RHI. Streamline registers one and
the FSR plugin registers one, so only one frame generator can be live at a time. Out of the box which
one you got came down to module load order, with no diagnostic when the loser silently did nothing.
This plugin arbitrates that deterministically instead.

### DLSS frame generation engages a moment after you ask for it

Switching DLSS Frame Generation **on** is deferred until the game is genuinely running: the plugin
waits for a map to finish loading, then a short margin of frames, then one completed render frame.
Engaging it earlier faults the GPU or hangs the process outright during startup, so the wait is
deliberate and is not tunable. Switching it **off** is always immediate.

Two consequences for a settings menu:

- **Do not drive a checkbox from the live getter immediately after setting it.**
  `GetDLSSFrameGenEnabled()` reads Streamline's actual state, so for a moment after the player ticks the
  box it still reports `false` and the box appears to pop back off. Bind the checkbox to the **saved**
  value — `UUpscalerGameUserSettings::GetDLSSFrameGenEnabled()` — and use the live getter for a status
  readout, not for the control itself.
- **A saved DLSS-FG choice is not active on a main menu** if no map has loaded yet. It engages shortly
  after the first level does.

FSR Frame Generation has no such wait and applies inline.

### Switching the frame generation *vendor* at runtime

**This is on by default, and the plugin handles it for you.** It picks the swapchain provider
deterministically at startup from the saved settings, and when the player turns frame generation on
`ApplyFSR` / `ApplyDLSS` hand the swapchain over and rebuild the game viewport so the change takes effect
immediately. No ini setting is required and no restart is involved. It works in both directions and as
often as the player likes.

The switch retires the game window and builds a new one, which makes it a heavier operation than anything
else here, and how cleanly it goes depends on how the installed DLSS and FSR plugin versions hold on to
the viewport they attached to. **Test it in a packaged build of your project, with the plugin versions you
actually ship.** If it misbehaves there, you can turn it off in **Project Settings → Plugins → Upscaler
Switcher → "Enable Runtime Frame Generation Switching (Experimental)"** — hence the *Experimental* label.

Turning it off changes nothing else: upscaling still switches instantly, and the player's **saved** frame
generation choice still works on every launch. What they lose is changing frame generation vendor
mid-session — that takes effect on the next launch instead, so bind `GetFrameGenRestartRequired()` and
tell them. `GetRuntimeFrameGenSwitchEnabled()` reports which mode your project is in.

What it costs on screen depends on whether either vendor's frame generation has already attached to the
current viewport:

| Situation | What the player sees |
| --- | --- |
| Neither vendor has generated a frame yet | One brief hitch, like a resolution change. |
| Either vendor's frame generation has run | The game window is retired and rebuilt — a visible blink. |

The second row is not FSR-specific. Both vendors keep a reference to the viewport their frame generator
attached to and neither offers a way to release it, so once *either* has run, that viewport can only be
retired along with its window. In practice this means the first frame generation switch of a session is
cheap and every one after it blinks.

DLSS and FSR *upscaling* are unaffected by any of this and always switch instantly.

### What your settings menu should bind

`GetFSRFrameGenAvailable()` and `GetDLSSFrameGenAvailable()` report who owns the swapchain **right
now**. Use them to label a control — a `false` is not a refusal, and calling the apply will still switch
the vendor over.

```cpp
// Which vendor's frame generation is live right now. Label the controls with these -
// a false means "not the current owner", not "cannot be switched to".
UUpscalerSwitcherUtils::GetFSRFrameGenAvailable();
UUpscalerSwitcherUtils::GetDLSSFrameGenAvailable();

// Can this machine run DLSS Frame Generation at all, hardware-wise? THIS is the one to
// grey a control out on - no switching will change the answer.
UUpscalerSwitcherUtils::GetDLSSFrameGenSupported();

// Fallback: true if a requested frame generator did not come up and a restart would fix it.
UUpscalerSwitcherUtils::GetFrameGenRestartRequired();
```

`GetDLSSFrameGenSupported()` is about the hardware; the two `...Available()` getters fold in who
currently owns the swapchain. Disable a control on the first, label it on the second.

`GetFrameGenRestartRequired()` is true only when a restart would actually help: the player has asked for
a frame generator the other vendor currently owns. Turning frame generation off never raises it, and
neither does hardware that cannot run frame generation in the first place. In a project shipping only one
vendor's plugin it is always false and the player is never prompted. With runtime switching left on it is
rare, covering only the cases a live handover cannot reach — a call made before the game viewport exists,
or from a thread other than the game thread. If you switch it off it becomes the normal way a vendor
change is communicated, so bind it either way.

### Turning it on

```cpp
UUpscalerSwitcherUtils::SetFSRFramegen(true);

UUpscalerSwitcherUtils::SetDLSSFramegen(true);
UUpscalerSwitcherUtils::SetDLSSFrameGenMode(UDLSSGMode_Custom::On2X);
```

`SetDLSSFramegen` deliberately leaves the chosen multiplier alone, so toggling frame generation off and
back on returns to the mode the player picked. `SetDLSSFrameGenMode` implies the on/off flag — passing
`Off` also switches frame generation off — so the two saved fields can never contradict each other.

Populate a DLSS frame generation dropdown from `GetSupportedDLSSFrameGenModes()`, and read the
[index-vs-value warning](#quick-start-blueprint) above before you wire it up.

## Showing an FPS counter

An ordinary FPS counter goes **down** when frame generation is switched on. Generated frames are
produced inside `Present`, below the engine, so `stat fps` and every Blueprint frame time node count
rendered frames only — and rendering one frame costs more with generation running. Measured on a 4080
at 2560x1358 with DLSS Quality: ~160 fps rendered with DLSS-FG off, ~101 with On2X, while the rate
actually reaching the display roughly doubled.

```cpp
// The frame rate actually reaching the display, generated frames included.
UUpscalerSwitcherUtils::GetCurrentFPS();
```

Each vendor is asked for its own presented rate, so the figure is correct for DLSS, for FSR, and for
neither. Comparing it against `stat fps` is what tells you frame generation is working.

> One side effect worth knowing: while **FSR** frame generation is running, `stat fps` and `stat unit`
> also switch to reporting presented frames, because that is how the FSR plugin publishes the figure.
> They stop being a reading of rendering cost until it is switched off again. This is armed only when
> FSR frame generation is actually enabled, so projects that never use it are unaffected. DLSS does not
> have this side effect.

`GetDLSSFrameGenTiming()` additionally reports Streamline's own presented rate and frame count.

## Applying the saved upscaler on game start

The simplest route is to set your GameInstance class to `UpscalerGameInstance` (or derive from it),
which does this for you.

To do it in your own GameInstance instead:

```cpp
// YourGameInstance.h
UCLASS()
class YOURPROJECT_API UYourGameInstance : public UGameInstance
{
    GENERATED_BODY()

public:
    virtual void Init() override;

protected:
    void PostEngineInit();
};
```

```cpp
// YourGameInstance.cpp
void UYourGameInstance::Init()
{
    Super::Init();

    if (GEngine && GEngine->IsInitialized())
    {
        PostEngineInit();
    }
    else
    {
        FCoreDelegates::OnPostEngineInit.AddUObject(this, &UYourGameInstance::PostEngineInit);
    }
}

void UYourGameInstance::PostEngineInit()
{
    // Wait one frame before applying.
    FTSTicker::GetCoreTicker().AddTicker(FTickerDelegate::CreateLambda([this](float DeltaTime)
    {
        if (UUpscalerGameUserSettings* Settings = UUpscalerGameUserSettings::GetUpscalerGameUserSettings())
        {
            Settings->ApplySettings(true);
        }
        return false; // don't repeat
    }), 0.0f);
}
```

> ⚠️ **The one frame delay matters — do not collapse it into a direct call.** Despite its name,
> `FCoreDelegates::OnPostEngineInit` fires *before* `PostEngineInit`-phase plugin modules are loaded.
> The DLSS plugin declares its modules at that phase, so `r.NGX.DLSS.Enable` does not exist yet when
> the delegate runs. Deferring to the first tick puts the work after all module loading is finished.

## Project defaults

*Project Settings → Plugins → Upscaler Switcher* sets what a player gets before they have chosen
anything: default upscaler, default quality mode per upscaler, and whether frame generation starts on.
These seed `UUpscalerGameUserSettings` the first time it is created, and are stored in
`DefaultEngine.ini`. The same page carries the *Enable Runtime Frame Generation Switching
(Experimental)* toggle described above, which is on by default.

## API reference

### `UUpscalerSwitcherUtils`

Every function below is Blueprint-callable, under the category **Upscaler Switcher**.

| Function | Purpose |
| --- | --- |
| `ApplyDLSS` / `ApplyFSR` / `DisableUpscaling` | Switch the active upscaler. |
| `ApplySavedUpscaler` | Reapply the saved choice. |
| `GetSavedUpscaler` | The saved choice, as `EUpscaler`. |
| `SetDLSSQuality` / `SetFSRQuality` | Change a quality mode. |
| `SetDLSSFramegen` / `SetFSRFramegen` | Turn a frame generator on or off. |
| `SetDLSSFrameGenMode` | Pick a DLSS multi frame generation mode. |
| `GetDLSSEnabled` / `GetFSREnabled` | Is that upscaler running right now. |
| `GetDLSSFrameGenEnabled` / `GetFSRFrameGenEnabled` | Is that frame generator running right now. |
| `GetDLSSFrameGenSupported` | Can this machine run DLSS Frame Generation. |
| `GetFSRFrameGenAvailable` | Does the FSR plugin currently own the swapchain. |
| `GetDLSSFrameGenAvailable` | Does Streamline currently own the swapchain, on hardware that supports DLSS-FG. |
| `GetFrameGenRestartRequired` | Is a restart needed for the saved frame generation choice. |
| `GetRuntimeFrameGenSwitchEnabled` | Is this project allowed to change frame generation vendor without a restart. |
| `GetSupportedDLSSFrameGenModes` | Modes to populate a DLSS frame generation dropdown with. |
| `GetDLSSFrameGenMode` | The DLSS frame generation mode running right now. |
| `GetCurrentFPS` | Frame rate reaching the display, generated frames included. |
| `GetDLSSFrameGenTiming` | Streamline's presented frame rate and frame count. |

The five `Set...` functions all take `bSaveConfig` and `bApplyUpscaling`, both defaulting to `true` —
see [Applying several changes at once](#applying-several-changes-at-once). `ApplyDLSS`, `ApplyFSR` and
`DisableUpscaling` take `bSaveConfig` only.

### `UUpscalerGameUserSettings`

Extends `UGameUserSettings`. Holds the saved upscaler, both quality modes, both frame generation
flags, the DLSS frame generation mode and the DLSS optimal screen percentage, with a getter and setter
for each. `ApplySettings()` applies them and calls `ApplySavedUpscaler()`.

The getters here return the **saved preference**; the `Get...Enabled` functions on
`UUpscalerSwitcherUtils` return the **live vendor state**. They differ on hardware that cannot honour a
saved choice, and briefly while DLSS frame generation is engaging. Bind controls to the saved value and
status readouts to the live one.

## Notes for advanced integrators

Things that are invisible in normal use, but worth knowing if you are integrating other rendering
plugins or debugging something odd.

**Module loading phase.** The module loads at `LoadingPhase: PreDefault`. That is load-bearing: it must
run after both vendors have registered their swapchain providers (`PostSplashScreen`) and before the game
viewport is first drawn. Do not change it.

**Swapchain handover is game-thread only.** `ApplyFSR` / `ApplyDLSS` will not hand the swapchain over from
another thread, or in the editor. Those are the cases `GetFrameGenRestartRequired()` falls back to.

**A Slate delegate is filtered while Streamline owns the swapchain.** In a project shipping **both**
vendor plugins, outside the editor, the plugin interposes on `FSlateRenderer::OnSlateWindowRendered` and
forwards it only while the FSR plugin owns the swapchain. This exists because the FSR plugin
unconditionally downcasts whatever custom present it finds to its own type, and with DLSS Frame
Generation active that object is Streamline's — a write roughly 200 bytes past the end of an allocation
Streamline made, twice per frame, which corrupts the heap and crashes the game at exit in a different
module every time. If you have your own subscriber to that delegate, it will also stop being called while
Streamline owns the swapchain. The shim is not installed in the editor, nor in a project shipping only one
vendor. When it is installed the log says so once:
`Routed OnSlateWindowRendered through this plugin`.

**Console variables are never written directly.** AMD renamed FSR's variables between plugin
generations — `r.FidelityFX.FSR3.*` on 3.x, `r.FidelityFX.FSR.*` on 4.x — and ships no aliases, so the
plugin resolves the prefix at runtime. Writing a console variable that does not exist fails silently, so
go through `UUpscalerSwitcherUtils` rather than setting them yourself.

**Enum member order is a wire format.** `EFFXFSR3QualityMode_Custom` is cast straight into the FSR
quality mode console variable. Reordering or inserting a value silently remaps quality modes.

## Troubleshooting

Plugin log categories: **`LogUpscalerUtils`** (most of it), `LogUpscalerDLSSWrapper` (the reflection
calls into NVIDIA's library), `LogUpscalerGameInstance`. Vendor categories worth watching alongside them
are `LogDLSS`, `LogStreamline` and `LogFFXFI`.

| Symptom | Cause |
| --- | --- |
| Nothing happens, log says *"default GameUserSettings is not set to UUpscalerGameUserSettings"* | Step 3 of Installation was skipped. |
| Frame generation does nothing in the editor or PIE | Expected. Both vendors disable it whenever `GIsEditor` is true. Test in a packaged build or *Play → Standalone Game*. |
| A DLSS frame generation checkbox flicks itself back off | It is bound to the live getter. Engaging DLSS-FG is deferred until after a map load — bind the control to the saved value instead. |
| FSR Frame Generation does nothing | Check `GetFSRFrameGenAvailable()`, and confirm you are not in the editor. |
| DLSS Frame Generation does nothing | Check `GetDLSSFrameGenSupported()` first — that is the hardware answer. If it is true, look for `now owns the game's swapchain` in the log: only one vendor's frame generation runs at a time, and `ApplyDLSS` switches ownership only when frame generation is actually requested. |
| DLSS frame generation dropdown only offers "Off" | Streamline is not in this session, so there is nothing to ask about DLSS-FG. |
| Changing frame generation vendor does nothing until restart | *Enable Runtime Frame Generation Switching* has been switched off, or the call was made off the game thread. Bind `GetFrameGenRestartRequired()` so the player is told. |
| The window blinks when switching frame generation | Expected once either vendor's frame generator has run this session. See the table under [Runtime switching](#switching-the-frame-generation-vendor-at-runtime). |
| FPS counter drops when frame generation is on | Expected — bind it to `GetCurrentFPS()`. |
| `stat fps` reports a suspiciously high number | FSR frame generation is running; it publishes presented frames through the engine's own average. |
| A Blueprint fails to compile on load in a packaged game with *Could not find a function named "GetSupportedDLSSGModes"* | The widget references a Streamline Blueprint node directly. Use `GetSupportedDLSSFrameGenModes()` instead. |
| FSR calls do nothing on one FSR plugin generation | AMD renamed the console variables between generations. The plugin resolves both spellings at runtime; make sure you are going through `UUpscalerSwitcherUtils` rather than setting them yourself. |

---

Penguru Games © 2026 Mehmet Furkan Gülmez. All Rights Reserved.
