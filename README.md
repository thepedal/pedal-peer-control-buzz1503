# Pedal Peer Control (Buzz 1503)

**Pedal Peer Control** is a managed control machine for **Jeskola Buzz build
1503 (32-bit)**, installed at `C:\Program Files (x86)\Jeskola\Buzz`. It is a
C# re-implementation of the classic **BTDSys PeerCtrl v1.5/1.6**. It is not
intended for ReBuzz.

It appears in the machine browser as **Pedal Peer Control** and deploys as
`Pedal Peer Control.NET.dll`.

> Pedal Peer Control is an independent port. It is not affiliated with or
> endorsed by BTDSys or Ed Powley.

The code comes from the ReBuzz port
([BTDSys-PeerCtrl-ReBuzz](https://github.com/thepedal/BTDSys-PeerCtrl-ReBuzz)),
retargeted to Buzz 1503. See *Port notes* below.

---

## What is Peer Control?

PeerCtrl is a **general-purpose control machine** – it controls the parameters
of other machines in the session without producing any audio of its own.  
Classic use-cases include:

| Goal | How |
|------|-----|
| Drive several parameters at once from one slider | Add one assignment per target parameter on the same track |
| Smooth (glide) any parameter | Set the **Inertia** global parameter |
| Non-linear mapping | Draw a curve in the **Mapping** editor |
| MIDI CC → Buzz parameter | Assign a CC number per track |
| Hardware controller feedback | Enable **Feedback** – controller lights snap to saved positions on reload |
| Endless rotary encoders (inc/dec) | Enable **Inc/Dec** mode on the MIDI panel |
| Tie tracks together | Enable **Slaved** on a track |

---

## Feature mapping vs. the original

| Original feature | Port status |
|---|---|
| Up to 255 tracks | 64 tracks (`MAX_TRACKS` constant) |
| Global Inertia parameter | ✅ identical behaviour |
| Per-track Slaved parameter | ✅ identical behaviour |
| Per-track Value parameter (0–65534) | ✅ identical behaviour |
| Multiple assignments per track | ✅ full support |
| Piecewise-linear mapping curve | ✅ interactive WPF editor with Mirror/Invert/Reset |
| MIDI CC input (absolute) | ✅ |
| MIDI CC learn (click Learn, move control) | ✅ |
| MIDI CC inc/dec (CC 96/97) | ✅ |
| MIDI inc/dec wrap mode | ✅ |
| MIDI feedback output | ✅ sends to every MIDI output device that will open |
| Ctrl Rate attribute (sub-tick updates) | Mapped to **Send Freq** setting |
| Stop on Mute attribute | ✅ |
| Plugin interface (XY pad, Mixer GUI…) | Not ported |
| bmx/bmxml state persistence | ✅ via `MachineState` / XML serialisation |
| ImportFinished (rename fix-up) | ✅ |

---

## Parameters

### Global
| Name | Range | Default | Description |
|------|-------|---------|-------------|
| Inertia | 0–1280 | 0 | Glide time in ticks/10. 0 = off; 10 = 1 tick; 1280 = 128 ticks |

### Track (per track)
| Name | Range | Default | Description |
|------|-------|---------|-------------|
| Slaved | 0/1 | 0 | When 1, this track mirrors the previous track's Value |
| Value | 0–65534 | 32767 | The 0–100% value sent through the mapping to the target parameter |

### Attributes (in Assignment Settings dialog)
| Name | Range | Default | Description |
|------|-------|---------|-------------|
| MIDI Inc/Dec Amount | 0–65534 | 1024 | Step size for endless-encoder CC 96/97 messages |
| Send Freq | 0–64 | 2 | How often (in Work() calls) inertia is pushed out. 0 = tick only |
| Stop on mute | 0/1 | 0 | When 1, no control changes are sent while the machine is muted |

---

## Assignment Settings dialog

Open with **right-click → Assignment Settings…**

The dialog shows one instance at a time; opening it again while it is
already visible brings it to the front.

### Track selector

Tracks that already have assignments are shown in **bright green**.
Empty tracks are shown in the default colour. This lets you see at a
glance which slots are free before adding a new assignment.

Machines in the Machine dropdown are listed **alphabetically**.

### MIDI Learn

1. Select an assignment in the list.
2. Click **Learn** (turns red).
3. Move a knob or fader on your hardware controller.
4. The Controller and Channel fields update automatically.

### Keyboard navigation in dropdowns

After selecting an item with the mouse, you can immediately use:
- **Arrow keys** – move one item at a time
- **Page Up / Page Down** – jump a page at a time
- **Enter** – confirm selection
- **Escape** – close the dropdown

### Mapping curve editor

- **Left-click** on empty canvas → adds a control point
- **Drag** an existing point → moves it
- **Right-click** on a point → removes it (endpoints are locked)
- **Right-click** on empty canvas → context menu with:
  - **Mirror** – flip the curve horizontally
  - **Invert** – flip the curve vertically
  - **Reset to linear** – restore the default straight line

The default mapping is linear: `Value=0 → 0%` of parameter range,
`Value=65534 → 100%`.

### MIDI Feedback

Enable **Feedback** on an assignment to send MIDI CC messages back to
the hardware whenever the parameter value changes. On song reload,
saved positions are transmitted to all MIDI output devices so controller
lights snap to the correct positions automatically.

Feedback is sent to **all** MIDI output devices simultaneously.
This avoids the need for a device selector and ensures hardware
connected on any port receives the update.

A device that refuses to open is skipped until the number of MIDI output
devices changes. This happens when a single-client driver is already held by
Buzz (for example, if the port is enabled as a MIDI output in Buzz's
preferences) or by another application. If feedback doesn't reach your
controller, check that nothing else has its output port open.

> **BCR2000 note:** use the BCR2000 in **absolute CC mode** (0–127).
> The **Inc/Dec** option is for Doepfer Pocket Dial-style endless
> encoders only (CC 96/97 protocol).

---

## Inertia

When **Inertia** is non-zero, every value change is interpolated over
`Inertia / 10` ticks. The step size comes from the host's `SamplesPerTick`.
The control-machine `Work()` has no sample count, so the samples elapsed
between calls are measured with a high-resolution clock and the song's
sample rate.

Setting **Inertia = 0** in a pattern immediately snaps all tracks to
their target values.

---

## Requirements

- A .NET SDK (any current version builds SDK-style `net48`).
- Buzz 1503 installed at `C:\Program Files (x86)\Jeskola\Buzz`. Its
  `BuzzGUI.Interfaces.dll` is the only host reference.
- Internet access on the first build, to restore
  `Microsoft.NETFramework.ReferenceAssemblies` (so the .NET Framework 4.8
  targeting pack doesn't need to be installed).

## Building

From an elevated prompt (the output goes under Program Files):

```powershell
dotnet build PedalPeerControl.csproj -c Release
```

`BuzzDir` defaults to `C:\Program Files (x86)\Jeskola\Buzz` (set in
`Directory.Build.props`). Override it with `-p:BuzzDir=…` if Buzz is elsewhere.

## What gets deployed

Only the `.dll` is deployed. No `.pdb` or `.deps.json` is produced.

| File | Destination |
|---|---|
| `Pedal Peer Control.NET.dll` | `C:\Program Files (x86)\Jeskola\Buzz\Gear\Generators` |

---

## Port notes (from the ReBuzz version)

**.NET version.** Targets `net48`, a deliberate exception to the ".NET 10 or
higher" rule: Buzz 1503 runs managed machines on the .NET Framework 4 CLR and
cannot load a .NET 10 assembly. `AnyCPU`, so it loads as 32-bit in Buzz 1503.
The source uses no .NET Core-only APIs (no `Math.Clamp`, `MathF`, `Span<T>`).

**References.** Only Buzz 1503's `BuzzGUI.Interfaces.dll`, which exports both
`Buzz.MachineInterface` and `BuzzGUI.Interfaces`. The ReBuzz build's
`ReBuzz.dll`, `BuzzGUI.Common.dll` (unused) and NAudio references are gone.

**Work signature.** Now `public void Work()`, the managed API's documented
control-machine form. The ReBuzz version used the block generator form
`bool Work(Sample[], int, WorkModes)`, which declares an audio generator.

**No Tick().** `Tick()` is not part of the documented managed-machine surface.
Its jobs moved:

- Post-load work (resolve targets, send saved MIDI feedback, sync from the
  `Value` parameters) runs on the first `Work()` after construction,
  `MachineState.set` or `ImportFinished`.
- Unresolved targets are retried every 64 `Work()` calls.

**MIDI feedback.** Uses `winmm` (`midiOutOpen` / `midiOutShortMsg`) directly
instead of NAudio, so the machine stays a single DLL with no dependency on
NAudio being present.

**Saved songs.** Songs made with the ReBuzz `BTDSys PeerCtrl` build don't carry
over: the machine name and the host are both different.

## Test checklist (Buzz 1503)

Not yet verified on Buzz 1503:

1. **Load.** The machine appears under Generators with no audio output plug and
   no `BadImageFormatException`.
2. **Work timing.** `Work()` is called when the song is stopped as well as
   playing. Check that an Inertia glide completes from a slider drag while
   stopped, and that its duration matches `Inertia / 10` ticks.
3. **Settings dialog.** Opens from right-click → Assignment Settings…, lists
   machines and parameters, and assignments take effect. The dialog runs on
   its own STA thread and reads the song graph from there.
4. **Save / reload.** Assignments, mapping curves and settings survive a
   save and reload, and targets resolve after loading.
5. **Template import / clone.** Assignments follow renamed targets
   (`ImportFinished`).
6. **MIDI.** CC input, Learn, Inc/Dec, and feedback on reload (with the
   controller's output port not held by Buzz).

---

## File overview

| File | Purpose |
|------|---------|
| `PeerCtrl.cs` | Machine class, track state, data structures, inertia, MIDI |
| `SettingsWindow.xaml.cs` | Code-only settings dialog + `CurveCanvas` + `DarkCombo` |
| `PedalPeerControl.csproj` | `net48` / WPF project for Buzz 1503 |
| `Directory.Build.props` | Sets `BuzzDir` |

---

## Licence

BSD 3-Clause, matching the original BTDSys source. See `LICENSE`.
