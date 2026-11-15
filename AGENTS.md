# AGENTS.md

Richtlijnen en projectoverzicht voor (AI-)ontwikkelaars die aan deze repository werken. Dit is het enige centrale document; `CLAUDE.md` verwijst hiernaar.

> Bijgewerkt na de merge met upstream `heishamon/HeishaMon` t/m release **v4.4.0** (november 2026).

**Lees eerst** de `heishamon-rules` skill in `Examples/Rules/SKILL.md` voordat je een rules-engine-bestand (`rules.txt`, `src/rules/`) schrijft of wijzigt.

Inhoud:

1. [Wat is HeishaMon?](#1-wat-is-heishamon)
2. [Build en tests](#2-build-en-tests)
3. [Architectuur](#3-architectuur)
4. [Bestandsstructuur](#4-bestandsstructuur)
5. [Het topic-systeem](#5-het-topic-systeem)
6. [Commando's](#6-commandos-schrijven-naar-de-warmtepomp)
7. [Overige subsystemen](#7-overige-subsystemen)
8. [Nieuwe topics vinden en toevoegen](#8-nieuwe-topics-vinden-en-toevoegen)
9. [Versies, releases en referentiedocumentatie](#9-versies-releases-en-referentiedocumentatie)

---

## 1. Wat is HeishaMon?

HeishaMon is Arduino/C++-firmware voor ESP8266- (small PCB) en ESP32-S3-boards (large PCB) die het eigen seriële protocol (CN-CNT, 9600 baud, 8E1) van **Panasonic Aquarea** lucht-water warmtepompen (H-, J-, K-, L- en M-serie) spreekt. De firmware leest 203-byte datapakketten, decodeert die naar 199 sensorwaarden en publiceert ze als **MQTT**-topics, via een JSON-HTTP-endpoint, de websocket van de web-UI en (alleen ESP32) **Modbus TCP**. Daarnaast kan HeishaMon:

- commando's terugsturen naar de warmtepomp (via MQTT, web-UI, Modbus of rules);
- een kleine lokale **rules-engine** draaien om de warmtepomp zonder externe domotica te automatiseren;
- (alleen ESP32) de originele Panasonic CZ-TAW1-controller proxyen/doorgeven.

De hele firmware staat in één Arduino-sketchmap: `HeishaMon/`. Er is geen aparte backend/frontend: de web-UI bestaat uit PROGMEM-HTML-strings die in de firmware zijn ingebed.

```
┌─────────────────┐    Serieel 9600 8E1    ┌──────────────┐  MQTT/HTTP/Modbus  ┌──────────────────┐
│  Panasonic       │◄─────────────────────►│  HeishaMon    │◄──────────────────►│  Domotica        │
│  Aquarea         │   203-byte pakketten  │  ESP8266/     │   199 topics       │  (Home Assistant, │
│  Warmtepomp      │   110-byte commando's │  ESP32-S3     │   69 commando's    │   Domoticz, etc.) │
└─────────────────┘                        └──────────────┘                    └──────────────────┘
                                                  │
                                           ┌──────┴──────┐
                                           │  Web UI     │
                                           │  (poort 80) │
                                           └─────────────┘
```

### Communicatieprotocol

1. HeishaMon stuurt elke ~2 seconden een **query** (110 bytes, header `0x71 0x6C 0x01 0x10`) naar de warmtepomp
2. De warmtepomp antwoordt met een **datapakket** van 203 bytes (header `0x71 0xC8 0x01 0x10`)
3. Nieuwere modellen (K/L/M) ondersteunen ook een **extra datablok** (header `0x71 0xC8 0x01 0x21`) met vermogensgegevens
4. Elk pakket eindigt met een **checksum**: `(XOR 0xFF) + 1` van alle bytes, zodat de som 0 is
5. Commando's gebruiken een 110-byte template (header `0xF1 0x6C 0x01 0x10`); alle bytes die niet gezet worden blijven 0 (= "geen wijziging")
6. Achtergrond over pariteit en seriële captures: `AquareaParity.md` en `Tools/capture_tools.py`

### Hardwareplatforms

| Platform | Chip | Kenmerken |
|----------|------|-----------|
| **Small** | ESP8266 (Wemos D1 mini) | WiFi, ~720KB firmware |
| **Large** | ESP32-S3 | WiFi + Ethernet (W5500), PSRAM, NeoPixel LED, proxy-poort (CZ-TAW1), Modbus TCP, ~1.5MB firmware |

Seriële pinnen:

- **ESP8266**: GPIO13 (RX), GPIO15 (TX) via `Serial.swap()`
- **ESP32**: GPIO18 (RX), GPIO17 (TX), plus proxy-poort op GPIO9 (RX) / GPIO8 (TX)

---

## 2. Build en tests

Dit is een Arduino-sketch, gebouwd met `arduino-cli` (geen `make`/`cmake`). De helper-scripts bevatten exact de `arduino-cli compile`-aanroepen die CI gebruikt:

```bash
scripts/build_esp8266.sh   # small PCB (Wemos D1 mini / ESP8266): esp8266:esp8266:d1_mini, 160MHz, 4MB flash, 2MB SPIFFS
scripts/build_esp32s3.sh   # large PCB (ESP32-S3): esp32:esp32@3.3.11, CDC boot, PSRAM, min SPIFFS
```

Beide scripts doen eerst `cd HeishaMon` en zetten de binary/map naast de sketch. Alternatief: PlatformIO via `platformio.ini`.

**Bibliotheken** (via `arduino-cli lib install`, zie `LIBSUSED.md` en `.github/workflows/main.yml`): `ringbuffer`, `pubsubclient`, `arduinojson`, `dallastemperature`, `onewire`, `Adafruit NeoPixel`, `Async TCP` (die van ESP32Async; niet `AsyncTCP`, dat naar een verouderde fork verwijst). eModbus apart installeren, want het staat niet in de Arduino-registry:

```bash
ARDUINO_LIBRARY_ENABLE_UNSAFE_INSTALL=true arduino-cli lib install --git-url https://github.com/eModbus/eModbus.git#v1.7.5stable
```

Benodigde board-cores: `esp8266:esp8266` en `esp32:esp32@3.3.11`. Er is een devcontainer (`.devcontainer/`, image `ghcr.io/the78mole/heishamon-dev`) met `arduino-cli` en alle afhankelijkheden voorgeïnstalleerd; gebruik die bij voorkeur.

### Tests (host, alleen `g++`/`python3`, geen Arduino-toolchain)

| Commando | Wat het test |
|----------|--------------|
| `python3 Tools/test_decode.py` | Compileert de echte decoder met een host-shim en test de MQTT-output |
| `python3 Tools/test_commands.py` | Elk hoofdcommando moet exact de vastgelegde datagrammen blijven opleveren |
| `python3 tests/modbus/run_tests.py` | Modbus registermap en request-handler |
| `Examples/Rules/run_tests.sh` | Rules-engine + alle voorbeeld-rulesets (zie §7) |
| `node Tools/chksumChecker.js` | Losse tool om checksums van commando-pakketten te berekenen/controleren |

Verder is verificatie: compileert het voor beide boards, en (waar mogelijk) handmatig testen met echte hardware of gelogde seriële captures. Draai `Examples/Rules/run_tests.sh` na elke wijziging in `src/rules/`, `src/common/timerqueue.cpp`, de decode-/commandotabellen of een voorbeeld-ruleset.

### CI

- `.github/workflows/main.yml`: bouwt beide boards en draait de decode-, command- en Modbus-tests bij tags, PR's en nightly; nooit een gewone build bij een kale push naar `main`.
- `.github/workflows/rules-tests.yml`: draait de rules-tests bij wijzigingen in rules/common/decode.h/commands.h.

### Pre-commit / formatting

`.pre-commit-config.yaml` draait: trailing-whitespace, end-of-file-fixer, check-yaml, BOM-fix, mixed-line-ending-fix, `black`+`isort` (Python) en `clang-format` op `*.c`/`*.cpp`/`*.h`/`*.hpp`. Draai `pre-commit run --all-files` vóór het committen van C/C++-wijzigingen.

---

## 3. Architectuur

### Twee hardwaredoelen, één codebase

`HeishaMon.ino` en de andere `.cpp`/`.h`-bestanden vertakken veel op `#ifdef ESP8266` / `#ifdef ESP32`:

- **ESP8266**: warmtepomp-serieel is de primaire `Serial`; single-threaded `loop()`; geen proxy-poort; TLS is opt-in.
- **ESP32-S3**: warmtepomp-serieel is `Serial1`; een aparte FreeRTOS-taak (`serialTXTask`, gestart via `xTaskCreatePinnedToCore` in `setupConditionals()`) doet alle uitgaande seriële writes en communiceert met de main loop via `pcbQueue`/`cmdQueue`/`logQueue`; tweede UART (`proxySerial`) om verkeer van/naar de CZ-TAW1 door te geven/te inspecteren (`readProxy()`); NeoPixel-status-LED en optioneel W5500-Ethernet (`setupETH()`); TLS altijd aan.

Controleer bij features die seriële I/O of instellingen raken altijd dat beide varianten compileren.

### Datastroom

1. **`readSerial()` / `readProxy()`** (`HeishaMon.ino`) lezen ruwe bytes, valideren header/lengte/checksum (`calcChecksum`, `isValidReceiveChecksum`) en stellen een van drie datagrammen samen: volledig datablok (`DATASIZE`=203, headerbyte `0x10`), extra datablok (headerbyte `0x21`, alleen K/L/M, afhankelijk van `extraDataBlockAvailable`), of de optional-PCB-ack (`OPTDATASIZE`=20).
2. **`decode.cpp`/`decode.h`** zetten een datagram om in losse waarden en publiceren elk op een eigen MQTT-subtopic (`decode_heatpump_data`, `decode_heatpump_data_extra`, `decode_optional_heatpump_data`). Alles is tabelgestuurd (zie §5): een nieuw protocolveld toevoegen is vooral een kwestie van tabellen uitbreiden.
3. **`commands.cpp`/`commands.h`** gaan de andere kant op: elk instelbaar warmtepomp-parameter is één rij in `commands[]` (zie §6). `send_command()` (of op ESP32 de `cmdQueue`) verstuurt; de checksum wordt automatisch berekend en te lange commando's worden geweigerd.
4. **MQTT, web/REST en Modbus** zijn front-ends op dezelfde tabellen: `mqtt_callback()` in `HeishaMon.ino` dispatcht inkomende berichten op topic-prefix (`commands/`, `gpio/`, `opentherm/`, `s0/...`, raw-send) naar dezelfde handlers die de web-route `/command` gebruikt (`webserver_cb()` route 100).

### Webserver

Geen framework: `webfunctions.cpp` implementeert een kleine HTTP-server bovenop `src/common/webserver.*` via één callback `webserver_cb()` (gedeclareerd in `HeishaMon.ino`) die schakelt op `client->route` (integer per URL, toegekend in de case `WEBSERVER_CLIENT_REQUEST_URI`) en `client->step` (methode / URI / args / header / write). Alle HTML staat als statische PROGMEM-strings in `htmlcode.h`; dynamische waarden worden met `sprintf`-achtige formattering in de response gezet. Instellingen gaan via `settingsStruct` (`webfunctions.h`) als `config.json` naar LittleFS met ArduinoJson (`settingsToJson`/`loadSettings`). Het update-wachtwoord wordt niet meer via `/getsettings` meegestuurd.

Web-UI-functies:

- **Dashboard**: live sensorwaarden, optionele eigen omschrijving naast het logo
- **Console**: logberichten + hexdump-toggle
- **Instellingen**: WiFi (incl. roaming), MQTT, serieel, topic-groepen, timers, optional PCB, OpenTherm, Modbus
- **Firmware** (`/firmware`): OTA-update; toont de nieuwste release en kan die direct vanuit de browser installeren
- **Rules**: lokale automatiseringsregels

---

## 4. Bestandsstructuur

### Hoofdmap (`/`)

| Bestand/map | Doel |
|-------------|------|
| `AGENTS.md` | Dit document |
| `CLAUDE.md` | Verwijzing naar dit document |
| `README.md` | Gebruikersdocumentatie (ook DE/FI/NL), incl. volledige rules-syntax |
| `MQTT-Topics.md` / `Manage-Topics.md` | Alle gepubliceerde en instelbare MQTT-topics |
| `ProtocolByteDecrypt.md` / `ProtocolByteDecrypt-extra.md` | Byte-layout van hoofd- en extra datablok |
| `Modbus-Register-Mapping.md` | Modbus TCP registermap (v3) |
| `AquareaParity.md` | Pariteit / seriële captures |
| `HeatPumpType.md` | Bevestigd ondersteunde warmtepompmodellen |
| `Examples/Rules/` | Voorbeeld-rulesets, `SKILL.md`, host-testharnas, `run_tests.sh` |
| `tests/modbus/` | Host-tests Modbus |
| `Tools/` | `chksumChecker.js`, `capture_tools.py`, `test_commands.py`, `test_decode.py` |
| `Integrations/` | Home Assistant, Domoticz, OpenHAB, Node-RED, ioBroker (submodules: `Integrations/Domoticz/HeishamonMQTT`, `Integrations/NodeRed`) |
| `binaries/` | Voorgecompileerde firmware (`model-type-small` / `model-type-large`) |
| `scripts/` | Build-scripts |
| `platformio.ini` | Alternatieve PlatformIO-config |

### Broncode (`/HeishaMon/`)

| Bestand | Regels | Functie |
|---------|--------|---------|
| **HeishaMon.ino** | ~2440 | Hoofdprogramma: WiFi/MQTT, seriële communicatie, main loop, `mqtt_callback()` |
| **decode.h** | ~1055 | Topic-namen, topic-groepen, byte-posities, conversiefuncties, beschrijvingen |
| **decode.cpp** | ~500 | Decodering bytes → waarden, publicatie |
| **commands.h** | ~220 | `commands[]` (69 rijen), `optionalCommands[]` (14), macro's |
| **commands.cpp** | ~430 | `heatpump_command()` + commando's met eigen functie |
| **webfunctions.h/cpp** | ~1870 | HTTP-server, REST API, instellingen |
| **htmlcode.h** | ~3700 | Ingebedde HTML/CSS/JS |
| **rules.h/cpp** | ~1150 | Koppeling rules-engine aan de sketch |
| **HeishaModbusServer.h/cpp**, **ModbusRegisterMap.h** | ~1170 | Modbus TCP (ESP32) |
| **dallas.h/cpp** | | DS18B20 1-wire |
| **s0.h/cpp**, **s0data.h/cpp** | | S0 kWh-meters + gedeelde S0-data |
| **gpio.h/cpp** | | GPIO, seriële init, LED |
| **HeishaOT.h/cpp** | | OpenTherm |
| **version.h** | | `HEISHAMON_VERSION` |
| **src/common/** | | Webserver, logging, SHA1, Base64, timerqueue, geheugenpool (`mem.cpp`, `MEMPOOL_SIZE`), string-utils |
| **src/opentherm/** | | OpenTherm-protocol |
| **src/rules/** | | Rules-parser/interpreter, functies, operators, stack |

### Belangrijke constanten

| Constante | Waarde | Betekenis |
|-----------|--------|-----------|
| `DATASIZE` | 203 | Datapakket van warmtepomp |
| `OPTDATASIZE` | 20 | Optioneel PCB-pakket |
| `PANASONICQUERYSIZE` | 110 | Query/commando naar warmtepomp |
| `NUMBER_OF_TOPICS` | 199 | Hoofdtopics (TOP0–TOP198) |
| `NUMBER_OF_TOPICS_EXTRA` | 6 | Extra topics (XTOP0–XTOP5) |
| `NUMBER_OF_OPT_TOPICS` | 7 | Optional-PCB-topics (OPT0–OPT6) |
| `NUMBER_OF_TOPIC_GROUPS` | 9 | Topic-groepen |
| `MAX_TOPIC_LEN` | 42 | Maximale lengte topic-naam + 1 |
| `SERIALTIMEOUT` | 2000 | Seriële timeout (ms) |

---

## 5. Het topic-systeem

### Drie categorieën read-topics

| Categorie | Aantal | Array | MQTT-prefix | Bron |
|-----------|--------|-------|-------------|------|
| **Hoofdtopics** | 199 (TOP0–TOP198) | `topics[]` | `main/` | 203-byte hoofdpakket |
| **Extra topics** | 6 (XTOP0–XTOP5) | `xtopics[]` | `extra/` | 203-byte extra datablok |
| **Optioneel PCB** | 7 (OPT0–OPT6) | `optTopics[]` | `optional/` | 20-byte optional-PCB-pakket |

Laatst toegevoegd: `TOP197 Heat_Pump_Capacity` (byte 191, kW) en `TOP198 Heat_Pump_Type` (byte 192, Standard/T-CAP).

### Definitie van een hoofdtopic (in `decode.h`)

Elk hoofdtopic bestaat uit **vijf parallelle arrays** op dezelfde index:

```
Index i ──► topics[i]           = "Heatpump_State"   (MQTT topic-naam)
        ──► topicGroup[i]       = TG_BASIC            (topic-groep)
        ──► topicBytes[i]       = 4                   (byte-positie in het 203-byte pakket)
        ──► topicFunctions[i]   = getBit7and8         (conversiefunctie: byte → waarde)
        ──► topicDescription[i] = OffOn               (beschrijving: {"2","Off","On"})
```

Een `static_assert` controleert dat `topicGroup[]` precies `NUMBER_OF_TOPICS` entries heeft.

### Topic-groepen

Elk hoofdtopic hoort bij één groep (`topicGroups_t`): `TG_BASIC`, `TG_HEATCOOL`, `TG_ZONE2`, `TG_DHW_HEATERS`, `TG_DIAGNOSTICS`, `TG_BUFFER_SOLAR_POOL`, `TG_BIVALENT`, `TG_EXTERNAL_SGREADY`, `TG_INSTALLER`. Alle groepen behalve `TG_BASIC` zijn in de instellingen uit te schakelen (settings-keys in `topicGroupKeys[]`, bijv. `topics_zone2`); standaard staat alleen `TG_INSTALLER` uit. Een uitgeschakeld topic wordt nog wél gedecodeerd (rules, Modbus en `/json?topic=` blijven werken), maar niet naar MQTT, de websocket of de volledige `/json`-lijst gepubliceerd.

### Decodering stap voor stap

1. **Ontvangst**: `readSerial()` leest 203 bytes, valideert header en checksum
2. **Routing**: byte 3 bepaalt het type (`0x10` → normaal, `0x21` → extra)
3. **Iteratie**: `decode_heatpump_data()` loopt door alle `NUMBER_OF_TOPICS` topics
4. **Extractie**: byte `topicBytes[i]` → `topicFunctions[i]()`; sommige topics (pump flow, errors, model, 16-bit waarden, power) hebben eigen logica in `getDataValue()`
5. **Vergelijking** met de vorige waarde (`actData`)
6. **Publicatie** bij wijziging (en als de groep aan staat) naar `{base}/main/{topic_naam}`

### Conversiefuncties (in `decode.cpp`)

| Functie | Werking |
|---------|---------|
| `getBit1` | Bit 7 (0/1) |
| `getBit1and2` / `getBit3and4` / `getBit5and6` / `getBit7and8` | 2-bit groep (bits 7-6 / 5-4 / 3-2 / 1-0), minus 1 |
| `getBit3and4and5` | 3-bit groep |
| `getLeft5bits` / `getRight3bits` | Bovenste 5 / onderste 3 bits |
| `getFirstByte` / `getSecondByte` | Hoge / lage nibble-varianten |
| `getHighNibbleOffset` / `getLowNibbleOffset` | Nibble als sensor-offset (K) |
| `getHighNibbleMinus1Times5` / `getLowNibbleMinus1Times5` | (Nibble − 1) × 5 |
| `getIntMinus128` | Waarde − 128 (°C) |
| `getIntMinus1` | Waarde − 1 |
| `getIntMinus1Div5` / `Div10` / `Div50` | (Waarde − 1) / n |
| `getIntMinus1Times10` / `Times30` / `Times50` | (Waarde − 1) × n |
| `getRawByte` | Ruwe bytewaarde |
| `getValvePID` | (Waarde − 1) / 2 |
| `getHeaterCapacity` | Capaciteit elektrisch element |
| `getOpMode` | Bit-mapping naar modusnummer (18 → "Heat") |
| `getZone1Active` / `getZone2Active` | Zone-status |
| `getUintt16` | 16-bit unsigned little-endian (XTOP's) |
| `getPumpFlow`, `getErrorInfo`, `getModel`, `getPower` | Eigen berekeningen |
| `unknown` | Niet geïmplementeerd |

### Beschrijvingsarrays (in `decode.h`)

Formaat `{"<aantal>", "label0", "label1", ...}`; `"0"` betekent een eenheid in plaats van een opsomming. Voorbeelden: `OffOn`, `DisabledEnabled`, `InactiveActive`, `OpModeDesc`, `Celsius`, `Kelvin`, `Watt`, `Kilowatt`, `Hertz`, `LitersPerMin`, `RotationsPerMin`, `Pressure`, `Bar`, `Hours`, `Minutes`, `Counter`, `Ampere`, `Percent`, `Steps`, `HeatPumpType`, `BivalentOperatingMode`, `ExtraPumpUse`.

---

## 6. Commando's (schrijven naar de warmtepomp)

### Tabelgestuurde structuur (in `commands.h`)

Bijna elk commando is één rij in `commands[]` die beschrijft welk veld in het 110-byte datagram wordt geschreven:

```c
struct cmdStruct {
  char name[29];      // MQTT-commandonaam, bijv. "SetDHWTemp"
  uint8_t flags;      // bijv. CMD_INSTALLER (alleen na setInstallerCommandsAllowed(true))
  uint8_t type;       // CMD_FUNC, CMD_NUMBER, CMD_CHOICE, CMD_STRICT, CMD_TENTHS
  uint8_t address;    // byte in het datagram
  uint8_t shift, mask, offset, divisor, step;
  int16_t minimum, maximum;
  unsigned int (*func)(char *msg, unsigned char *cmd, char *log_msg);  // alleen voor CMD_FUNC
};
// veld = ((waarde / divisor + offset) & mask) << shift, geschreven naar byte 'address'
```

| Type | Gedrag |
|------|--------|
| `CMD_NUMBER` | Elk getal (voorloopcijfers van de payload), geen bereikcontrole |
| `CMD_CHOICE` | Als `CMD_NUMBER`, maar buiten 0..maximum telt als 0 |
| `CMD_STRICT` | Hele payload moet geheel getal minimum..maximum in stappen van step zijn, anders niets verzonden |
| `CMD_TENTHS` | Als `CMD_STRICT`, maar met één decimaal; waarden in tienden |
| `CMD_FUNC` | Eigen functie |

Rijen schrijf je met macro's: `CMD_ONOFF(address, shift)`, `CMD_PAIR(address, shift, maximum)`, `CMD_TRIGGER`, `CMD_VALUE(address, offset)`, `CMD_FIELD(address, shift, mask, offset)`, `CMD_CHECKED(...)`, `CMD_DECIMAL(...)`, `CMD_FUNCTION(func)`. Alleen `SetOperationMode`, `SetCurves` en `SetZones` hebben nog een eigen functie. Er zijn **69 hoofdcommando's** en **14 optional-PCB-commando's** (`optionalCommands[]`, elk met een eigen `set_*`-functie).

### Hoe een commando werkt

1. `heatpump_command()` zet de payload om in het veld, of weigert hem (retourneert 0)
2. `send_command()` / `cmdQueue` berekent de checksum en verstuurt
3. De logregel toont de waarde die echt wordt verzonden

**Voorbeeld**: `SetDHWTemp` 50 → byte 42 wordt `50 + 128 = 178 (0xB2)`.

### Een commando toevoegen

1. Voeg een rij toe aan `commands[]` met de passende macro
2. Geef het een nieuw, **nooit hergebruikt** ID in `MAIN_COMMANDS` in `ModbusRegisterMap.h` (een `static_assert` bewaakt dat beide tabellen even lang zijn)
3. Voeg verwachte datagrammen toe in `Tools/test_commands.py` en draai die test
4. Documenteer in `MQTT-Topics.md` (en `Modbus-Register-Mapping.md`)

---

## 7. Overige subsystemen

### Rules-engine (`src/rules/`, plus `rules.cpp`/`rules.h` in de sketchmap)

Een kleine scripttaal (syntax in `README.md` onder "Rules functionality" en in `Examples/Rules/SKILL.md`) waarmee het apparaat reageert op warmtepomp-/thermostaatwaarden en timers. `src/rules/rules.cpp` is de parser/interpreter; `function.cpp` en `operator.cpp` dispatchen ingebouwde functies en operatoren; `src/rules/functions/*.cpp` bevatten de losse functies (`coalesce`, `min`, `max`, `isset`, `round`, `floor`, `ceil`, `print`, `concat`, `gpio`, `settimer`) — nieuwe functie = nieuw bestandspaar + registratie in `function.cpp`. Rulesets worden gevalideerd voordat ze worden toegepast (ongeldig → vorige blijft actief); een ruleset die het apparaat laat crashen wordt bij de volgende boot automatisch uitgeschakeld. De `rules.cpp`/`rules.h` in de sketchmap laden/bewaren `rules.txt` op LittleFS en koppelen de variabelen: `#` globals, `$` locals, `@` warmtepomp-parameters, `%` datum/tijd, `?` thermostaat, `ds18b20#...`, `s0#...`.

**Off-device testen**: `Examples/Rules/harness/` compileert de engine voor de host (`./build.sh`, alleen `g++`/`python3`) en voert dezelfde `rule_initialize()`-validatie uit als bij upload, inclusief `@Name`-controles tegen de echte `decode.h`/`commands.h`. Scenario's simuleren topicwijzigingen, timers (getrouw aan `timerqueue.cpp`: `setTimer(id, 0)` annuleert en vuurt nooit) en `%hour`/`%minute`-schema's, met `expectcmd`/`expectnone`-asserties. Elke voorbeeld-ruleset heeft regressiescenario's in `tests/`; `Examples/Rules/run_tests.sh` draait ze allemaal. Voeg scenario's toe bij nieuw rule-gedrag — zie `Examples/Rules/harness/README.md`.

### Modbus TCP (`HeishaModbusServer.*`, `ModbusRegisterMap.h`, alleen ESP32)

- Opt-in via *Enable Modbus TCP* (poort 502, unit ID 1) en *Allow Modbus writes*; beide standaard uit, effect zonder reboot.
- eModbus-callbacks draaien in de AsyncTCP-taak: ze lezen alleen een data-snapshot en zetten writes in een queue; `loop()` ververst de snapshot en voert de writes uit. Een write wordt bevestigd zodra hij in de queue staat.
- Registermap v3: int16-metingen 0–4999 (main 0, extra 1000, optional 2000, S0 3000), int16-commando's 5000–9999 (heat pump 5000, optional 6000, systeem 7000), float32-metingen vanaf 10000 en -commando's vanaf 20000, coils (relais) 30000, device-info 32000.
- Topic-nummers en commando-ID's zijn **permanent**: alleen toevoegen, nooit hergebruiken. Details: `Modbus-Register-Mapping.md`.

### Overig

- **`dallas.cpp/.h`**: DS18B20-sensoren op GPIO4, aliassen, periodieke MQTT-resend.
- **`s0.cpp/.h`, `s0data.cpp/.h`**: S0-pulstelling op 2 poorten (ESP8266 GPIO12/14, ESP32 GPIO1/2) met bewaarde totalen (herstelbaar uit een retained MQTT-waarde bij boot); `S0Reading` als consistente momentopname voor Modbus.
- **`gpio.cpp/.h`**: extra GPIO's, ook via rules (`gpio()`) en MQTT (`gpio/`).
- **`HeishaOT.cpp/.h`**: OpenTherm-thermostaat (`?`-variabelen, `mqttOTCallback`).
- **`src/common/`**: base64, sha1, string-helpers, geheugenpool (`mem.cpp`, `MEMPOOL_SIZE`), `timerqueue` en de eigen `webserver.*`.

---

## 8. Nieuwe topics vinden en toevoegen

### 8.1 Topics vinden door mee te luisteren

De warmtepomp stuurt elke ~2 seconden 203 bytes. 199 topics zijn gekoppeld; er zijn nog bytes met onbekende betekenis. Door pakketten te vergelijken terwijl je instellingen wijzigt, ontdek je welke byte waarbij hoort.

**Methode 1: hexdump via de web-UI (aanbevolen)**

1. Open de web-UI (`http://heishamon.local` of het IP-adres)
2. Ga naar het **Console**-tabblad en zet **Hexdump** aan
3. Je ziet regels als `data: 71 C8 01 10 56 55 62 49 12 55 55 55 B2 55 ...` (32 bytes per regel)

Tip: permanent aan via **Instellingen → "Debug hexdump from start"**.

**Methode 2: hexdump via MQTT**

1. Schakel hexdump in (web-UI of `http://heishamon.local/togglehexdump`)
2. Subscribe op het log-topic (vervang `panasonic_heat_pump` door jouw `mqtt_topic_base`):
   ```bash
   mosquitto_sub -h <broker-ip> -t "panasonic_heat_pump/log"
   ```

**Methode 3: seriële monitor (geavanceerd)**

USB-serieel op de debug-poort (ESP8266: GPIO1/GPIO3; ESP32: USB-C), 115200 baud. Voor het analyseren van ruwe captures: `Tools/capture_tools.py` en `AquareaParity.md`.

**Een onbekende byte identificeren**

1. **Baseline**: kopieer een volledig pakket terwijl de warmtepomp in rust is
2. **Verander één instelling** (DHW aan/uit, doeltemperatuur, modus, stille modus, …)
3. **Vergelijk**:
   ```
   Baseline:  71 C8 01 10 56 55 62 49 12 55 55 55 B2 55 55 ...
   Na wijzig: 71 C8 01 10 56 55 62 49 12 55 55 55 BE 55 55 ...
                                                    ^^ byte 12: B2 → BE
   ```
   DHW-temperatuur 50 → 62 °C: `178 − 128 = 50`, `190 − 128 = 62` ✓
4. **Verifieer** met meerdere waarden, controleer of de conversie consistent is en kijk in `ProtocolByteDecrypt.md` of de byte al gedocumenteerd is

Aandachtspunten:

- Eén byte kan meerdere waarden als bitgroepen bevatten (bijv. byte 5: holiday mode, schema, dry concrete)
- 16-bit waarden beslaan 2 opeenvolgende bytes (little-endian)
- Temperaturen meestal `waarde − 128`, tellers vaak `waarde − 1`, sensor-offsets in nibbles
- Bytes 0–3 zijn header, byte 202 is de checksum
- Gebruik **listen-only** in de instellingen als je alleen wilt meeluisteren

### 8.2 Een nieuw hoofdtopic toevoegen

Alle wijzigingen zitten in `HeishaMon/decode.h` (plus documentatie). Voorbeeld: byte 164 blijkt de zuigtemperatuur van de compressor te zijn (`waarde − 128 = °C`), nieuw topic `Compressor_Suction_Temp` als TOP199.

1. **Verhoog `NUMBER_OF_TOPICS`**: `199` → `200`
2. **`topicGroup[]`**: voeg een groep toe voor TOP199 (bijv. `TG_DIAGNOSTICS`). Vergeet je dit, dan faalt de `static_assert`.
3. **`topics[]`**, na `"Heat_Pump_Type", //TOP198`:
   ```c
   "Compressor_Suction_Temp", //TOP199
   ```
4. **`topicBytes[]`**, na `192, //TOP198`:
   ```c
   164, //TOP199
   ```
5. **`topicFunctions[]`**, na `getIntMinus1, //TOP198`:
   ```c
   getIntMinus128, //TOP199
   ```
6. **`topicDescription[]`**, na `HeatPumpType, //TOP198`:
   ```c
   Celsius, //TOP199
   ```

Na compileren en flashen verschijnt het topic automatisch als `{mqtt_topic_base}/main/Compressor_Suction_Temp`, in `/json`, de websocket, de rules (`@Compressor_Suction_Temp`) en in Modbus op adres `MAIN_TOPIC_BASE + 199`. Topic-nummers zijn permanent: hernummer of hergebruik nooit bestaande topics.

### 8.3 Speciale gevallen

- **Multi-byte waarden (16-bit)**: voeg een case toe in `getDataValue()` in `decode.cpp`, bijv.:
  ```c
  case 199:
    Topic_Value = String(word(data[183], data[182]) - 1);
    break;
  ```
- **Bitgroep binnen een byte**: gebruik dezelfde byte-positie met de juiste `getBit...`- of nibble-functie.
- **Extra datablok (XTOP)**: verhoog `NUMBER_OF_TOPICS_EXTRA`, voeg toe aan `xtopics[]`, `xtopicBytes[]` en `xtopicFunctions[]` (en de bijbehorende beschrijving); publicatie via `extra/`.
- **Optional PCB (OPT)**: verhoog `NUMBER_OF_OPT_TOPICS` en voeg toe aan `optTopics[]`; publicatie via `optional/`.

### 8.4 Checklist nieuw topic

- [ ] Byte-positie gevonden via hexdump-vergelijking en meerdere keren geverifieerd
- [ ] Conversieformule bepaald
- [ ] `NUMBER_OF_TOPICS` verhoogd
- [ ] Entry toegevoegd aan `topicGroup[]`, `topics[]`, `topicBytes[]`, `topicFunctions[]` en `topicDescription[]`
- [ ] (Optioneel) speciale case in `decode.cpp`
- [ ] `python3 Tools/test_decode.py`, `python3 tests/modbus/run_tests.py` en `Examples/Rules/run_tests.sh` groen
- [ ] Gecompileerd voor ESP8266 én ESP32-S3
- [ ] `MQTT-Topics.md`, `ProtocolByteDecrypt.md` en waar nodig `Modbus-Register-Mapping.md` bijgewerkt

---

## 9. Versies, releases en referentiedocumentatie

`HeishaMon/version.h` bevat `HEISHAMON_VERSION`; CI overschrijft die per ref (tag → release/tagnaam, scheduled/dispatch → `Nightly-<sha>`, anders `Alpha-<sha>`). Nightly-builds maken de release en tag elke build opnieuw aan. Binaries heten `HeishaMon_ESP8266-<type>-<md5>.bin` / `HeishaMon_ESP32-<type>-<md5>.bin`; `binaries/` bevat de vrijgegeven firmware voor eindgebruikers (`model-type-small` = ESP8266, `model-type-large` = ESP32-S3).

Referentiedocumentatie:

- `README.md` (+ `README_DE.md` / `README_NL.md` / `README_FI.md`): installatie, gebruik en rules-syntax
- `MQTT-Topics.md` / `Manage-Topics.md`: alle topics en commando's
- `ProtocolByteDecrypt.md` / `ProtocolByteDecrypt-extra.md`: byte-layout die de decode-tabellen implementeren
- `Modbus-Register-Mapping.md`: Modbus registermap
- `AquareaParity.md`: pariteit en captures
- `HeatPumpType.md`: bevestigde modellen
- `Integrations/`: voorbeeldconfiguraties voor Home Assistant, Domoticz, OpenHAB, Node-RED en ioBroker
