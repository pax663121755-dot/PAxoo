# FB_TemperatureControl – TwinCAT 3 temperature control library

A self-contained TwinCAT 3 PLC library for closed-loop temperature regulation with heating and cooling outputs, PID control, and relay-feedback autotune.

---

## File structure

```
TC_TemperatureControl.plcproj   ← TwinCAT 3 PLC project file
DUTs/
  DUT_TemperatureControlParams  ← All tunable parameters (VAR_IN_OUT to FB)
  E_AutotuneState               ← Enum for autotune state machine
POUs/
  FB_Autotune                   ← Relay-feedback autotuner (Åström–Hägglund)
  FB_TemperatureControl         ← Main temperature control block
  MAIN                          ← Example wiring program
```

---

## FB_TemperatureControl

### Inputs

| Variable | Type | Description |
|---|---|---|
| `EN` | `BOOL` | Block enable (EN/ENO pattern) |
| `rProcessValue` | `REAL` | Measured temperature [°C] |
| `tCycleTime` | `TIME` | PLC task cycle time (default `T#10MS`) |

### VAR_IN_OUT (parameters – linked to HMI)

All parameters live in `stParams : DUT_TemperatureControlParams`.

#### Setpoint & safety

| Parameter | Default | Description |
|---|---|---|
| `rSetpoint` | 25.0 | Target temperature [°C] |
| `rDeadband` | 0.2 | Heat/cool digital output deadband [°C] |
| `rLimitHigh` | 80.0 | High-temperature alarm threshold [°C] |
| `rLimitLow` | 0.0 | Low-temperature alarm threshold [°C] |

#### PID tuning

| Parameter | Default | Description |
|---|---|---|
| `rKp` | 1.0 | Proportional gain |
| `rTi` | 20.0 | Integral time [s] (`0` = integrator off) |
| `rTd` | 0.0 | Derivative time [s] |
| `rTf` | 0.5 | Derivative filter time constant [s] |

#### Output limits & scaling

| Parameter | Default | Description |
|---|---|---|
| `rOutMax` | 100.0 | PID output maximum [%] |
| `rOutMin` | −100.0 | PID output minimum [%] (negative = cooling) |
| `rAnalogOutMax` | 10.0 | Physical analog output maximum [V or mA scale] |
| `rAnalogOutMin` | 0.0 | Physical analog output minimum |

#### Mode

| Parameter | Default | Description |
|---|---|---|
| `bAutoMode` | TRUE | `TRUE` = PID auto, `FALSE` = manual |
| `rManualOut` | 0.0 | Manual output value [%] |
| `bHeatEnable` | TRUE | Enable heating digital output |
| `bCoolEnable` | TRUE | Enable cooling digital output |

#### Autotune

| Parameter | Default | Description |
|---|---|---|
| `bAutotuneRequest` | FALSE | Rising edge starts autotune (self-clears) |
| `rAutotuneRelay` | 5.0 | Relay amplitude [%] |
| `rAutotuneNoise` | 0.1 | Relay switching noise band [°C] |
| `nAutotuneCycles` | 3 | Oscillation cycles to average |

### Outputs

| Variable | Type | Description |
|---|---|---|
| `ENO` | `BOOL` | Block enable output |
| `rAnalogOut` | `REAL` | Scaled physical analog output value |
| `rPidOut` | `REAL` | Raw PID output [%] |
| `bHeat` | `BOOL` | Heater digital output |
| `bCool` | `BOOL` | Cooler digital output |
| `rError` | `REAL` | Control error = setpoint − PV [°C] |
| `bAlarmHigh` | `BOOL` | PV above `rLimitHigh` |
| `bAlarmLow` | `BOOL` | PV below `rLimitLow` |
| `bAutotuneActive` | `BOOL` | Autotune in progress |
| `bAutotuneDone` | `BOOL` | Pulsed one cycle on successful completion |
| `bAutotuneError` | `BOOL` | Autotune failed (timeout or invalid result) |
| `eAutotuneState` | `E_AutotuneState` | Current autotune state (for HMI display) |

---

## Algorithm details

### PID

- **ISA parallel form**: `u = Kp · (e + (1/Ti) · ∫e dt + Td · de/dt)`
- **Derivative on measurement** (not on error) – avoids derivative kick on setpoint steps
- **Derivative low-pass filter** with time constant `rTf`
- **Anti-windup**: back-calculation – when output is clamped the integrator is corrected proportionally
- **Bumpless manual→auto transfer**: integrator is pre-loaded from the manual output value

### Digital outputs

```
bHeat = bHeatEnable AND (rPidOut >  rDeadband)
bCool = bCoolEnable AND (rPidOut < −rDeadband)
```

### Analog output scaling

The raw PID output `rPidOut ∈ [rOutMin … rOutMax]` is linearly mapped to the physical signal range `[rAnalogOutMin … rAnalogOutMax]`.

### Autotune (relay-feedback, Åström–Hägglund)

1. The autotune block drives the process with a relay of ±`rAutotuneRelay` around the setpoint.
2. After `nAutotuneCycles` complete oscillations, the ultimate gain **Ku** and ultimate period **Tu** are computed:

   ```
   Ku = 4 · d / (π · a)
   ```
   where `d` = relay amplitude, `a` = half of average peak-to-peak PV swing.

3. Ziegler–Nichols PID parameters are derived:

   | Parameter | Formula |
   |---|---|
   | Kp | 0.6 · Ku |
   | Ti | 0.5 · Tu |
   | Td | 0.125 · Tu |

4. Results are written back into `stParams` automatically. The integrator is reset to avoid a bump.

---

## Usage / HMI integration

### Instantiation

```pascal
VAR
    fbTempCtrl   : FB_TemperatureControl;
    stTempParams : DUT_TemperatureControlParams;
END_VAR
```

### Calling

```pascal
fbTempCtrl(
    EN            := TRUE,
    rProcessValue := rTemperaturePV,   (* analog input, scaled to °C *)
    tCycleTime    := T#10MS,
    stParams      := stTempParams
);

rHeaterAO := fbTempCtrl.rAnalogOut;
bHeaterDO := fbTempCtrl.bHeat;
bCoolerDO := fbTempCtrl.bCool;
```

### HMI controls (recommended)

| HMI control | PLC variable | Type |
|---|---|---|
| Setpoint numeric input | `stTempParams.rSetpoint` | RW |
| Mode toggle (Auto/Manual) | `stTempParams.bAutoMode` | RW |
| Manual output slider | `stTempParams.rManualOut` | RW |
| Kp / Ti / Td inputs | `stTempParams.rKp/rTi/rTd` | RW |
| Start autotune button | `stTempParams.bAutotuneRequest` | RW (momentary) |
| Autotune state display | `fbTempCtrl.eAutotuneState` | RO |
| PV display | `rTemperaturePV` | RO |
| PID output bar | `fbTempCtrl.rPidOut` | RO |
| Heat / Cool indicators | `fbTempCtrl.bHeat / .bCool` | RO |
| Alarm indicators | `fbTempCtrl.bAlarmHigh / .bAlarmLow` | RO |

---

## Structured text coding conventions

All function blocks follow the EN/ENO pattern required by the project rules:

```pascal
IF EN THEN
    (* ... block logic ... *)
ELSE
    (* ... reset outputs to safe/zero state ... *)
END_IF

(* External FB calls (TON, sub-FBs) always placed after the IF/ELSE *)
fbSubBlock(...);

ENO := EN;
```
