# PAxoo

## TwinCAT 3 - FB do kontroli temperatury

W repo dodano gotowy zestaw plikow:

- `TwinCAT/ST_TemperatureControlParams.TcDUT`
  - parametry regulatora jako DUT do podania przez `VAR_IN_OUT`
- `TwinCAT/FB_TemperatureControl.TcPOU`
  - FB z funkcja grzania i chlodzenia
  - regulacja PID
  - wyjscie analogowe signed `-100..100`
  - osobne wyjscia analogowe `grzanie/chlodzenie` oraz wyjscia cyfrowe
  - funkcja autotuningu z identyfikacja FOPDT:
    - wykrycie opoznienia `L`
    - estymacja stalej czasowej `T`
    - estymacja wzmocnienia procesu `Kproc`
    - wyliczenie `Rth` i `Cth` (przy zadanej mocy grzalki)
    - strojenie IMC-PID na podstawie modelu
- `TwinCAT/PRG_TemperatureControlDemo.TcPOU`
  - prosty program demonstracyjny podlaczenia pod HMI

## Szybkie uzycie

1. Dodaj do projektu TwinCAT pliki DUT i POU.
2. Utworz instancje:
   - `stTempParams : ST_TemperatureControlParams;`
   - `fbTempControl : FB_TemperatureControl;`
3. Wywoluj blok cyklicznie i podaj:
   - PV temperatury (`fTemperaturePv`)
   - impuls startu autotune z HMI (`bAutoTuneStart`)
   - `stParams := stTempParams` przez `VAR_IN_OUT`
4. Uzyj wyjsc:
   - analog signed: `fAnalogOutputPct`
   - analog grzanie/chlodzenie: `fHeatAnalogPct`, `fCoolAnalogPct`
   - cyfrowe: `bHeatOn`, `bCoolOn`

## Parametry autotune i diagnostyka modelu

W `ST_TemperatureControlParams`:
- wejscia autotune:
  - `fAutoTuneStep` - krok mocy grzania [%]
  - `tAutoTuneObserve` - czas obserwacji odpowiedzi skokowej
  - `fAutoTuneNoiseBand` - prog wykrycia startu odpowiedzi
  - `fAutoTuneLambdaFactor` - agresywnosc strojenia IMC
  - `fHeaterPowerNominalW` - moc grzalki przy 100% (do Cth)
- wyniki identyfikacji:
  - `fIdentProcessGainDegCPerPct`
  - `fIdentDeadTimeS`
  - `fIdentTimeConstantS`
  - `fIdentThermalResistanceKPerW`
  - `fIdentThermalCapacityJPerK`
