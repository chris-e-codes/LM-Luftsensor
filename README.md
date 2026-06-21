# LM-Luftsensor — DIY-Luftqualitäts-Monitor (SEN5x + ESP32-C3 + Runddisplay)

Ein kleiner, selbstgebauter Luftqualitäts-Monitor für Werkstatt / Maker-Space /
3D-Druck-Raum. Er misst mit einem einzigen Sensirion **SEN55** den **Feinstaub
(PM1.0 / 2.5 / 4 / 10), einen VOC-Index, einen NOx-Index sowie Temperatur und
Luftfeuchte**, zeigt eine **Ampel** live auf einem runden GC9A01-Display und streamt
alles per MQTT ins Smart Home (iobroker / Home Assistant) und ein fertiges
**Grafana-Dashboard**.

Ursprünglich gebaut, um *eine* Frage zu beantworten — *„senkt mein Luftfilter
wirklich die Belastung, wenn ich ABS/ASA drucke?"* — taugt er genauso als
allgemeiner Innenraum-Luftmonitor.

> ⚠️ **Hinweis zum VOC-Wert:** Der VOC- (und NOx-) Wert ist ein *relativer Index*,
> keine absolute Konzentration. Siehe [Werte richtig lesen](#-werte-richtig-lesen).

---

## ✨ Features

- **Ein Sensor, sechs Werte:** PM1.0 / PM2.5 / PM4 / PM10 (µg/m³), VOC-Index, NOx-Index, Temperatur, Luftfeuchte
- **Runddisplay** mit grün/gelb/roter **Ampel** — Luftstatus auf einen Blick, ganz ohne App
- **MQTT-Ausgang** → läuft mit iobroker, Home Assistant, Node-RED, …
- **Grafana-Dashboard** dabei (portabel, mit Abfrage von Datenquelle & Prefix)
- **Standalone-fähig** — der ESP läuft autark; das Smart-Home-Teil ist optional
- Günstig (~60 €), kein Löten nötig (alles steckbar)

---

## 🧱 Stückliste

| # | Teil | Spec | ~Preis | Bezug |
|---|------|------|--------|-------|
| 1 | **Sensirion SEN55** | `SEN55-SDN-T` — PM + VOC + NOx + Temp/Feuchte, I²C | 30–38 € | [Reichelt](https://www.reichelt.de/de/de/index.html?ACTION=446&LA=0&nbc=1&q=SEN55-SDN-T) · [BerryBase](https://www.berrybase.de/search?search=SEN55) |
| 2 | **ESP32-C3 Dev-Board** | beliebiges C3, ≥ 4 MB Flash, USB-C | 4–10 € | [Amazon](https://www.amazon.de/dp/B0F8QQG1WM?&linkCode=ll2&tag=layermeister-21&linkId=b472edad2cf0129e305519de0df01b14&ref_=as_li_ss_tl) · [BerryBase](https://www.berrybase.de/search?search=ESP32-C3) |
| 3 | **GC9A01 Runddisplay** | 1,28″, 240×240, SPI, 7-Pin | 7–13 € | [Amazon](https://www.amazon.de/dp/B0G1B7ST2V?tag=layermeister-21) · [BerryBase](https://www.berrybase.de/search?search=GC9A01) |
| 4 | **JST-GH-Kabel** | 1,25 mm Raster, 6-polig, Buchse-auf-offen (SEN5x → C3) | 3–9 € | [Amazon](https://www.amazon.de/dp/B0FCMC2ZJK?&linkCode=ll2&tag=layermeister-21&linkId=68a4e1e3535c174cb71cdf6f25aeb828&ref_=as_li_ss_tl) · [BerryBase CAB-18079](https://www.berrybase.de/search?search=CAB-18079) |
| 5 | **Dupont-Jumper-Kabel** | fürs Display (7 Adern) + USB-C-Kabel für Strom | paar € | [Amazon](https://www.amazon.de/dp/B0F98KBV61?th=1&linkCode=ll2&tag=layermeister-21&linkId=f7ff9f3f906449737911cafe1f88587a&ref_=as_li_ss_tl) |

> 💡 **SEN55 vs. SEN66:** Der SEN66 hat zusätzlich CO₂, ist aber teurer und schlechter
> lieferbar. 3D-Druck erzeugt kein CO₂, daher ist der SEN55 für diesen Zweck die bessere
> Wahl. (CO₂ lässt sich später per SCD41 nachrüsten — siehe [Roadmap](#-roadmap).)

> 🔗 Die Amazon-Links sind Affiliate-Links (Tag `layermeister-21`) — siehe [Affiliate-Hinweis](#-affiliate-links).

---

## 🔌 Verdrahtung

Alles läuft auf **einem ESP32-C3**. Der SEN55 spricht **I²C**, das Display spricht **SPI**.

### SEN55 → ESP32-C3 (I²C)

> Die Aderfarben des 6-poligen JST-GH-Kabels sind beliebig — geh nach **Pin-Position**
> und prüf vor dem Einschalten gegen das [SEN55-Datenblatt](https://sensirion.com/products/catalog/SEN55)
> (ein vertauschtes VDD/GND kann den Sensor zerstören).

| SEN55-Pin | Funktion | → ESP32-C3 |
|-----------|----------|------------|
| 1 | **VDD** | **5V** (NICHT 3,3 V) |
| 2 | GND | GND |
| 3 | SDA | GPIO5 |
| 4 | SCL | GPIO6 |
| 5 | **SEL** | **GND** ← wählt I²C-Modus, Pflicht! |
| 6 | NC | offen lassen |

### GC9A01 Display → ESP32-C3 (SPI)

| Display-Pin | → ESP32-C3 |
|-------------|------------|
| VCC | 3,3 V |
| GND | GND |
| SCL (CLK) | GPIO4 |
| SDA (MOSI) | GPIO3 |
| DC | GPIO10 |
| CS | GPIO7 |
| RES (RST) | **GPIO1** ← der Pin mit der Beschriftung **„1"**, *nicht* der **„RST"**-Pin des Boards! |

```
                ESP32-C3
            ┌───────────────┐
   SEN55 ───┤ 5V GND 5/6    │   (I²C: SDA=5, SCL=6, SEL→GND, VDD=5V)
            │               │
 GC9A01 ────┤ 3V3 GND 3/4   │   (SPI: CLK=4, MOSI=3, DC=10, CS=7, RES=1)
            │  7  10  1      │
            └──────┬────────┘
                   │ USB-C (Strom + erstes Flashen)
```

> **⚠️ Häufige Falle:** Generische C3-Boards haben einen Pin **„RST"** (Chip-Reset)
> direkt neben **„1"** (GPIO1). Das RES des Displays muss an **„1"**, nicht an „RST".

---

## ⚡ ESP32-C3 flashen

Die Firmware ist **ESPHome**. Du schreibst keinen Code — du flashst eine Config.

```bash
# 1. ESPHome installieren
pip install esphome          # oder offizielles Docker-Image / Home-Assistant-Add-on

# 2. Secrets anlegen
cd esphome
cp secrets.example.yaml secrets.yaml
#   -> secrets.yaml bearbeiten: WLAN + MQTT-Broker

# 3. Flashen (erstes Mal per USB, danach OTA)
esphome run air-monitor.yaml
```

Nach dem Upload springt ESPHome in die **Log-Ansicht** — du solltest sehen:

```
[i2c] Found device at address 0x69      <- SEN55 richtig verkabelt ✅
[sensor] 'PM2.5': Sending state 1.2 µg/m³
[sensor] 'VOC Index': Sending state 100
```

Der VOC-Index braucht **~24 h, um seine Baseline zu lernen** (anfangs liest er niedrig — normal).

Config: [`esphome/air-monitor.yaml`](esphome/air-monitor.yaml) · Secrets-Vorlage: [`esphome/secrets.example.yaml`](esphome/secrets.example.yaml)

---

## 📡 Daten-Pipeline & Grafana

Der ESP veröffentlicht jeden Wert per MQTT unter `air-monitor/sensor/<name>/state`.

**Das Setup dieses Projekts** (eines von vielen möglichen):

```
SEN55 → ESP32-C3 → MQTT → iobroker → InfluxDB → Grafana
```

- **iobroker:** *MQTT-Adapter* (als Broker) + *InfluxDB-Adapter* installieren, dann das
  History-Logging für die `air-monitor.*`-States aktivieren. Sie landen in InfluxDB als
  `mqtt.0.air-monitor.sensor.<feld>.state`.
- **Home Assistant:** MQTT/iobroker komplett überspringen — den `mqtt:`-Block in der Config
  gegen den nativen `api:`-Block tauschen (in der yaml auskommentiert) und HA-Dashboards
  oder Grafana mit InfluxDB/Prometheus nutzen.

### Grafana-Dashboard importieren

1. Grafana → **Dashboards → New → Import**
2. [`grafana/dashboard.json`](grafana/dashboard.json) **hochladen**
3. Beim Prompt deine **InfluxDB**-Datenquelle auswählen
4. Oben im Dashboard die **`prefix`**-Variable auf deinen Mess-Pfad setzen
   (Default `mqtt.0.air-monitor.sensor` — an deine iobroker-Instanz / dein Topic anpassen)

Das Dashboard bringt: 4 Status-Kacheln (PM2.5, PM10, VOC, Temp) mit Gesundheits-Schwellen,
einen Feinstaub-Verlauf, einen Gas-Index-Verlauf (mit Baseline-Referenzlinien) und einen
Klima-Graph — alles mit geteiltem Fadenkreuz, damit du Ereignisse über die Graphen
hinweg ablesen kannst.

---

## 📖 Werte richtig lesen

Genau hier liegen die meisten DIY-Luftmonitore daneben, also einmal lesen:

### PM (PM1.0 / 2.5 / 4 / 10) — **absolut**, einfach
Echte Konzentrationen in **µg/m³**. Niedriger = sauberer. **PM2.5** ist der Leitwert.

| PM2.5 (µg/m³) | Bedeutung |
|---|---|
| 0–5 | exzellent |
| 5–15 | gut (WHO 24 h ≈ 15) |
| 15–35 | mäßig |
| 35–55 | für Empfindliche ungesund |
| 55+ | ungesund |

*(Die wirklich ultrafeinen Druckpartikel < 0,3 µm liegen unter der Sensor-Reichweite, PM2.5
**unterschätzt** sie also leicht — der relative Anstieg beim Drucken ist trotzdem valide.)*

### VOC-Index — **relativ**, der knifflige
**Keine** Konzentration. Ein adaptiver Index **1–500 mit Bezugspunkt 100**, wobei 100 = der
gleitende ~24-h-Durchschnitt für *diesen* Raum. Der Sensor ist eine elektronische „Nase",
die auf *alle* Gase zusammen reagiert.

- **Auf die Veränderung schauen, nicht auf die absolute Zahl.** Ein Sprung von ~100 → 300
  beim Druckstart ist das Signal — nicht der Rohwert.
- Er **re-zentriert sich**: ein dauerhaft müffelnder Raum driftet über ~24 h zurück auf 100.
  Also ein **Ereignis-Melder, kein Langzeit-Gütesiegel**.
- Temperatur/Feuchte können ihn ebenfalls bewegen — ein wärmer werdender Raum gast mehr aus,
  unabhängig von jeder neuen Quelle.

| VOC-Index | grobe Bedeutung |
|---|---|
| ≤ 150 | normal |
| 150–250 | leicht erhöht |
| 250–400 | erhöht |
| 400–500 | hoch |

### NOx-Index
Gleiche adaptive Idee, aber **Baseline = 1** (nicht 100). Steigt nur bei **Verbrennung**
(Gasherd, Heizung, Abgase, Rauch). **3D-Druck erzeugt kein NOx**, fürs Filter-/Emissions-
Testen sitzt der Wert also bei 1 — nur als „brennt irgendwo was?"-Indikator nützlich.

> 🧪 **Echten Filter-/Emissionstest machen?** Halte die Variablen kontrolliert: Baseline →
> Druck starten → Filter zuschalten, immer nur **eine** Sache ändern, Lüftung/Tageszeit
> konstant halten. Sonst machen Confounder (Lüftung, Sonneneinstrahlung, die Baseline-Drift
> des Sensors) die Daten unzuordenbar.

---

## 🎨 Display anpassen

Das Display wird im ESPHome-`lambda:` gezeichnet (reines C++). Einfache Tweaks:

- **Schwellen / Farben:** die `if (voc_val > 250)`-Zeilen (und die PM-Zeilen) anpassen.
- **Ringdicke:** `it.circle(...)`-Zeilen hinzufügen/entfernen (jede = 1 px).
- **Schriftgröße / Position:** `font_big`-Größe und die `it.printf`-x/y-Werte ändern.
- **Schicke Tachos:** für Arc-/Meter-Optik vom Lambda auf ESPHomes `lvgl:`-Komponente wechseln.

---

## 🛠️ Troubleshooting

Hart erarbeitet beim Bauen — das spart dir Stunden:

| Symptom | Ursache & Fix |
|---------|---------------|
| **Display zeigt nur statisches Farb-Rauschen** | Du nutzt das veraltete `ili9xxx`. **Nimm `mipi_spi`** (Modell `GC9A01A`) — das ist *der* Fix. Die Verkabelung ist meist in Ordnung. |
| Compilefehler `base operand of '->' is not a pointer` | Eine lokale Lambda-Variable heißt wie eine Komponenten-`id` (z. B. `float voc` bei `id: voc`). Umbenennen (`voc_val`). |
| Weißer Hintergrund / falsche Farben | `invert_colors: true` ↔ `false`, und `color_order: bgr` ↔ `rgb`. |
| Nichts am Display, nur RAM-Rauschen, initialisiert nicht | **RES-Ader an GPIO „1", nicht „RST"** prüfen, und CLK/MOSI/DC neu stecken (ein totes Jumper-Kabel = Rauschen). |
| Lücken / Löcher in den Grafana-Linien | InfluxDB-Query `fill(null)` → `fill(none)` (im mitgelieferten Dashboard schon erledigt). |
| `WinError 5` beim `esphome`-Lauf unter Windows | Virenscanner/Editor sperrt `.esphome/build`. Schließen oder Defender-Ausnahme setzen. |
| SEN55 liest nichts / nicht auf I²C gefunden | `SEL` muss an **GND**, und VDD muss **5V** sein. |

---

## 🗺️ Roadmap

- **CO₂:** einen Sensirion **SCD41**-Breakout am selben I²C-Bus ergänzen (ESPHome `scd4x`).
- **Formaldehyd (HCHO):** der **SFA40** ist der richtige Sensor (der ältere SFA30 ist
  abgekündigt), hat aber noch keinen gepflegten ESPHome-Treiber — aufgreifen, sobald eine
  `sfa40`-Komponente auftaucht.
- Ein richtiges Verdrahtungs-Diagramm / Gehäuse-STL.

Beiträge willkommen — gern Issue oder PR aufmachen.

---

## 🤝 Affiliate-Links

Einige Shop-Links in der Stückliste können Affiliate-Links sein. Wenn du darüber kaufst,
unterstützt du dieses Projekt **ohne Mehrkosten für dich**. Du kannst die Teile natürlich
überall kaufen — das Projekt hängt nicht davon ab.

---

## 📄 Lizenz

[MIT](LICENSE) — mach damit, was du willst, ohne Gewährleistung. Gebaut und gepflegt von
[chris-e-codes](https://github.com/chris-e-codes).
