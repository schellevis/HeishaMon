# HeishaMon – Projectoverzicht

## Wat is HeishaMon?

HeishaMon is een ESP8266/ESP32-gebaseerd IoT-apparaat dat via een seriële verbinding (9600 baud, 8E1) communiceert met **Panasonic Aquarea** lucht-water warmtepompen (H-, J-, K- en L-serie). Het leest 203-byte datapakketten uit de warmtepomp, decodeert deze naar 143+ sensorwaarden, en publiceert ze via **MQTT** naar domoticasystemen zoals Home Assistant, Domoticz, OpenHAB en Node-RED. Daarnaast kunnen instellingen van de warmtepomp worden aangepast via MQTT-commando's.

---

## Architectuur op hoofdlijnen

```
┌─────────────────┐    Serieel 9600 8E1    ┌──────────────┐     MQTT/HTTP      ┌──────────────────┐
│  Panasonic       │◄─────────────────────►│  HeishaMon    │◄──────────────────►│  Domotica        │
│  Aquarea         │   203-byte pakketten  │  ESP8266/     │   143+ topics      │  (Home Assistant, │
│  Warmtepomp      │   110-byte commando's │  ESP32-S3     │   64 commando's    │   Domoticz, etc.) │
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
3. Nieuwere modellen ondersteunen ook een **extra datablok** (header `0x71 0xC8 0x01 0x21`) met vermogensgegevens
4. Elk pakket eindigt met een **checksum**: `(XOR 0xFF) + 1` van alle bytes, zodat de som 0 is

### Commando's naar de warmtepomp

- Commando's gebruiken een 110-byte template (header `0xF1 0x6C 0x01 0x10`)
- Specifieke bytes worden gewijzigd voor de gewenste instelling
- Checksum wordt automatisch berekend en toegevoegd

---

## Hardwareplatforms

| Platform | Chip | Kenmerken |
|----------|------|-----------|
| **Small** | ESP8266 (Wemos D1 mini) | WiFi, ~720KB firmware |
| **Large** | ESP32-S3 | WiFi + Ethernet, PSRAM, NeoPixel LED, ~1.5MB firmware |

### Seriële pinnen

- **ESP8266**: GPIO13 (RX), GPIO15 (TX) – via `Serial.swap()`
- **ESP32**: GPIO18 (RX), GPIO17 (TX) – plus proxy-poort op GPIO9/GPIO8

---

## Bestandsstructuur en componenten

### Hoofdmap (`/`)

| Bestand/map | Doel |
|-------------|------|
| `README.md` | Projectdocumentatie (ook DE/FI/NL vertalingen) |
| `MQTT-Topics.md` | Alle 143+ sensor-topics en 42+ commando-topics |
| `ProtocolByteDecrypt.md` | Mapping van elke byte in het 203-byte pakket |
| `ProtocolByteDecrypt-extra.md` | Extra datablok (vermogensgegevens) |
| `Manage-Topics.md` | Handleiding voor het toevoegen van nieuwe topics |
| `Integrations/` | Configuratievoorbeelden voor Home Assistant, Domoticz, etc. |
| `binaries/` | Voorgecompileerde firmware (ESP8266 & ESP32-S3) |
| `scripts/` | Build-scripts (`build_esp8266.sh`, `build_esp32s3.sh`) |
| `Tools/` | Hulpprogramma's (checksum checker) |

### Broncode (`/HeishaMon/`)

#### Kernbestanden

| Bestand | Regels | Functie |
|---------|--------|---------|
| **HeishaMon.ino** | ~2150 | Hoofdprogramma: WiFi/MQTT setup, seriële communicatie, main loop |
| **decode.h** | ~730 | Definieert alle topic-namen, byte-posities, conversiefuncties en beschrijvingen |
| **decode.cpp** | ~340 | Implementeert decodering: bytes → waarden, publiceert naar MQTT |
| **commands.h** | ~100 | Definieert 64 commando-structuren (naam + functiepointer) |
| **commands.cpp** | ~1000 | Implementeert alle set-commando's (MQTT → bytes → warmtepomp) |

#### Randapparatuur

| Bestand | Functie |
|---------|---------|
| **dallas.h/cpp** | 1-Wire DS18B20 temperatuursensoren |
| **s0.h/cpp** | S0 kWh-meter pulsmetingen (2 poorten) |
| **gpio.h/cpp** | GPIO-configuratie, seriële initialisatie, LED-aansturing |
| **HeishaOT.h/cpp** | OpenTherm-protocol voor thermostaatcommunicatie |

#### Webinterface

| Bestand | Functie |
|---------|---------|
| **webfunctions.h/cpp** | HTTP-server, REST API, instellingenbeheer |
| **htmlcode.h** | Ingebedde HTML/CSS/JavaScript voor de web-UI |

#### Hulpbibliotheken (`/HeishaMon/src/`)

| Map | Inhoud |
|-----|--------|
| **common/** | Webserver, logging, SHA1, Base64, timerqueue, geheugen, string-utilities |
| **opentherm/** | Volledige OpenTherm-protocolimplementatie |
| **rules/** | Lokale regelengine (functies, operators, stack, timer, GPIO-regels) |

---

## Het topic-systeem in detail

### Drie categorieën read-topics

| Categorie | Aantal | Array | Prefix | Bron |
|-----------|--------|-------|--------|------|
| **Hoofdtopics** | 143 (TOP0–TOP142) | `topics[]` | `main/` | 203-byte hoofdpakket |
| **Extra topics** | 6 (XTOP0–XTOP5) | `xtopics[]` | `extra/` | 203-byte extra datablok |
| **Optioneel PCB** | 7 (OPT0–OPT6) | `optTopics[]` | `optional/` | 20-byte optioneel PCB-pakket |

### Hoe een topic gedefinieerd is (in `decode.h`)

Elk topic bestaat uit **vier parallelle arrays** die op dezelfde index werken:

```
Index i ──► topics[i]           = "Heatpump_State"     (MQTT topic-naam)
        ──► topicBytes[i]       = 4                     (byte-positie in het 203-byte pakket)
        ──► topicFunctions[i]   = getBit7and8           (conversiefunctie: byte → waarde)
        ──► topicDescription[i] = OffOn                 (beschrijvingsarray: {"2","Off","On"})
