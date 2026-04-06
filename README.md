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
- `TwinCAT/ST_HeatingPlantSimParams.TcDUT`
  - parametry modelu symulacji obiektu grzania/chlodzenia
- `TwinCAT/FB_HeatingPlantSimulator.TcPOU`
  - symulator obiektu: wolny start narastania, szybszy przyrost po rozgrzaniu grzalki
  - po odcieciu mocy chlodzenie przyspiesza wraz z czasem bez zasilania
- `TwinCAT/ST_HeatingPlantPhysicalSimParams.TcDUT`
  - parametry fizycznego modelu cieplnego R-C-C (grzalka + obiekt)
- `TwinCAT/FB_HeatingPlantPhysicalSimulator.TcPOU`
  - fizyczny symulator cieplny (moc grzalki, pojemnosci cieplne, rezystancje cieplne)
  - dodatkowe wejscie zaklocenia procesu (np. otwarcie drzwi, naplyw zimnego medium)

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

## Symulator obiektu grzania

W `PRG_TemperatureControlDemo` dodano:
- `stSimParams : ST_HeatingPlantSimParams`
- `fbPlantSim : FB_HeatingPlantSimulator`
- `stPhysicalSimParams : ST_HeatingPlantPhysicalSimParams`
- `fbPhysicalPlantSim : FB_HeatingPlantPhysicalSimulator`

Symulator otrzymuje sterowanie:
- `fHeatCmdPct` (grzanie 0..100%)
- `fCoolCmdPct` (chlodzenie 0..100%)
- `fAmbientTemp` (temperatura otoczenia)

oraz zwraca temperature procesu (`fTemperaturePv`), ktora mozna podac bezposrednio na wejscie regulatora.

### Wariant fizyczny R-C-C

`FB_HeatingPlantPhysicalSimulator` modeluje:
- pojemnosc cieplna grzalki `fHeaterCapacityJPerK`
- pojemnosc cieplna obiektu `fProcessCapacityJPerK`
- transfer ciepla grzalka->obiekt przez `fRthHeaterToProcessKPerW`
- straty obiektu i grzalki do otoczenia przez rezystancje cieplne
- aktywne chlodzenie i zaklocenie procesu:
  - `fCoolCmdPct` (chlodzenie sterowane)
  - `fDisturbancePct` (zaklocenie, np. drzwi komory / zimny wsad)

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
