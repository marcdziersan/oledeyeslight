1. Ziel und Funktionsbeschreibung

Dieses Arduino-Projekt nutzt einen LDR (Fotowiderstand) + 10 kΩ als Helligkeitssensor und ein SSD1306-OLED (128×64, I²C) zur Darstellung von „Augen“:

SLEEP (schlafen): Augen sind geschlossen (zwei Linien).

AWAKE (wach): Augen sind offen (Kreise + Pupillen).

Aufwachen/ Einschlafen erfolgt über Hysterese-Schwellwerte (THRESH_ON / THRESH_OFF), basierend auf einem geglätteten Analogwert.

Im Wachmodus:

Pupillen bewegen sich weich zu zufälligen Zielen (Blickwechsel 1–5 s).

Es gibt gelegentliches Blinzeln (alle 10–25 s, Dauer ~180 ms).

Alles ist nicht-blockierend (millis), also keine delay()-Stopps.

2. Stückliste (Bill of Materials)

Pflicht:

Arduino UNO

OLED SSD1306 128×64 I²C (typisch Adresse 0x3C)

LDR (Fotowiderstand)

10 kΩ Widerstand (für Spannungsteiler)

Breadboard + Jumperkabel

Optional (empfohlen):

100 nF Kerko nahe OLED-VCC/GND (gegen Störungen)

10 µF Elko nahe OLED-VCC/GND (bei langen Leitungen)

3. Hardware-Aufbau
3.1 OLED (I²C) an Arduino UNO

OLED-Pins (typisch):

GND → GND

VCC → 5V (oder 3.3 V, je nach Modul)

SCL → A5

SDA → A4

Hinweis: Beim UNO sind I²C-Pins auch oft als SCL/SDA extra herausgeführt (zusätzlich zu A4/A5).

3.2 LDR + 10 kΩ Spannungsteiler an A0

Ziel: Aus dem lichtabhängigen Widerstand wird eine analoge Spannung.

Empfohlene Standard-Verschaltung:

5V → LDR → Knotenpunkt → 10 kΩ → GND

Knotenpunkt → A0

Das ergibt:

hell (LDR klein) → Spannung am A0 hoch → Analogwert hoch

dunkel (LDR groß) → Spannung am A0 niedrig → Analogwert niedrig

Wenn deine Werte „verkehrt herum“ laufen (bei dunkel hoch statt niedrig), einfach LDR und 10 kΩ im Teiler tauschen:

5V → 10 kΩ → Knoten → LDR → GND

4. Software-Abhängigkeiten

Benötigte Bibliotheken (Arduino Library Manager):

Adafruit GFX Library

Adafruit SSD1306

Wire (standardmäßig dabei)

Board: Arduino UNO
Serielle Baudrate: 9600

5. Programmstruktur (Übersicht)
5.1 Globale Konfiguration

OLED:

Auflösung: 128×64

Adresse: 0x3C (SCREEN_ADDRESS)

LDR / Analog:

READ_PIN = A0

Hysterese-Schwellen:

THRESH_ON = 670 (ab hier wird wach)

THRESH_OFF = 630 (ab hier wird wieder schlafen)

Glättung (Low-Pass Filter):

filtered = ALPHA * raw + (1-ALPHA) * filtered

ALPHA = 0.15 (0…1; höher = reaktionsschneller, niedriger = ruhiger)

5.2 Zustandsautomaten

Mode

SLEEP

AWAKE

BlinkState

OPEN

CLOSED

5.3 Timing (millis)

Blickzielwechsel: zufällig alle 1–5 s

Pupillen-Schritt: alle 50 ms maximal 1 Pixel pro Achse

Blinkintervall: zufällig 10–25 s

Blinkdauer: 180 ms

6. Ablauf im Detail (Logik)
6.1 setup()

Serial startet (nur Debug).

Zufallsseed: randomSeed(analogRead(A0));

OLED init: display.begin(...)

A0 als Input

Initialwert für filtered wird gesetzt

Startzustand: Eyes Closed (SLEEP)

Timer initialisiert

6.2 loop()

Schritt 1: Messen + Glätten

raw = analogRead(A0)

