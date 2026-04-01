# Nieuwe MQTT-topics vinden en toevoegen aan HeishaMon

Deze handleiding legt uit hoe je **onbekende bytes** in het Panasonic-protocol kunt ontdekken en als nieuwe MQTT-topics aan HeishaMon kunt toevoegen.

---

## Deel 1: Topics vinden door mee te luisteren

### Wat je zoekt

De warmtepomp stuurt elke ~2 seconden een datapakket van **203 bytes** naar HeishaMon. Momenteel zijn **143 topics** (TOP0–TOP142) gekoppeld aan specifieke bytes, maar er zijn nog tientallen bytes waarvan de betekenis onbekend is. Door het pakket te monitoren terwijl je instellingen op de warmtepomp wijzigt, kun je ontdekken welke bytes bij welke functie horen.

### Methode 1: Hexdump via de web-UI (aanbevolen)

1. **Open de HeishaMon webinterface** in je browser (bijv. `http://heishamon.local` of het IP-adres)
2. **Ga naar het Console-tabblad** (rechtsboven in het dashboard)
3. **Zet de "Hexdump" toggle aan** — dit schakelt het loggen van ruwe 203-byte pakketten in
4. Je ziet nu regels verschijnen zoals:
   ```
   data: 71 C8 01 10 56 55 62 49 12 55 55 55 B2 55 ...
   ```
   Dit zijn de 203 bytes in hexadecimaal, 32 bytes per regel.

> **Tip**: Je kunt hexdump ook permanent inschakelen via **Instellingen → "Debug hexdump from start"**

### Methode 2: Hexdump via MQTT

1. **Schakel hexdump in** (via web-UI of HTTP: `http://heishamon.local/togglehexdump`)
2. **Subscribe op het log-topic** in je MQTT-client:
   ```bash
   mosquitto_sub -h <broker-ip> -t "panasonic_heat_pump/log"
   ```
   (Vervang `panasonic_heat_pump` door jouw `mqtt_topic_base`)
3. De ruwe hex-data verschijnt als logberichten

### Methode 3: Seriële monitor (geavanceerd)

1. **Sluit een USB-serieel adapter** aan op de debug-poort:
   - ESP8266: GPIO1/GPIO3 (standaard Serial)
   - ESP32: USB-C poort
2. **Open een seriële monitor** op **115200 baud**
3. Met hexdump ingeschakeld zie je de ruwe pakketten op de seriële console

### Hoe een onbekende byte identificeren

#### Stap 1: Neem een baseline op

Kopieer een volledig 203-byte pakket wanneer de warmtepomp in rust is. Dit is je referentie.

#### Stap 2: Verander één instelling

Wijzig **één ding** op de warmtepomp of via de afstandsbediening. Bijvoorbeeld:
- Zet DHW (warm tapwater) aan of uit
- Verander de doeltemperatuur
- Schakel een modus om (verwarmen → koelen)
- Activeer stille modus

#### Stap 3: Vergelijk de pakketten

Leg de twee pakketten naast elkaar en zoek de byte(s) die veranderd zijn.

**Voorbeeld**:
```
Baseline:  71 C8 01 10 56 55 62 49 12 55 55 55 B2 55 55 ...
Na wijzig: 71 C8 01 10 56 55 62 49 12 55 55 55 BE 55 55 ...
                                                 ^^
                                          Byte 12 veranderd: B2 → BE
```

In dit voorbeeld is byte 12 veranderd van `0xB2` (178 decimaal) naar `0xBE` (190 decimaal).
Als je de DHW-temperatuur van 50°C naar 62°C hebt gezet: `178 - 128 = 50`, `190 - 128 = 62` ✓

#### Stap 4: Verifieer

- Herhaal de test meerdere keren met verschillende waarden
- Controleer of de conversie consistent is (bijv. altijd `waarde - 128 = °C`)
- Vergelijk met `ProtocolByteDecrypt.md` om te zien of de byte al gedocumenteerd is

#### Tips voor het vinden van onbekende bytes

- **Bits vs. bytes**: Sommige bytes bevatten meerdere waarden als bitgroepen (bijv. byte 5 bevat holiday mode, schema, en dry concrete in verschillende bits)
- **16-bit waarden**: Sommige waarden beslaan 2 opeenvolgende bytes (little-endian: lage byte eerst, hoge byte tweede)
- **Offset-patronen**: De meeste temperaturen gebruiken `waarde - 128`, sommige tellers gebruiken `waarde - 1`
- **Bytes 0-3**: Header, niet wijzigen
- **Byte 202**: Checksum, niet als data gebruiken
- **Listen-only modus**: Schakel dit in via de instellingen als je alleen wilt luisteren zonder commando's te sturen

---

## Deel 2: Een nieuw topic toevoegen aan HeishaMon

Wanneer je een byte hebt geïdentificeerd, voeg je het als volgt toe. **Alle wijzigingen vinden plaats in `HeishaMon/decode.h`** (plus optioneel documentatie).

### Stap-voor-stap voorbeeld

Stel: je hebt ontdekt dat **byte 164** de zuigtemperatuur van de compressor bevat, met conversie `waarde - 128 = °C`. Je wilt dit toevoegen als topic `Compressor_Suction_Temp`.

#### 1. Verhoog `NUMBER_OF_TOPICS`

```c
// Was:
#define NUMBER_OF_TOPICS 143

// Wordt:
#define NUMBER_OF_TOPICS 144
```

#### 2. Voeg de topic-naam toe aan `topics[]`

Voeg onderaan het array toe, **na de laatste bestaande entry** (TOP142 = "Expansion_Valve"):

