# MoriFixPLC

**A portable hardware abstraction layer and framework for industrial PLCs.**

Business logic reads `.Value`. The same patterns are in a Rockwell Studio 5000 project, a Siemens TIA Portal project, and a Beckhoff TwinCAT project.

![Platforms](https://img.shields.io/badge/platforms-Studio%205000%20v35%20%7C%20TIA%20Portal%20V20%20%7C%20TwinCAT%203.1-lightgrey)

The sources in the TwinCAT project are marked **CMo Library v1.0**. There is one shared framework tag per PLC. Function blocks take that tag and a signal structure, and your program stays on `.Value`.

<!-- TODO(owner): Add one screenshot per platform of the same demo running — MAIN in TwinCAT, and the equivalent call of CMoMainRuntime plus the part-present / guard / motor-run logic in Studio 5000 and TIA Portal. No vendor IDE was available to capture those images. -->

## What this is

MoriFixPLC abstracts two things a PLC vendor leaves in your code:

1. **Your I/O.** Every digital signal supports hardware, simulation, and manual override (hand) modes. Business logic reads `.Value` and stays out of the mode logic.
2. **Your platform.** Rockwell, Siemens, and Beckhoff each read the controller clock differently. Business logic reads `Framework.CurrentLocalDateTime` and `Framework.CurrentUTCDateTime`.

The TwinCAT `MAIN` program is a working demo of that split: simulated part-present and guard-closed inputs, a two-point interlock, and a motor-run output. The demo turns simulation on, so the online view is enough to watch it.

## The .Value rule

**Business logic reads `.Value` only.**

Leave `.Input`, `.SimInput`, `.HandValue`, `.QualityInput`, and the other plumbing fields to the HAL block. `CMoDigitalInput` resolves the mode, filters the signal, evaluates quality, and writes `.Value`. Application code consumes `.Value`.

On outputs, business logic **writes** `.Value`. `CMoDigitalOutput` applies polarity, hand mode, and simulation capture, then exposes the physical coil as `DeviceOutput`.

Hand mode is active when `EnableHand` is set and `Framework.MaintenanceEnabled` is set. Simulation is active when `EnableSimulation` is set and `Framework.SimulationEnabled` is set. On a digital input, an active hand mode overrides the debounced hardware or simulation value.

## Quick start

Open the controller project for your IDE. Paths below are the files in the repo.

| Platform | Open this | Saved with |
|---|---|---|
| Beckhoff | [`Beckhoff TwinCat XAE/CMoLibrary_Dev/CMoLibrary_Dev.tsproj`](Beckhoff%20TwinCat%20XAE/CMoLibrary_Dev/CMoLibrary_Dev.tsproj) | TwinCAT 3.1.4024 |
| Siemens | [`Siemens TIA V20/CMoLibrary_Dev/CMoLibrary_Dev.ap20`](Siemens%20TIA%20V20/CMoLibrary_Dev/CMoLibrary_Dev.ap20) | TIA Portal V20 |
| Rockwell | [`Rockwell V35/CMoLib.ACD`](Rockwell%20V35/CMoLib.ACD) | Studio 5000 V35.00.00 |

`CMoMainRuntime` is the heartbeat. Call it **once per scan, first**, before the other blocks:

```iecst
fbRuntime(Framework := GVL.Framework);  // first line of MAIN
```

### Beckhoff TwinCAT

1. Install TwinCAT 3 XAE Shell from [beckhoff.com](https://www.beckhoff.com/). The project file declares `TcVersion` 3.1.4024.74.
2. Open `Beckhoff TwinCat XAE/CMoLibrary_Dev/CMoLibrary_Dev.tsproj`.
3. The PLC project is `Beckhoff TwinCat XAE/CMoLibrary_Dev/CMoLib/CMoLib.plcproj` (name `CMoLib`). It references `Tc2_Standard`, `Tc2_System`, `Tc2_Utilities`, and `Tc3_Module`.
4. Build, activate the configuration, log in, and start.
5. `PlcTask` already calls `MAIN`. The first statement in `MAIN` is `fbMainRuntime(Framework := GVL.Framework)`.

Online, set the local variables `SimPartPresent` and `SimGuardClosed`. `GVL.Interlock_Motor.OK` follows those two conditions, and `GVL.DO_MotorRun.Value` follows the interlock. `saveStart` and `saveEnd` capture `Framework.CurrentUTCDateTime`; `CMoDiffTime` then writes `totSec` and `totuSec`.

### Siemens TIA Portal

1. Install TIA Portal V20 and PLCSIM from Siemens support. The project compatibility version is 20.0.0.0.
2. Open `Siemens TIA V20/CMoLibrary_Dev/CMoLibrary_Dev.ap20` (project name `CMoLibrary_Dev`).
3. The station in the project is an S7-1500 **CPU 1516-3 PN/DP**, order number 6ES7 516-3AN00-0AB0.
4. Compile and download to PLCSIM or to that CPU.
5. Call `CMoMainRuntime` once per scan from OB1 before the other CMo blocks, and pass the framework tag. The project database names the Phase 1 blocks, including `CMoAlarm`.

The TIA project contains call sites for `RD_SYS_T` and `RD_LOC_T`.

### Rockwell Studio 5000

1. Open `Rockwell V35/CMoLib.ACD` in Studio 5000 v35. The file header was written by RSLogix 5000 and saved as V35.00.00.
2. Download to a controller or to Logix Echo.
3. Call `CMoMainRuntime` once per scan, first in the task, with the framework tag.

The ACD is a binary project. Block names in the sections below come from the TwinCAT Structured Text and from strings in the Siemens project data. The ACD references `RxWallClockTime`, which is the Logix wall-clock object used with GSV.

## Try the demo

This is the signal path in [`Beckhoff TwinCat XAE/CMoLibrary_Dev/CMoLib/POUs/MAIN.TcPOU`](Beckhoff%20TwinCat%20XAE/CMoLibrary_Dev/CMoLib/POUs/MAIN.TcPOU). The global tags live in `GVL`: `Framework`, `DI_PartPresent`, `DI_GuardClosed`, `DO_MotorRun`, and `Interlock_Motor`.

```iecst
// Heartbeat: always first
fbMainRuntime(Framework := GVL.Framework);

GVL.Framework.SimulationEnabled := TRUE;

// --- Part Present sensor ---
GVL.DI_PartPresent.EnableSimulation := TRUE;
GVL.DI_PartPresent.SimInput := SimPartPresent;
GVL.DI_PartPresent.Name := 'Part Present';

fbPartPresent(
    HAL := GVL.DI_PartPresent,
    DeviceInput := FALSE,
    QualityInput := TRUE,
    Framework := GVL.Framework
);

// --- Guard Door sensor ---
GVL.DI_GuardClosed.EnableSimulation := TRUE;
GVL.DI_GuardClosed.SimInput := SimGuardClosed;
GVL.DI_GuardClosed.Name := 'Guard Closed';

fbGuardClosed(
    HAL := GVL.DI_GuardClosed,
    DeviceInput := FALSE,
    QualityInput := TRUE,
    Framework := GVL.Framework
);

// --- Interlock ---
GVL.Interlock_Motor.PointCount := 2;
GVL.Interlock_Motor.ConditionName[0] := 'Part Present';
GVL.Interlock_Motor.ConditionName[1] := 'Guard Closed';
GVL.Interlock_Motor.Condition[0] := GVL.DI_PartPresent.Value;
GVL.Interlock_Motor.Condition[1] := GVL.DI_GuardClosed.Value;

fbInterlock(Ilk := GVL.Interlock_Motor);

// --- Business logic: .Value only ---
GVL.DO_MotorRun.EnableSimulation := TRUE;
GVL.DO_MotorRun.Value := GVL.Interlock_Motor.OK;
GVL.DO_MotorRun.Name := 'Motor Run';

fbMotorRun(
    HAL := GVL.DO_MotorRun,
    QualityInput := TRUE,
    Framework := GVL.Framework
);
```

With simulation active, `CMoDigitalOutput` holds the physical `DeviceOutput` false and copies the commanded state to `SimOutput`. To drive a real coil, wire the block output:

```iecst
Module2:O.Data[0] := fbOutput.DeviceOutput;
```

That wiring comment is in `CMoDigitalOutput`.

## Why

After 25 years in industrial automation, the same problems keep showing up at every company, in every plant, on every platform:

- PLC code that can't be tested without physical hardware
- "Digital twins" that are just 3D renderings with no connection to real control logic
- Simulation tools that only run on a Windows desktop with a dongle
- No version control, no code review, no CI/CD
- Complete vendor lock-in on day one

The industry's standards bodies (OPC UA, Industry 4.0/5.0/6.0, PackML) promised interoperability and didn't deliver. MoriFixPLC is what you build when you stop waiting.

## Library status

### Phase 1 — Foundation

The TwinCAT PLC project contains **6 structure types** and **10 function blocks**, plus the `MAIN` demo above. The Siemens project names those same blocks and also **`CMoAlarm`**.

| Function block | In the TwinCAT sources | Named in the Siemens project |
|---|---|---|
| `CMoMainRuntime` | yes | yes |
| `CMoDigitalInput` | yes | yes |
| `CMoDigitalOutput` | yes | yes |
| `CMoEvalInterlock` | yes | yes |
| `CMoGetLocalDateTime` | yes | yes |
| `CMoGetUTCDateTime` | yes | yes |
| `CMoDateTimeToString` | yes | yes |
| `CMoDiffTime` | yes | yes |
| `CMoElapsedTime` | yes | yes |
| `CMoRandom` | yes | yes |
| `CMoAlarm` | | yes |

Structure types in both the TwinCAT sources and the Siemens project: `CMoFramework`, `CMoHALDigitalInput`, `CMoHALDigitalOutput`, `CMoInterlock`, `CMoDateTime`, `CMoDateTimeString`.

### Phase 2 — Analog and motion (in progress)

Analog I/O, and a motion HAL with runtime capability discovery.

### Phase 3 — Devices and IO-Link (planned)

Module status, IO-Link abstraction, device drivers.

### Phase 4 — Higher level (planned)

Mode select, open/close actuators, runtime tracking, conveyor logic.

`CMoFramework` already carries placeholders used by later phases: `ModuleStatusMinInterval` (5000 ms), `ModuleStatusRandomDistribution` (2000 ms), `ModuleStatusGraceTime` (30000 ms), and spare `NotUsedDI` / `NotUsedDO` instances.

<!-- TODO(owner): Cut a GitHub release that marks Phase 1 once you are ready to tag these sources. There is no release on the repository today. -->

## Phase 1 function blocks

### CMoMainRuntime

The system heartbeat. Call once per scan, first in your task.

It measures scan cycle time with `CMoElapsedTime` and stores `ElapsedMs` in `Framework.CycleTime`. It generates tick pulses at 100 ms, 1 s, and 10 s. It generates blinker signals at 100 ms, 250 ms, 500 ms, 1 s, and 2 s for HMI use. It updates UTC and local date/time and formats them through `CMoDateTimeToString`. It runs `CMoRandom` every scan. It latches alarm, simulation, and hand rollups. It sets `Framework.AlwaysTrue` and `Framework.AlwaysFalse`, and it increments `Framework.FreeRunningTimer` by one each scan.

**LastScan accumulator pattern.** `CMoMainRuntime` copies `*LastScan` fields into the published rollups at the start of the call, then clears the accumulators for the new scan. During the scan, `CMoDigitalInput` and `CMoDigitalOutput` set `AnySimulationOnLastScan` and `AnyHandOnLastScan`. The runtime comment also assigns alarm accumulators to `CMoAlarm`. The published flags (`AnyAlarmActive`, `AnyWarningActive`, `AnyFaultActive`, `AnyCriticalActive`, `AnyAudibleOn`, `AlarmCount`, `AnySimulationOn`, `AnyHandOn`) therefore reflect the previous scan, independent of block order in the current scan.

```iecst
fbRuntime(Framework := GVL.Framework);  // first line of MAIN
```

### CMoDigitalInput

Takes a raw hardware input and produces a conditioned `.Value`. That is the field business logic reads.

The TwinCAT block's inputs are `DeviceInput`, `QualityInput`, and in-out `HAL` (`CMoHALDigitalInput`) plus `Framework`.

**Signal conditioning pipeline:**

1. Determines hand and simulation from `EnableHand` / `MaintenanceEnabled` and `EnableSimulation` / `SimulationEnabled`.
2. Evaluates quality from hardware health. A spare point or an active simulation forces good quality. Otherwise `PointQuality` follows `QualityInput`.
3. Selects `SimInput` or the latched hardware `Input`, and applies `Inverted`.
4. Runs configurable on-delay (`TON`) and off-delay (`TOF`) debounce when `EnableDelay` is set, using `InFilterOnTimeMsec` and `InFilterOffTimeMsec`, with automatic resync.
5. Tracks `HandValue` from `.Value` while hand mode is off, so the transfer into hand is bumpless.
6. Gates on quality: bad quality forces `.Value` to `BadQualityValue`.
7. Generates `RisingEdge` and `FallingEdge` one-shot pulses.

```iecst
fbPartPresent(
    HAL          := GVL.DI_PartPresent,
    DeviceInput  := FALSE,
    QualityInput := TRUE,
    Framework    := GVL.Framework
);

// Business logic: only ever read .Value
IF GVL.DI_PartPresent.Value THEN
    // part is here
END_IF;
```

The block comment shows the hardware form: pass the module input as `DeviceInput` and the module health bit as `QualityInput`.

`Name` is a `STRING(28)` on the HAL structure, for the HMI.

### CMoDigitalOutput

The output side of the HAL. Business logic writes `.Value`. The function block produces `DeviceOutput`.

TwinCAT behavior:

- Polarity comes from `Inverted`, applied to `.Value` (a spare point forces the conditioned output false).
- Hand mode (`EnableHand`, `MaintenanceEnabled`, and not spare) substitutes `HandValue`. While hand is off, `HandValue` tracks `.Value`.
- **Simulation capture.** When simulation is active, `DeviceOutput` is false and `.SimOutput` holds the value the logic commanded (after hand and inversion). The physical output tag `HAL.Output` follows `DeviceOutput`.
- `PointQuality` is stored on the HAL. A spare point or an active simulation counts as good quality. Otherwise `PointQuality` follows `QualityInput`.

```iecst
GVL.DO_MotorRun.Value := GVL.Interlock_Motor.OK;

fbMotorRun(
    HAL          := GVL.DO_MotorRun,
    QualityInput := TRUE,
    Framework    := GVL.Framework
);
// physical coil:
// someOutput := fbMotorRun.DeviceOutput;
```

### CMoAlarm

Standardized alarm behavior, wired into the framework rollups.

`CMoAlarm` is in the Siemens TIA project (an S7-1500 block). The project data names these members: `TriggerCondition`, `Reset`, `CountReset`, `AlarmActive`, `InhibitActive`, `AlarmSilenced`, `AlarmCount`, `CountExceeded`, `AlarmTimer` (`TON_TIME`), `Framework`, and the group fields `IntGroupNo`, `IntID`, `GroupBit`, and `GroupWord`.

`CMoFramework` is the plant-wide side of that design:

- Rollup flags for any alarm, any audible, and separate warning, fault, and critical flags, each with a `*LastScan` accumulator.
- `AlarmCount` and `TempAlarmCount`.
- HMI commands `SilenceCmd`, `GlobalAlmRstCmd`, and `GlobalAlmRstEn`.
- `InhibitAlarms`, plus `InhibitAlarmGroups` as 16 `DWORD`s (the structure comment describes 512 groups as bit arrays).

The TwinCAT `CMoMainRuntime` comment describes the contract those alarms use: each alarm sets the `*LastScan` accumulators during the scan, and the runtime publishes them on the next call. Build alarms against that contract so every alarm in the plant shares one HMI faceplate.

### CMoEvalInterlock

A 16-condition AND gate with diagnostics.

Feed up to 16 boolean conditions (`PointCount`, clamped to 1..16; the structure default is 4). The block ANDs the active conditions. A bypassed condition counts as met. If the result drops from OK to not OK, `FirstOut` latches the index of the first failed, non-bypassed condition (`-1` when the interlock is OK).

Each condition has its own `Bypass`. `InBypass` and `BypassAlarm` are true when any active condition is bypassed. `ConditionMet` and `ConditionFaulted` are per-point status for an HMI popup. `ConditionName` is `STRING(28)` per point, so the popup can show "Part Present" instead of "Condition[7] = FALSE".

```iecst
GVL.Interlock_Motor.PointCount := 2;
GVL.Interlock_Motor.ConditionName[0] := 'Part Present';
GVL.Interlock_Motor.ConditionName[1] := 'Guard Closed';
GVL.Interlock_Motor.Condition[0] := GVL.DI_PartPresent.Value;
GVL.Interlock_Motor.Condition[1] := GVL.DI_GuardClosed.Value;

fbInterlock(Ilk := GVL.Interlock_Motor);

IF GVL.Interlock_Motor.OK THEN
    // all conditions met, safe to run
END_IF;
```

This is the building block for reactive logic. Are all the conditions met? If yes, proceed. If no, `FirstOut` says which one dropped.

### CMoGetLocalDateTime

Reads the controller clock into a `CMoDateTime`: year, month, day, hour, minute, second, and microsecond, plus `IsLocal`.

The platform-specific part is this block:

- **Beckhoff:** `FB_LocalSystemTime` from `Tc2_Utilities`. Milliseconds from the system time are scaled into `MicroSecond`. The block writes the structure when `bValid` is true.
- **Siemens:** the TIA project references `RD_LOC_T` (and `RD_SYS_T` for the system clock).
- **Rockwell:** GSV on `WallClockTime`. The ACD references `RxWallClockTime`.

`CMoMainRuntime` calls it every scan. The result lives in `Framework.CurrentLocalDateTime`.

### CMoGetUTCDateTime

Fills the same `CMoDateTime` structure with UTC (`IsLocal` false).

On Beckhoff the block reads local time with `FB_LocalSystemTime`, reads the timezone with `FB_GetTimeZoneInformation`, and converts with `FB_TzSpecificLocalTimeToSystemTime`. It publishes when local time is valid and the timezone read has succeeded. `CMoMainRuntime` calls it every scan into `Framework.CurrentUTCDateTime`.

Local time is for the operator. UTC is for logs and for anything that leaves the plant.

### CMoDateTimeToString

Takes a `CMoDateTime` and writes a `CMoDateTimeString`:

- `TimeString` — `HH:MM:SS` (`STRING(16)`)
- `DateString` — `YYYY-MM-DD` (`STRING(16)`)
- `HMIString` — `YYYY-MM-DD HH:MM:SS` (`STRING(32)`)

Integer-to-string conversion with zero-padding. Bind `HMIString` on the HMI. `CMoMainRuntime` does this for both the local and the UTC values (`Framework.CurrentLocalString` and `Framework.CurrentUTCString`).

### CMoDiffTime

Subtracts two `CMoDateTime` values.

The day count uses a Julian day number, so month lengths and leap years stay in the integer conversion. The time-of-day remainder borrows from seconds up through hours. `Result` holds the difference in `Day`, `Hour`, `Minute`, `Second`, and `MicroSecond`. `TotalSeconds` rolls that difference up into seconds. `TotalMicroseconds` is the microsecond component after the borrow (`diffUs` in the block).

`MAIN` uses it between two captured UTC timestamps:

```iecst
fDiffTime(StartTime := StartDate, EndTime := EndDate);
totSec := fDiffTime.TotalSeconds;
totuSec := fDiffTime.TotalMicroseconds;
```

Use it for how long an alarm has been active, how long since the last maintenance cycle, or how long between two events.

### CMoElapsedTime

Measures elapsed time between a `Start` rising edge and a `Stop` rising edge. `ElapsedMs` updates while `Running` is true. `Reset` clears the measurement and returns.

The block runs a `TON` with a 49-day preset and converts `ET` with `TIME_TO_REAL`. `CMoMainRuntime` uses it once per scan: stop the previous interval, copy `ElapsedMs` into `Framework.CycleTime`, reset, and start the next interval.

### CMoRandom

A Park-Miller minimal-standard generator. The recurrence is `next = (seed * 16807) MOD 2147483647`. `Out` is a `REAL` from the new seed divided by 2147483647. The default input seed is 48271. A seed of 0 or below is replaced with 1.

`CMoMainRuntime` calls it every scan:

```iecst
fbRandom(Seed := Framework.LastRandomSeed);
Framework.Random := fbRandom.Out;
Framework.LastRandomSeed := fbRandom.NewSeed;
```

`Framework.ModuleStatusRandomDistribution` is the hook for spreading module-status polls across scans. The same generator is there for simulated scenarios and timing jitter.

## Repository layout

```
Rockwell V35/CMoLib.ACD
Siemens TIA V20/CMoLibrary_Dev/CMoLibrary_Dev.ap20
Beckhoff TwinCat XAE/CMoLibrary_Dev/CMoLibrary_Dev.tsproj
Beckhoff TwinCat XAE/CMoLibrary_Dev/CMoLib/
    POUs/     function blocks and MAIN
    DUTs/     the six structure types
    GVLs/     GVL.TcGVL
    PlcTask.TcTTO
```

The readable Structured Text is the TwinCAT project. Siemens logic is stored in the TIA project databases. The Rockwell logic is stored in the ACD.

## Platform differences

The HAL structures and the block names match across the TwinCAT sources and the Siemens project. The clock read is platform-specific. Beckhoff string length and Beckhoff clock calls below are from the `.TcDUT` and `.TcPOU` files. Siemens `RD_SYS_T` and `RD_LOC_T` appear as call sites in the TIA project, and `CMoAlarm.AlarmTimer` is a `TON_TIME`. The Rockwell timer, string, `BIT` packing, and AOI notes, and the Siemens `STRING[28]` and `#` prefix notes, are the platform differences recorded with the library. Those details are not stored as readable text in the ACD or the TIA databases.

| Concern | Rockwell | Siemens | Beckhoff |
|---|---|---|---|
| Project in this repo | `Rockwell V35/CMoLib.ACD` | `Siemens TIA V20/CMoLibrary_Dev/CMoLibrary_Dev.ap20` | `Beckhoff TwinCat XAE/CMoLibrary_Dev/` |
| DateTime | GSV `WallClockTime` (`RxWallClockTime` in the ACD) | `RD_SYS_T` / `RD_LOC_T` | Local: `FB_LocalSystemTime`. UTC: `FB_GetTimeZoneInformation` and `FB_TzSpecificLocalTimeToSystemTime` |
| Timers | `TON` with `.PRE` / `.ACC` / `.DN` | `TON_TIME` (used by `CMoAlarm.AlarmTimer`) | `TON` / `TOF` with `IN` and `PT` |
| Strings | Custom `CMoString28` UDT | `STRING[28]` | `STRING(28)` |
| Variable prefix | (none) | `#` | (none) |
| Bool in a UDT | `BIT` with `ZZZZ` packing | `BOOL` | `BOOL` |
| File you edit | Studio 5000 ACD | TIA Portal project | `.TcPOU` / `.TcDUT` / `.TcGVL` |
| Block model | AOI (local tags, InOut) | FB + instance DB | FB (`VAR`, `VAR_IN_OUT`) |

## The series

This library is being built in public as part of the **#MoriFixPLC** series on LinkedIn. Each episode covers a piece of the architecture: what it does, why it exists, and what problem it solves.

Follow the series: [linkedin.com/in/c-mori](https://www.linkedin.com/in/c-mori/)

## Author

**Cristian Mori** — 25 years in industrial automation. PLC programmer, C# developer, software architect, automation director, VP, SVP. Every seat in the room. Every version of the same problem.

This library exists because the vendors should have built it and didn't.

## Related projects

Part of a family of open-source robotics / industrial-automation libraries by @CristianMori:

- **EtherNet/IP + CIP Safety:** [Python](https://github.com/CristianMori/EthernetIpPython) · [C#](https://github.com/CristianMori/EthernetIpSharp) · [Rust](https://github.com/CristianMori/EthernetIpRust) · [C++](https://github.com/CristianMori/EthernetIpCpp)
- **PLC tooling:** [MoriFixPlc](https://github.com/CristianMori/MoriFixPlc) (portable PLC HAL) · [EscaFlow](https://github.com/CristianMori/Mori.EscaFlow) (PLC transpiler)
- **ROS in pure C#:** [Mori.RosSharp](https://github.com/CristianMori/Mori.RosSharp) (ROS 1) · [Mori.Ros2Sharp](https://github.com/CristianMori/Mori.Ros2Sharp) (ROS 2)
- **Simulation & vision:** [GemelliStudio](https://github.com/CristianMori/GemelliStudio) · [LocateAnythingSharp](https://github.com/CristianMori/LocateAnythingSharp) · [Mori.SkyScope](https://github.com/CristianMori/Mori.SkyScope)
- **AI agents:** [Deckhand](https://github.com/CristianMori/DeckHand)

## License

No `LICENSE` file is in this repository, and GitHub reports no license.

<!-- TODO(owner): Choose a license and add a LICENSE file. Do not document MIT, Apache-2.0, or any other license in this README until that file exists. -->
