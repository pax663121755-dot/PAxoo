# PAxoo - FB_TempControl

Blok funkcyjny (Function Block) dla **TwinCAT 3** do regulacji temperatury z pełną obsługą grzania, chłodzenia, regulacji PID, autotuningu oraz interfejsu HMI.

## Struktura projektu

```
PLC/
├── DUTs/
│   ├── E_TempControlMode.TcDUT       Tryby pracy (Idle, Auto, Manual, AutoTune, Error)
│   ├── E_TempControlState.TcDUT      Stany bloku (Idle, Heating, Cooling, AtSetpoint, ...)
│   ├── ST_TempControl_Duty.TcDUT     Parametry DUTY (VAR_IN_OUT)
│   └── ST_TempControl_HMI.TcDUT      Interfejs HMI (VAR_IN_OUT)
└── POUs/
    └── FB_TempControl.TcPOU          Główny blok funkcyjny
```

## Funkcjonalności

| # | Funkcja | Opis |
|---|---------|------|
| 1 | **Parametry DUTY** | Struktura `ST_TempControl_Duty` jako `VAR_IN_OUT` — parametry PID, limity, histereza, filtr, konfiguracja autotuningu |
| 2 | **Grzanie i chłodzenie** | Wyjścia cyfrowe `bHeating` / `bCooling` z histerezą |
| 3 | **Regulacja PID** | Proporcjonalno-całkująco-różniczkujący regulator z anti-windup |
| 4 | **Wyjście analogowe** | `rAnalogOutput` (0–100%) do sterowania zaworami / grzałkami |
| 5 | **Wyjścia cyfrowe** | `bHeating` – grzej, `bCooling` – chłodź (z progami i histerezą) |
| 6 | **Autotuning** | Metoda relay-feedback (Åström-Hägglund) → parametry Ziegler-Nichols |
| 7 | **Interfejs HMI** | Struktura `ST_TempControl_HMI` — komendy + status, gotowa do podpięcia |

## Parametry DUTY (`ST_TempControl_Duty`)

| Parametr | Typ | Domyślna | Opis |
|----------|-----|----------|------|
| `rKp` | REAL | 10.0 | Wzmocnienie proporcjonalne |
| `rKi` | REAL | 0.5 | Wzmocnienie całkujące |
| `rKd` | REAL | 1.0 | Wzmocnienie różniczkujące |
| `rSetpoint` | REAL | 25.0 | Temperatura zadana [°C] |
| `rTempMin` / `rTempMax` | REAL | -20 / 200 | Zakres czujnika [°C] |
| `rSetpointMin` / `rSetpointMax` | REAL | 0 / 150 | Limity setpointu [°C] |
| `rOutputMin` / `rOutputMax` | REAL | 0 / 100 | Limity wyjścia PID [%] |
| `rHysteresis` | REAL | 1.0 | Histereza wyjść cyfrowych [°C] |
| `rDeadband` | REAL | 0.5 | Martwa strefa wokół setpointu [°C] |
| `rHeatingThreshold` | REAL | 5.0 | Próg grzania w trybie manual [%] |
| `rCoolingThreshold` | REAL | -5.0 | Próg chłodzenia w trybie manual [%] |
| `rManualOutput` | REAL | 0.0 | Wartość ręczna wyjścia [%] |
| `rFilterTime` | REAL | 0.5 | Czas filtru PT1 [s] |
| `tCycleTime` | TIME | T#100MS | Czas cyklu PLC |
| `rAutoTuneStep` | REAL | 30.0 | Krok relay auto-tune [%] |
| `rAutoTuneHyst` | REAL | 2.0 | Histereza relay auto-tune [°C] |
| `nAutoTuneOscCount` | INT | 4 | Liczba oscylacji do obliczeń |

## Interfejs HMI (`ST_TempControl_HMI`)

### Komendy (HMI → PLC)

| Pole | Typ | Opis |
|------|-----|------|
| `bStart` | BOOL | Start regulacji |
| `bStop` | BOOL | Stop regulacji |
| `bReset` | BOOL | Kasowanie błędów |
| `bStartAutoTune` | BOOL | Start autotuningu |
| `bStopAutoTune` | BOOL | Przerwanie autotuningu |
| `eRequestedMode` | E_TempControlMode | Żądany tryb pracy |
| `rSetpoint` | REAL | Setpoint z HMI [°C] |
| `rManualOutput` | REAL | Wyjście ręczne z HMI [%] |

### Status (PLC → HMI)

| Pole | Typ | Opis |
|------|-----|------|
| `rActualTemp` | REAL | Aktualna temperatura [°C] |
| `rPidOutput` | REAL | Wyjście PID [%] |
| `rAnalogOutput` | REAL | Wyjście analogowe [%] |
| `eActiveMode` | E_TempControlMode | Aktywny tryb |
| `eState` | E_TempControlState | Stan bloku |
| `bHeating` / `bCooling` | BOOL | Wyjścia cyfrowe |
| `bAtSetpoint` | BOOL | Temperatura osiągnięta |
| `bError` | BOOL | Flaga błędu |
| `nErrorId` | DINT | Kod błędu |
| `sErrorMsg` | STRING | Opis błędu |
| `rTunedKp/Ki/Kd` | REAL | Nastrojone parametry PID |

## Struktura bloku (wzorzec EN/ENO)

```
IF ENO THEN
    <logika bloku>
ELSE
    <reset do stanu zerowego>
END_IF;
<wywołania zewnętrznych FB (TON)>
ENO := EN;
```

## Tryby pracy

- **Idle** — blok nieaktywny, wyjścia wyzerowane
- **Auto** — regulacja PID z automatycznym sterowaniem grzania/chłodzenia
- **Manual** — bezpośrednie sterowanie wyjściem analogowym
- **AutoTune** — automatyczne strojenie PID metodą relay-feedback
- **Error** — błąd (wymaga reset z HMI)

## Algorytm autotuningu

1. Blok przełącza relay ON/OFF wokół setpointu z histerezą `rAutoTuneHyst`
2. Mierzy amplitudę i okres oscylacji
3. Po zebraniu `nAutoTuneOscCount` oscylacji oblicza:
   - Ku (ultimate gain) = 4d / (π·a) gdzie d = krok relay, a = amplituda
   - Ziegler-Nichols: Kp = 0.6·Ku, Ki = 1.2·Ku/Tu, Kd = 0.075·Ku·Tu
4. Zapisuje wyniki do `stDuty` i przechodzi do trybu Auto

## Przykład użycia

```iecst
PROGRAM MAIN
VAR
    fbTemp      : FB_TempControl;
    stDuty      : ST_TempControl_Duty;
    stHMI       : ST_TempControl_HMI;
    rTempSensor : REAL;     (* z wejścia analogowego *)
END_VAR

(* Konfiguracja *)
stHMI.rSetpoint := 80.0;
stHMI.eRequestedMode := E_TempControlMode.eAuto;
stHMI.bStart := TRUE;

(* Wywołanie bloku *)
fbTemp(
    EN              := TRUE,
    rActualTemp_Raw := rTempSensor,
    stDuty          := stDuty,
    stHMI           := stHMI
);

(* Użycie wyjść *)
(* fbTemp.bHeating    → wyjście cyfrowe grzanie *)
(* fbTemp.bCooling    → wyjście cyfrowe chłodzenie *)
(* fbTemp.rAnalogOutput → wyjście analogowe 0-100% *)
```
