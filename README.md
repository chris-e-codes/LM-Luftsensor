# LM-Luftsensor — DIY Air Quality Monitor (SEN5x + ESP32-C3 + round display)

A small, self-built air quality monitor for the workshop / maker space / 3D-print
room. It measures **particulate matter (PM1.0 / 2.5 / 4 / 10), a VOC index, a NOx
index, temperature and humidity** with a single Sensirion **SEN55** sensor, shows a
live **traffic-light** on a round GC9A01 display, and streams everything over MQTT
into your smart home (iobroker / Home Assistant) and a ready-made **Grafana dashboard**.

It was originally built to answer one question — *"does my air filter actually reduce
the emissions when I print ABS/ASA?"* — but it works just as well as a general indoor
air quality monitor.

> ⚠️ **Note on the VOC value:** the VOC (and NOx) value is a *relative index*, not an
> absolute concentration. See [Understanding the readings](#-understanding-the-readings).

---

## ✨ Features

- **One sensor, six values:** PM1.0 / PM2.5 / PM4 / PM10 (µg/m³), VOC index, NOx index, temperature, humidity
- **Round display** with a green/yellow/red **traffic-light ring** — air status at a glance, no smart-home app needed
- **MQTT output** → works with iobroker, Home Assistant, Node-RED, …
- **Grafana dashboard** included (portable, with data-source & prefix prompts)
- **Standalone-capable** — the ESP runs on its own; the smart-home part is optional
- Cheap (~€60), no soldering required (everything is plug-together)

---

## 🧱 Bill of Materials

| # | Part | Spec | ~Price | Where to get it |
|---|------|------|--------|-----------------|
| 1 | **Sensirion SEN55** | `SEN55-SDN-T` — PM + VOC + NOx + temp/humidity, I²C | €30–38 | [Reichelt](https://www.reichelt.de/de/de/index.html?ACTION=446&LA=0&nbc=1&q=SEN55-SDN-T) · [BerryBase](https://www.berrybase.de/search?search=SEN55) |
| 2 | **ESP32-C3 dev board** | any generic C3, ≥ 4 MB flash, USB-C | €4–10 | [BerryBase](https://www.berrybase.de/search?search=ESP32-C3) · [Amazon](https://www.amazon.de/s?k=ESP32-C3+devkit) |
| 3 | **GC9A01 round display** | 1.28″, 240×240, SPI, 7-pin | €7–13 | [BerryBase](https://www.berrybase.de/search?search=GC9A01) · [Amazon](https://www.amazon.de/s?k=GC9A01+1.28) |
| 4 | **JST-GH cable** | 1.25 mm pitch, 6-pin, socket-to-bare (SEN5x → C3) | €3–9 | [BerryBase CAB-18079](https://www.berrybase.de/search?search=CAB-18079) · [Amazon](https://www.amazon.de/s?k=JST+GH+1.25mm+6+pin+cable) |
| 5 | Jumper wires + USB-C cable | for the display + power | a few € | anywhere |

> 💡 **SEN55 vs SEN66:** the SEN66 adds CO₂ but is pricier and harder to get. CO₂ isn't
> produced by 3D printing, so the SEN55 is the better pick for this use case. (CO₂ can be
> added later via an SCD41 — see [Roadmap](#-roadmap).)

<!-- Maintainer note: the shop links above are plain links. Replace with your own
     affiliate links (Amazon PartnerNet / Awin) if you want — see Affiliate section. -->

---

## 🔌 Wiring

Everything runs on **one ESP32-C3**. The SEN55 talks **I²C**, the display talks **SPI**.

### SEN55 → ESP32-C3 (I²C)

> The 6-pin JST-GH cable's wire colours are arbitrary — go by **pin position**, and
> verify against the [SEN55 datasheet](https://sensirion.com/products/catalog/SEN55) before
> powering on (a swapped VDD/GND can kill the sensor).

| SEN55 pin | Function | → ESP32-C3 |
|-----------|----------|------------|
| 1 | **VDD** | **5V** (NOT 3.3 V) |
| 2 | GND | GND |
| 3 | SDA | GPIO5 |
| 4 | SCL | GPIO6 |
| 5 | **SEL** | **GND** ← selects I²C mode, mandatory! |
| 6 | NC | leave open |

### GC9A01 display → ESP32-C3 (SPI)

| Display pin | → ESP32-C3 |
|-------------|------------|
| VCC | 3.3 V |
| GND | GND |
| SCL (CLK) | GPIO4 |
| SDA (MOSI) | GPIO3 |
| DC | GPIO10 |
| CS | GPIO7 |
| RES (RST) | **GPIO1** ← the pin labelled **"1"**, *not* the board's **"RST"** pin! |

```
                ESP32-C3
            ┌───────────────┐
   SEN55 ───┤ 5V GND 5/6    │   (I²C: SDA=5, SCL=6, SEL→GND, VDD=5V)
            │               │
 GC9A01 ────┤ 3V3 GND 3/4   │   (SPI: CLK=4, MOSI=3, DC=10, CS=7, RES=1)
            │  7  10  1      │
            └──────┬────────┘
                   │ USB-C (power + first flash)
```

> **⚠️ Common trap:** generic C3 boards have a pin labelled **"RST"** (the chip reset)
> right next to **"1"** (GPIO1). The display's RES must go to **"1"**, not "RST".

---

## ⚡ Flashing the ESP32-C3

The firmware is **ESPHome**. You don't write code — you flash a config.

```bash
# 1. install ESPHome
pip install esphome          # or use the official Docker image / Home Assistant add-on

# 2. configure your secrets
cd esphome
cp secrets.example.yaml secrets.yaml
#   -> edit secrets.yaml: Wi-Fi + MQTT broker

# 3. flash (USB the first time, OTA after that)
esphome run air-monitor.yaml
```

After the upload, ESPHome drops into the **log view** — you should see:

```
[i2c] Found device at address 0x69      <- SEN55 wired correctly ✅
[sensor] 'PM2.5': Sending state 1.2 µg/m³
[sensor] 'VOC Index': Sending state 100
```

The VOC index needs **~24 h to learn its baseline** (it reads low at first — that's normal).

Config: [`esphome/air-monitor.yaml`](esphome/air-monitor.yaml) · secrets template: [`esphome/secrets.example.yaml`](esphome/secrets.example.yaml)

---

## 📡 Data pipeline & Grafana

The ESP publishes each value to MQTT under `air-monitor/sensor/<name>/state`.

**This project's setup** (one of many possible):

```
SEN55 → ESP32-C3 → MQTT → iobroker → InfluxDB → Grafana
```

- **iobroker:** install the *MQTT adapter* (as broker) + the *InfluxDB adapter*, then enable
  history logging on the `air-monitor.*` states. They land in InfluxDB as
  `mqtt.0.air-monitor.sensor.<field>.state`.
- **Home Assistant users:** skip MQTT/iobroker entirely — swap the `mqtt:` block in the
  config for the native `api:` block (commented in the yaml) and use HA's own dashboards or
  Grafana with an InfluxDB/Prometheus exporter.

### Import the Grafana dashboard

1. Grafana → **Dashboards → New → Import**
2. **Upload** [`grafana/dashboard.json`](grafana/dashboard.json)
3. When prompted, select your **InfluxDB** data source
4. At the top of the dashboard, set the **`prefix`** variable to your measurement path
   (default `mqtt.0.air-monitor.sensor` — adjust to your iobroker instance / topic)

The dashboard gives you: 4 status tiles (PM2.5, PM10, VOC, temp) with health thresholds,
a particulate-matter trend, a gas-index trend (with baseline reference lines) and a climate
graph — all with a shared crosshair so you can line up events across charts.

---

## 📖 Understanding the readings

This is the part most DIY air monitors get wrong, so read it once:

### PM (PM1.0 / 2.5 / 4 / 10) — **absolute**, easy
Real concentrations in **µg/m³**. Lower = cleaner. **PM2.5** is the headline health value.

| PM2.5 (µg/m³) | meaning |
|---|---|
| 0–5 | excellent |
| 5–15 | good (WHO 24 h ≈ 15) |
| 15–35 | moderate |
| 35–55 | unhealthy for sensitive groups |
| 55+ | unhealthy |

*(The truly ultrafine 3D-print particles < 0.3 µm are below the sensor's range, so PM2.5
slightly **under**-reports them — but the relative rise during printing is still a valid signal.)*

### VOC Index — **relative**, the tricky one
**Not** a concentration. It's an adaptive index **1–500 centred on 100**, where 100 = the
rolling ~24 h average for *this* room. The sensor is an electronic "nose" reacting to *all*
gases together.

- **Watch the change, not the absolute number.** A jump from ~100 → 300 when you start a
  print is the signal — not the raw value.
- It **re-centres**: a permanently smelly room drifts back to 100 over ~24 h. So it's an
  **event detector, not a long-term air-quality grade**.
- Temperature/humidity can move it too — a warming room outgasses more, independent of any
  new source.

| VOC index | rough meaning |
|---|---|
| ≤ 150 | normal |
| 150–250 | slightly elevated |
| 250–400 | elevated |
| 400–500 | high |

### NOx Index
Same adaptive idea, but **baseline = 1** (not 100). Rises only with **combustion** (gas
stove, heater, exhaust, smoke). **3D printing produces no NOx**, so for print-emission
testing this value just sits at 1 — useful only as a "is something burning?" indicator.

> 🧪 **Doing a real filter / emission test?** Control your variables: take a baseline →
> start printing → switch the filter on, changing **only one thing at a time** and keeping
> ventilation/time-of-day constant. Otherwise confounders (ventilation, sunlight warming the
> room, the sensor's own baseline drift) make the data impossible to attribute.

---

## 🎨 Customising the display

The display is drawn in the ESPHome `lambda:` (plain C++). Easy tweaks:

- **Thresholds / colours:** edit the `if (voc_val > 250)` lines (and the PM ones).
- **Ring thickness:** add/remove `it.circle(...)` lines (each is 1 px).
- **Font size / position:** change the `font_big` size and the `it.printf` x/y values.
- **Fancy gauges:** for arc/meter style, switch from the lambda to ESPHome's `lvgl:` component.

---

## 🛠️ Troubleshooting

Hard-won lessons from building this — these will save you hours:

| Symptom | Cause & fix |
|---------|-------------|
| **Display shows only static colour noise** | You're on the deprecated `ili9xxx` platform. **Use `mipi_spi`** (model `GC9A01A`) — this is *the* fix. The wiring is usually fine. |
| Compile error `base operand of '->' is not a pointer` | A lambda local variable has the **same name as a component `id`** (e.g. `float voc` with `id: voc`). Rename it (`voc_val`). |
| Display white background / wrong colours | `invert_colors: true` ↔ `false`, and `color_order: bgr` ↔ `rgb`. |
| Nothing on display, just RAM noise, won't init | Check the **RES wire goes to GPIO "1", not "RST"**, and re-seat CLK/MOSI/DC (a single dead jumper = noise). |
| Gaps / holes in the Grafana lines | Change the InfluxDB query `fill(null)` → `fill(none)` (already done in the included dashboard). |
| `WinError 5` when running `esphome` on Windows | Antivirus/editor is locking `.esphome/build`. Close it or add a Defender exclusion. |
| SEN55 reads nothing / not found on I²C | `SEL` must be tied to **GND**, and VDD must be **5V**. |

---

## 🗺️ Roadmap

- **CO₂:** add a Sensirion **SCD41** breakout on the same I²C bus (ESPHome `scd4x`).
- **Formaldehyde (HCHO):** the **SFA40** is the right sensor (the older SFA30 is EOL), but
  it has no maintained ESPHome driver yet — revisit when an `sfa40` component appears.
- A proper wiring diagram / enclosure STL.

Contributions welcome — open an issue or PR.

---

## 🤝 Affiliate links

Some shop links in the Bill of Materials may be affiliate links. If you buy through them you
support this project **at no extra cost to you**. You're of course free to buy the parts
anywhere — the project doesn't depend on it.

---

## 📄 License

[MIT](LICENSE) — do whatever you like, no warranty. Built and maintained by
[chris-e-codes](https://github.com/chris-e-codes).
