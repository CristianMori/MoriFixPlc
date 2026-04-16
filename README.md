# MoriFixPLC

**A portable Hardware Abstraction Layer and framework library for industrial control systems.**

Same code, same tests, three platforms: Rockwell (Studio 5000), Siemens (TIA Portal), Beckhoff (TwinCAT).

---

## What Is This?

MoriFixPLC is an open-source library that abstracts two things your PLC vendor won't abstract for you:

1. **Your I/O.** Every digital signal supports normal hardware, simulation, and manual override modes. Business logic reads `.Value` and never knows which mode is active.

2. **Your platform.** How does the controller get the current time? Rockwell uses GSV. Siemens uses RD_SYS_T. Beckhoff uses NT_GetTime. Your business logic doesn't care. It reads `Framework.CurrentLocalDateTime` and moves on.

The result: business logic that is vendor-independent, testable, and simulatable. Write it once. Run it on any platform. Test it without hardware.

---

## Why?

After 25 years in industrial automation, the same problems keep showing up at every company, in every plant, on every platform:

- PLC code that can't be tested without physical hardware
- "Digital twins" that are just 3D renderings with no connection to real control logic
- Simulation tools that only run on a Windows desktop with a dongle
- No version control, no code review, no CI/CD
- Complete vendor lock-in on day one

The industry's standards bodies (OPC UA, Industry 4.0/5.0/6.0, PackML) promised interoperability and didn't deliver. MoriFixPLC is what you build when you stop waiting.

---

## Library Status

### Phase 1 — Foundation (Complete)

7 data types (UDTs) and 11 function blocks, running on all three platforms.

### Phase 2 — Analog & Motion (In Progress)

Analog I/O, motion HAL with runtime capability discovery.

### Phase 3 — Devices & IO-Link (Planned)

Module status, IO-Link abstraction, device drivers.

### Phase 4 — Higher Level (Planned)

Mode select, open/close actuators, runtime tracking, conveyor logic.

---

## Phase 1 Function Blocks

### CMoMainRuntime

The system heartbeat. Call once per scan, first in your task.

Measures actual scan cycle time. Generates tick pulses at 100ms, 1s, and 10s. Generates blinker signals at 100ms, 250ms, 500ms, 1s, and 2s for HMI use. Updates UTC and local date/time with formatted strings. Runs the random number generator. Latches all alarm, simulation, and hand mode rollups.

Uses the **LastScan accumulator pattern**: every CMoAlarm and CMoDigitalInput sets flags into LastScan accumulators during the scan. CMoMainRuntime copies those into final rollup values at the start of the next scan. This eliminates execution order dependencies. It doesn't matter what order your blocks run. The rollups are always consistent.

```
fbRuntime(Framework := GVL.Framework);  // first line of MAIN
```

---

### CMoDigitalInput

Takes a raw hardware input and produces a conditioned `.Value`. That's the only field your business logic ever touches.

**Signal conditioning pipeline:**
1. Determines active mode (normal / simulation / hand)
2. Evaluates quality from hardware health
3. Selects source signal and applies polarity inversion
4. Runs configurable on/off debounce filtering with automatic resync
5. Tracks hand value with bumpless transfer (no glitch on mode switch)
6. Gates on quality: bad quality forces `.Value` to a configurable safe state
7. Generates RisingEdge and FallingEdge one-shot pulses

Every digital input in your system behaves identically. Same filtering. Same simulation path. Same quality chain. Same HMI interface. One faceplate template for every DI in the plant.

```
fbPartPresent(
    HAL         := GVL.DI_PartPresent,
    QualityInput := TRUE,
    Framework   := GVL.Framework
);

// Business logic: only ever read .Value
IF GVL.DI_PartPresent.Value THEN
    // part is here
END_IF;
```

---

### CMoDigitalOutput

The output side of the HAL. Business logic writes `.Value`. The function block handles everything else.

Supports **simulation capture**: when sim is active, the digital twin reads `.SimOutput` to see what the logic intended without driving real hardware. Hand mode lets operators override from HMI. Polarity inversion is configurable. Quality gating forces the output to a safe state on bad quality.

Digital twin integration is built in at the signal level. You don't need a separate simulation framework. Every output already knows how to talk to a twin.

```
GVL.DO_MotorRun.Value := GVL.Interlock_Motor.OK;

fbMotorRun(
    HAL         := GVL.DO_MotorRun,
    QualityInput := TRUE,
    Framework   := GVL.Framework
);
```

---

### CMoAlarm

Standardized alarm behavior across the entire system.

- **Trigger polarity:** alarm on TRUE or alarm on FALSE
- **Delay timer:** configurable, so transient conditions don't spam the operator
- **Reset modes:** auto-reset or manual-reset
- **Occurrence counting:** trend how often a condition trips
- **Severity classification:** critical, fault, warning
- **Group rollup:** aggregates into the Framework for system-wide alarm counts, inhibit states, and silence commands
- **Inhibit:** suppress during commissioning or maintenance without removing logic
- **Silence:** stop the audible without acknowledging the condition

Every alarm in the plant works the same way. Same HMI faceplate. Same behavior. No surprises.

---

### CMoEvalInterlock

A 16-condition AND gate with diagnostics.

Feed up to 16 boolean conditions (configurable via `PointCount`). The block ANDs them together. If the result drops from OK to NOT OK, it latches which condition failed first — your **first-out root cause**. The operator sees it immediately on the HMI without digging through alarm logs.

Each condition supports **individual bypass** from HMI. If a sensor is bad during commissioning, bypass that one condition, not the whole interlock. A bypass alarm fires to remind you it's active.