filtered per ALPHA geglättet

Schritt 2: Moduswechsel (Hysterese)

Wenn SLEEP und filtered >= THRESH_ON → AWAKE

Wenn AWAKE und filtered <= THRESH_OFF → SLEEP

Schritt 3: AWAKE-Verhalten

Wenn Augen offen:

ggf. neues Blickziel

Pupille schrittweise Richtung Ziel

Blinken:

OPEN → nach nextBlinkDelayMs kurz CLOSED

CLOSED → nach BLINK_DURATION_MS wieder OPEN, neuer Delay

7. Parameter-Tuning (wichtig für deinen Aufbau)
7.1 Schwellwerte korrekt bestimmen

Da LDRs, Widerstände, Umgebungslicht und Verkabelung stark variieren, sind 670/630 nur Beispielwerte.

Praktischer Kalibrier-Test:

Temporär in loop() ausgeben:

Serial.print(raw); Serial.print(" "); Serial.println(filtered);

Dann:

Werte bei „dunkel“ notieren

Werte bei „hell“ notieren

Regel:

THRESH_ON etwas unterhalb „hell“-Wert

THRESH_OFF etwas oberhalb „dunkel“-Wert

Abstand (Hysterese) typ. 20–80 Schritte, je nach Rauschen

7.2 ALPHA einstellen

0.05–0.10: sehr ruhig, aber träge

0.15: guter Standard

0.25–0.35: reagiert schnell, aber kann flackern

Wenn dein System „zittert“ um die Schwelle:

ALPHA kleiner machen und/oder

THRESH_ON/OFF weiter auseinander ziehen

8. Typische Fehlerbilder und Lösungen
8.1 OLED bleibt schwarz

I²C-Adresse stimmt evtl. nicht (0x3C vs 0x3D).

SDA/SCL vertauscht?

VCC falsch (manche Module wollen 3.3 V, die meisten 5 V tolerant)

GND fehlt oder Wackelkontakt

8.2 Augen flackern / wechseln schnell zwischen SLEEP und AWAKE

Schwellwerte zu nah beieinander

Filter zu „schnell“ (ALPHA zu hoch)

Verkabelung zu lang / Störungen → Kondensator an OLED und sauberer Aufbau

8.3 Analogwert „verkehrt herum“

Spannungsteiler anders herum gesteckt → LDR/10k tauschen oder Schwellenlogik invertieren.

8.4 Random wirkt nicht random

Seed über A0 ist ok, kann aber bei sehr stabilem Licht ähnlich starten.

Alternative: unbeschalteten Analogpin seed nutzen (z. B. A1 frei lassen und analogRead(A1)).

9. Hinweise zur Performance / Darstellung

display.display() ist relativ teuer; dein Code ruft es nur dann auf, wenn sich tatsächlich etwas ändert (Pupillenschritt, Blinkwechsel). Das ist sinnvoll.

Pupillen bewegen sich pixelweise, dadurch wirkt es „organisch“ statt ruckartig.

10. Erweiterungen (optional, aber passend)

Helligkeitsanzeige (Debug im OLED): kleiner Balken oben.

„Schlafphase“ mit Atemanimation: Augenlinien minimal wippen.

Mehr Modi: neugierig, überrascht (größere Pupillen), müde (halboffene Augen).

Blick folgt Lichtänderung: statt random Ziele aus filtered-Dynamik ableiten.

Auto-Kalibrierung: Min/Max über die ersten 5 Sekunden sammeln und daraus Schwellen berechnen.

11. Schaltplan in Textform (kompakt)

OLED:

OLED GND → UNO GND

OLED VCC → UNO 5V

OLED SDA → UNO A4

OLED SCL → UNO A5

LDR-Teiler (Variante: hell = hoher Wert):

UNO 5V → LDR → (Knoten) → A0

(Knoten) → 10 kΩ → UNO GND

12. Betrieb / Testprotokoll

Verkabelung prüfen (GND gemeinsam, I²C korrekt).

Sketch flashen.

Start: Augen geschlossen.

LDR beleuchten (Handy-Lampe) → Augen öffnen, Pupillen bewegen.

Licht wegnehmen → Augen schließen.
