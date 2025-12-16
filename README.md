# Arduino OLED Eyes – LDR Wake/Sleep Demo

Ein spielerisches Arduino‑Projekt mit **OLED‑Display (SSD1306)** und **Fotowiderstand (LDR)**.  
Je nach Umgebungshelligkeit „schlafen“ oder „wachen“ animierte Augen, inklusive zufälliger Blickbewegungen und realistischem Blinzeln.

---

## ✨ Features

- **Wake / Sleep per Licht**
  - Dunkel → Augen geschlossen
  - Hell → Augen offen
- **Hysterese‑Logik** verhindert Flackern
- **Geglätteter Analogwert** (Low‑Pass‑Filter)
- **Lebendige Animation**
  - Zufällige Blickrichtungen
  - Weiche Pupillenbewegung
  - Natürliches Blinzeln
- **Nicht‑blockierend**
  - Komplett auf `millis()` basierend
  - Keine `delay()`‑Aufrufe

---

## 🧰 Hardware

### Benötigte Komponenten

- Arduino **UNO**
- OLED **SSD1306 128×64 (I²C)**
- **Fotowiderstand (LDR)**
- **10 kΩ Widerstand**
- Breadboard & Jumperkabel

### Optional (empfohlen)

- 100 nF Keramik‑Kondensator (Entstörung)
- 10 µF Elko (stabile OLED‑Versorgung)

---

## 🔌 Verdrahtung

### OLED (I²C)

| OLED | Arduino UNO |
|----|----|
| GND | GND |
| VCC | 5V |
| SDA | A4 |
| SCL | A5 |

*(Manche Module akzeptieren auch 3.3 V – Datenblatt prüfen.)*

### LDR + 10 kΩ Spannungsteiler

Standard‑Variante (hell = hoher Analogwert):

```
5V ── LDR ──┬── A0
            │
          10kΩ
            │
           GND
```

Falls die Werte invertiert sind, LDR und 10 kΩ tauschen.

---

## 📦 Software

### Benötigte Bibliotheken

Installierbar über den Arduino Library Manager:

- **Adafruit GFX Library**
- **Adafruit SSD1306**
- **Wire** (Standard)

### Board & Einstellungen

- Board: **Arduino UNO**
- Baudrate (optional Debug): **9600**
- OLED‑Adresse: `0x3C`

---

## 🧠 Funktionsprinzip

### Zustände

- **SLEEP**
  - Augen geschlossen
- **AWAKE**
  - Augen offen
  - Pupillen bewegen sich zufällig
  - Gelegentliches Blinzeln

### Hysterese

```cpp
THRESH_ON  = 670; // Wechsel zu AWAKE
THRESH_OFF = 630; // Wechsel zu SLEEP
```

Dadurch kein permanentes Umschalten bei Grenzwerten.

### Glättung

```cpp
filtered = ALPHA * raw + (1 - ALPHA) * filtered;
```

- `ALPHA = 0.15` → guter Kompromiss aus Ruhe und Reaktion

---

## ⏱️ Timings

| Funktion | Wert |
|-------|------|
| Blickwechsel | 1–5 s |
| Pupillen‑Step | 50 ms |
| Blinzeln | alle 10–25 s |
| Blinkdauer | 180 ms |

---

## 🛠️ Kalibrierung (wichtig)

Jeder LDR reagiert anders.

1. Temporär im Code ausgeben:
   ```cpp
   Serial.println(filtered);
   ```
2. Messwerte notieren:
   - **dunkel**
   - **hell**
3. Schwellen anpassen:
   ```cpp
   THRESH_ON  knapp unter hell
   THRESH_OFF knapp über dunkel
   ```

Empfohlene Hysterese: **20–80 ADC‑Stufen**.

---

## 🧪 Typische Probleme

### OLED bleibt schwarz
- Falsche I²C‑Adresse (`0x3C` vs `0x3D`)
- SDA / SCL vertauscht
- Versorgungsspannung prüfen

### Flackern zwischen Sleep/Wake
- Schwellen zu nah beieinander
- `ALPHA` zu hoch
- Unruhige Stromversorgung

### Analogwert „falsch herum“
- Spannungsteiler tauschen

---

## 🚀 Erweiterungen

- Licht‑abhängige Blickrichtung
- Pupillengröße dynamisch
- „Müde“ oder „neugierige“ Modi
- Auto‑Kalibrierung beim Start
- Debug‑Overlay (Helligkeitsbalken)

---

## 📄 Lizenz

Freie Nutzung für Lern‑ und Bastelprojekte.  
Keine Garantie – Einsatz auf eigene Verantwortung.

---

## 👤 Autor

Projekt & Idee: **Marcus**  
Arduino / Embedded Spielereien mit Fokus auf Lernen & Experimentieren.