```

### Decodering stap voor stap

1. **Ontvangst**: `readSerial()` leest 203 bytes, valideert header en checksum
2. **Routing**: Byte 3 bepaalt het type (`0x10` → normaal, `0x21` → extra)
3. **Iteratie**: `decode_heatpump_data()` loopt door alle `NUMBER_OF_TOPICS` (143) topics
4. **Extractie**: Voor elk topic:
   - Haal byte op positie `topicBytes[i]` uit het pakket
   - Pas `topicFunctions[i]()` toe (bijv. `getIntMinus128`: waarde − 128 = °C)
   - Speciaal: sommige topics (pump flow, errors, 16-bit waarden) hebben custom logica
5. **Vergelijking**: Vergelijk met vorige waarde (`actData`)
6. **Publicatie**: Bij wijziging → publiceer naar `{base}/main/{topic_naam}` via MQTT

### Beschikbare conversiefuncties (in `decode.cpp`)

| Functie | Werking | Voorbeeld |
|---------|---------|-----------|
| `getBit1` | Bit 7 extraheren | 0/1 |
| `getBit1and2` | Bits 7-6, minus 1 | -1/0/1 |
| `getBit3and4` | Bits 5-4, minus 1 | -1/0/1 |
| `getBit5and6` | Bits 3-2, minus 1 | -1/0/1 |
| `getBit7and8` | Bits 1-0, minus 1 | -1/0/1 |
| `getIntMinus128` | Waarde − 128 | 178 → 50 (°C) |
| `getIntMinus1` | Waarde − 1 | 51 → 50 |
| `getIntMinus1Div5` | (Waarde − 1) / 5 | 251 → 50.0 |
| `getIntMinus1Times10` | (Waarde − 1) × 10 | 6 → 50 |
| `getIntMinus1Times50` | (Waarde − 1) × 50 | 2 → 50 |
| `getOpMode` | Bit-mapping naar modusnummer | 18 → "Heat" |
| `getUintt16` | 16-bit unsigned (little-endian) | 2 bytes → getal |
| `getPumpFlow` | Complexe berekening met 2 bytes | byte 169+170 → l/min |
| `getErrorInfo` | Foutcode-extractie | F-codes, H-codes |
| `unknown` | Niet-geïmplementeerd | Retourneert "unknown" |

### Beschikbare beschrijvingsarrays (in `decode.h`)

Voorbeelden: `OffOn`, `DisabledEnabled`, `OpModeDesc`, `Celsius`, `Watt`, `Hertz`, `LitersPerMin`, `RotationsPerMin`, `Pressure`, `Hour`, `Counter`, `Ampere`, etc.

---

## Commando's (schrijven naar de warmtepomp)

### Structuur (in `commands.h`)

```c
struct cmdStruct {
  char name[29];                                              // MQTT commando-naam
  unsigned int (*func)(char *msg, unsigned char *cmd, char *log_msg);  // Implementatiefunctie
};
```

Er zijn 64 commando's gedefinieerd in `commands[]` (bijv. `SetHeatpump`, `SetDHWTemp`, `SetOperationMode`).

### Hoe een commando werkt (in `commands.cpp`)

1. Kopieer het 110-byte template (`panasonicSendQuery`)
2. Wijzig de relevante byte(s) op basis van de MQTT-payload
3. Retourneer de lengte → `send_command()` berekent checksum en stuurt het naar de warmtepomp

**Voorbeeld**: DHW-temperatuur instellen op 50°C → byte 42 wordt `50 + 128 = 178 (0xB2)`

---

## Build-systeem

- **Tool**: Arduino CLI
- **ESP8266**: `arduino-cli compile --fqbn=esp8266:esp8266:d1_mini` (160MHz, 4MB flash, 2MB SPIFFS)
- **ESP32-S3**: `arduino-cli compile --fqbn=esp32:esp32:esp32s3` (CDC boot, PSRAM, min SPIFFS)
- **CI/CD**: GitHub Actions (`.github/workflows/main.yml`) bouwt automatisch bij tags/PRs
- **Afhankelijkheden**: PubSubClient, ArduinoJson, DallasTemperature, OneWire, RingBuffer, Adafruit NeoPixel

---

## Belangrijke constanten

| Constante | Waarde | Betekenis |
|-----------|--------|-----------|
| `DATASIZE` | 203 | Grootte datapakket van warmtepomp |
| `OPTDATASIZE` | 20 | Grootte optioneel PCB-pakket |
| `PANASONICQUERYSIZE` | 110 | Grootte query/commando naar warmtepomp |
| `NUMBER_OF_TOPICS` | 143 | Aantal hoofdtopics (TOP0–TOP142) |
| `NUMBER_OF_TOPICS_EXTRA` | 6 | Aantal extra topics (XTOP0–XTOP5) |
| `NUMBER_OF_OPT_TOPICS` | 7 | Aantal optionele PCB-topics (OPT0–OPT6) |
| `MAX_TOPIC_LEN` | 42 | Maximale lengte topic-naam |
| `SERIALTIMEOUT` | 2000 | Seriële timeout in ms |

---

## Web-UI functies

- **Dashboard**: Live sensorwaarden
- **Console**: Logberichten + hexdump-toggle voor debug
- **Instellingen**: WiFi, MQTT, serieel, timers, optional PCB, OpenTherm
- **Firmware-update**: OTA (Over-The-Air) via webinterface
- **Rules**: Lokale automatiseringsregels (zonder cloud/WiFi afhankelijkheid)

---

## Integraties (`/Integrations/`)

| Platform | Inhoud |
|----------|--------|
| **Home Assistant** | MQTT autodiscovery configuratie |
| **Domoticz** | MQTT-koppeling handleiding |
| **OpenHAB 2/3** | Items/Things configuratie |
| **Node-RED** | Flow-templates |
| **ioBroker** | Handmatige configuratie |