Condition names are stored in the UDT, so the HMI popup shows human-readable text instead of "Condition[7] = FALSE".

This is the building block for **reactive logic**. No sequences. No step counters. Just: are all the conditions met? If yes, proceed. If no, here's exactly why.

```
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

---

### CMoGetLocalDateTime

Reads the controller's system clock and populates a `CMoDateTime` structure: year, month, day, hour, minute, second, microsecond.

The platform-specific part is isolated to one call:
- **Rockwell:** GSV WallClockTime
- **Siemens:** RD_SYS_T + RD_LOC_T
- **Beckhoff:** NT_GetTime + FB_LocalSystemTime

Called every scan by CMoMainRuntime. The result lives in `Framework.CurrentLocalDateTime`.

---

### CMoGetUTCDateTime

Identical to CMoGetLocalDateTime, returns UTC instead of local time.

UTC matters when your plant spans time zones or when you're correlating events with external systems. Local time is for the operator. UTC is for the data.

Both are updated every scan. Both available in the Framework.

---

### CMoDateTimeToString

Takes a `CMoDateTime` and produces three formatted strings:
- `TimeString` → "HH:MM:SS"
- `DateString` → "YYYY-MM-DD"
- `HMIString` → "YYYY-MM-DD HH:MM:SS"

Pure integer-to-string conversion with zero-padding. No platform-specific formatting libraries. Works the same everywhere. Bind directly to HMI displays with no parsing or formatting on the HMI side.

---

### CMoDiffTime

Subtracts two `CMoDateTime` structures and returns the difference in days, hours, minutes, seconds, and microseconds.

Full carry propagation: if microseconds go negative, borrow from seconds. If seconds go negative, borrow from minutes. All the way up to days.

Use cases: how long has this alarm been active? How long since the last maintenance cycle? How long between two events?

---

### CMoElapsedTime

Measures elapsed time between a Start and Stop rising edge. Live readout while running. Reset to zero on demand.

Uses a TON timer internally for millisecond resolution. CMoMainRuntime uses it to measure actual scan cycle time every scan: stop previous → read elapsed → reset → start new.

---

### CMoRandom

Generates a pseudo-random number between 0.0 and 1.0 using a Park-Miller linear congruential generator.

Feed it a seed, get back a new seed and a random value. CMoMainRuntime calls it every scan, storing the result in `Framework.Random` with the seed persisting in `Framework.LastRandomSeed`.

Why does a PLC need a random number generator? Randomized module status checks (spread polling load across scans). Randomized test scenarios for simulation. Jitter injection for timing tests. It's a utility you don't need until you need it.

---

## The .Value Rule

This is the core principle of the entire library:

**Business logic reads `.Value` only.**

Never touch `.Input`, `.SimInput`, `.HandValue`, `.QualityInput`, or any internal field. The HAL function block resolves the active mode, applies filtering, evaluates quality, and writes the result to `.Value`. Your application code consumes `.Value` and nothing else.

This is what makes simulation, hand mode, digital twin integration, and vendor portability possible. The business logic doesn't know and doesn't care where the signal came from.

---

## Platform Differences

The logic is identical across all three platforms. What changes:

| Concern | Rockwell | Siemens | Beckhoff |
|---|---|---|---|
| DateTime | GSV WallClockTime | RD_SYS_T / RD_LOC_T | NT_GetTime / FB_LocalSystemTime |
| Timers | TON with .PRE/.ACC/.DN | TON_TIME | TON with IN:=/PT:= |
| Strings | Custom CMoString28 UDT | STRING[28] | STRING(28) |
| Variable prefix | (none) | # | (none) |
| Bool in UDT | BIT with ZZZZ packing | BOOL | BOOL |
| Export format | L5X (XML) | SCL source | .TcPOU / .TcDUT (XML) |
| FB data model | AOI (local tags, InOut) | FB + Instance DB | FB (VAR, VAR_IN_OUT) |

---

## Project Structure

```
MoriFixPLC/
├── Beckhoff/
│   ├── DUTs/           # Data type definitions (.TcDUT)
│   ├── POUs/           # Function blocks (.TcPOU)
│   └── GVLs/           # Global variable lists
├── Siemens/
│   ├── UDTs/           # PLC data types
│   └── FBs/            # Function blocks (.scl)
├── Rockwell/
│   ├── UDTs/           # Data types (.L5X)
│   └── AOIs/           # Add-On Instructions (.L5X)
└── README.md
```

---

## Getting Started

### Beckhoff (Free, no expiration)

1. Download TwinCAT 3 XAE Shell from beckhoff.com
2. Create a new TwinCAT XAE Project
3. Add a Standard PLC Project
4. Import DUTs in dependency order, then POUs
5. Build, activate, login, start

### Siemens (21-day trial)

1. Download TIA Portal V20 trial + PLCSIM from Siemens support
2. Create a new project with an S7-1500 CPU
3. Add PLC data types manually in the table editor
4. Create FBs in SCL, paste code
5. Compile and download to PLCSIM

### Rockwell (License required)

1. Open Studio 5000 v35+
2. File → Import Component for each L5X file
3. Import UDTs first (dependency order), then AOIs
4. Create controller tags and wire up MAIN

---

## The Series

This library is being built in public as part of the **#MoriFixPLC** series on LinkedIn. Each episode covers a piece of the architecture: what it does, why it exists, and what problem it solves.

Follow the series: [linkedin.com/in/c-mori](https://www.linkedin.com/in/c-mori/)

---

## License

MIT

---

## Author

**Cristian Mori** — 25 years in industrial automation. PLC programmer, C# developer, software architect, automation director, VP, SVP. Every seat in the room. Every version of the same problem.

This library exists because the vendors should have built it and didn't.
