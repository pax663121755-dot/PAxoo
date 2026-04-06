# PAxoo

## TwinCAT 3 - Temperature controller FB

Repo zawiera gotowe pliki eksportu TwinCAT 3 dla prostego bloku kontroli temperatury:

- `TwinCAT3/TemperatureControl/DUT/ST_TemperatureControllerParams.TcDUT`
- `TwinCAT3/TemperatureControl/DUT/ST_TemperatureControllerHmi.TcDUT`
- `TwinCAT3/TemperatureControl/POUs/FB_TemperaturePid.TcPOU`
- `TwinCAT3/TemperatureControl/POUs/FB_TemperatureAutoTune.TcPOU`
- `TwinCAT3/TemperatureControl/POUs/FB_TemperatureController.TcPOU`

### Funkcje bloku

- parametry kontrolera przekazywane jako DUT przez `VAR_IN_OUT`
- grzanie i chlodzenie w jednym FB
- regulator PID
- wyjscie analogowe `fAnalogOutput`
- wyjscia cyfrowe `boHeat` i `boCool`
- prosty autotuning typu relay test
- struktura HMI do prostego sterowania i podgladu statusu

### Przykladowe uzycie

```iecst
VAR
    fbTemperature : FB_TemperatureController;
    stTemperatureParams : ST_TemperatureControllerParams := (
        fKp := 2.0,
        fTi_s := 120.0,
        fTd_s := 5.0,
        fCycleTime_s := 0.1,
        fOutputMin := -100.0,
        fOutputMax := 100.0,
        fHeatOutputMax := 100.0,
        fCoolOutputMax := 100.0
    );
    stTemperatureHmi : ST_TemperatureControllerHmi;
END_VAR

fbTemperature(
    EN := TRUE,
    stParams := stTemperatureParams,
    stHmi := stTemperatureHmi
);
```

### Logika wyjsc

- `fAnalogOutput > 0` - sterowanie grzaniem
- `fAnalogOutput < 0` - sterowanie chlodzeniem
- `fHeatingPercent` - analog dla toru grzania
- `fCoolingPercent` - analog dla toru chlodzenia
- `boHeat` i `boCool` - wyjscia cyfrowe

### Autotuning

Autotuning uruchamia sie przez ustawienie `stHmi.boStartAutoTune := TRUE` przy aktywnym `boEnable` i `boAutoMode`.
Po zakonczeniu blok zapisuje nowe nastawy PID bezposrednio do `stParams.fKp`, `stParams.fTi_s`, `stParams.fTd_s`.