```c
static const char topics[][MAX_TOPIC_LEN] PROGMEM = {
  // ... bestaande topics TOP0 t/m TOP142 ...
  "Expansion_Valve",                //TOP142
  "Compressor_Suction_Temp",        //TOP143  ← NIEUW
};
```

#### 3. Voeg de byte-positie toe aan `topicBytes[]`

Voeg onderaan het array de byte-positie toe:

```c
static const byte topicBytes[] PROGMEM = {
  // ... bestaande entries ...
  175,    //TOP142
  164,    //TOP143  ← NIEUW: byte 164 in het 203-byte pakket
};
```

#### 4. Voeg de conversiefunctie toe aan `topicFunctions[]`

Kies de juiste functie op basis van hoe de byte geïnterpreteerd moet worden:

```c
static const topicFP topicFunctions[] PROGMEM = {
  // ... bestaande entries ...
  getIntMinus1,        //TOP142
  getIntMinus128,      //TOP143  ← NIEUW: waarde - 128 = °C
};
```

**Beschikbare functies**:

| Functie | Berekening | Gebruik voor |
|---------|-----------|--------------|
| `getBit1` | Bit 7 | Enkele aan/uit vlag |
| `getBit1and2` | Bits 7-6, −1 | 2-bit waarde |
| `getBit3and4` | Bits 5-4, −1 | 2-bit waarde (midden) |
| `getBit5and6` | Bits 3-2, −1 | 2-bit waarde |
| `getBit7and8` | Bits 1-0, −1 | 2-bit waarde |
| `getIntMinus128` | waarde − 128 | Temperaturen (°C) |
| `getIntMinus1` | waarde − 1 | Frequentie, snelheid |
| `getIntMinus1Div5` | (waarde − 1) / 5 | Druk, vermogen |
| `getIntMinus1Times10` | (waarde − 1) × 10 | Grote getallen |
| `getIntMinus1Times50` | (waarde − 1) × 50 | Nog grotere getallen |
| `getOpMode` | Bit-lookup | Bedrijfsmodus |

#### 5. Voeg de beschrijving toe aan `topicDescription[]`

Kies het juiste beschrijvingsarray:

```c
static const char **topicDescription[] PROGMEM = {
  // ... bestaande entries ...
  Steps,        //TOP142
  Celsius,      //TOP143  ← NIEUW: eenheid is °C
};
```

**Beschikbare beschrijvingsarrays**: `OffOn`, `DisabledEnabled`, `Celsius`, `Watt`, `Hertz`, `LitersPerMin`, `RotationsPerMin`, `Pressure`, `Hour`, `Counter`, `Ampere`, `OpModeDesc`, `Quiet`, `Powerful`, enzovoort.

### Na het compileren en flashen

- Het topic is automatisch beschikbaar als: `{mqtt_topic_base}/main/Compressor_Suction_Temp`
- De waarde wordt elke ~2 seconden bijgewerkt (alleen bij wijziging, of periodiek)
- Geen verdere code-aanpassingen nodig – de decode-loop itereert automatisch over alle `NUMBER_OF_TOPICS`

---

## Speciale gevallen

### Multi-byte waarden (16-bit)

Als je ontdekt dat een waarde over 2 bytes verdeeld is (bijv. bytes 182-183 voor bedrijfsuren), moet je een **speciale case toevoegen** in de `getDataValue()` functie in `decode.cpp`:

```c
case 143:  // Jouw nieuwe topic
  Topic_Value = String(word(data[183], data[182]) - 1);  // 16-bit waarde
  break;
```

### Bitgroep binnen een byte

Als meerdere topics dezelfde byte delen (maar verschillende bits gebruiken), gebruik dan de juiste `getBitXandY` functie en dezelfde byte-positie in `topicBytes[]`.

### Extra datablok topics (XTOP)

Voor topics uit het extra datablok (header `0x21`):
1. Verhoog `NUMBER_OF_TOPICS_EXTRA`
2. Voeg toe aan `xtopics[]`, `xtopicBytes[]`, en `xtopicFunctions[]`
3. De publicatie gaat via het `extra/` prefix

### Optioneel PCB topics (OPT)

Voor topics van het optionele PCB:
1. Verhoog `NUMBER_OF_OPT_TOPICS`
2. Voeg toe aan `optTopics[]`

---

## Documentatie bijwerken

Vergeet niet om bij een nieuw topic ook de documentatie bij te werken:

1. **`MQTT-Topics.md`** — Voeg een rij toe aan de sensortabel met TOPxx, naam, en beschrijving
2. **`ProtocolByteDecrypt.md`** — Documenteer welke byte(s) je hebt ontdekt en hun betekenis
3. **`Manage-Topics.md`** — Wordt automatisch correct als je de bovenstaande stappen volgt

---

## Checklist nieuw topic

- [ ] Byte-positie geïdentificeerd via hexdump-vergelijking
- [ ] Conversieformule bepaald (−128, −1, /5, bits, etc.)
- [ ] Meerdere keren geverifieerd met verschillende waarden
- [ ] `NUMBER_OF_TOPICS` verhoogd in `decode.h`
- [ ] Topic-naam toegevoegd aan `topics[]` array
- [ ] Byte-positie toegevoegd aan `topicBytes[]` array
- [ ] Conversiefunctie toegevoegd aan `topicFunctions[]` array
- [ ] Beschrijving toegevoegd aan `topicDescription[]` array
- [ ] (Optioneel) Speciale case in `decode.cpp` als multi-byte of complexe logica
- [ ] Gecompileerd en getest
- [ ] `MQTT-Topics.md` bijgewerkt
- [ ] `ProtocolByteDecrypt.md` bijgewerkt
