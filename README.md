Spielerischer Test

10k resistor
photorisistor
oled
uno

```
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// --------------------
// OLED
// --------------------
#define SCREEN_WIDTH    128
#define SCREEN_HEIGHT   64
#define OLED_RESET      -1
#define SCREEN_ADDRESS  0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// --------------------
// LDR / Analog
// --------------------
const uint8_t READ_PIN = A0;

// Schwellwerte mit Hysterese
const int THRESH_ON  = 670;
const int THRESH_OFF = 630;

// Glättung
float filtered = 0.0f;
const float ALPHA = 0.15f;

// --------------------
// Augen-Geometrie
// --------------------
const int EYE_R  = 10;
const int PUP_R  = 4;

const int L_EYE_X = 40;
const int R_EYE_X = 88;
const int EYE_Y   = 32;

// Max. Pupillen-Abweichung (2D)
const int PUP_MAX_OFF_X = 4;  // links/rechts
const int PUP_MAX_OFF_Y = 3;  // hoch/runter

// Aktueller Pupillen-Offset + Ziel
int pupX = 0, pupY = 0;
int targetX = 0, targetY = 0;

// Timing für Blickwechsel und weiche Bewegung
unsigned long nextGazeChangeAtMs = 0;
unsigned long lastPupilStepMs    = 0;

const unsigned long PUP_STEP_MS = 50; // Bewegungsgeschwindigkeit

// --------------------
// Zustände
// --------------------
enum class Mode { SLEEP, AWAKE };
enum class BlinkState { OPEN, CLOSED };

Mode mode = Mode::SLEEP;
BlinkState blinkState = BlinkState::CLOSED;

// Blink Timing
unsigned long lastBlinkChangeMs = 0;
unsigned long nextBlinkDelayMs  = 1500;
const unsigned long BLINK_DURATION_MS = 180;

// --------------------
// Helpers
// --------------------
unsigned long randRangeUL(unsigned long a, unsigned long b) {
  // inkl. a..b
  return a + (unsigned long)random(0, (long)(b - a + 1));
}

unsigned long pickNextBlinkDelay() {
  return randRangeUL(10000, 25000); // 10..25 Sekunden
}

void pickNewGazeTarget() {
  // Zufälliges Ziel in 2D (inkl. 0)
  targetX = random(-PUP_MAX_OFF_X, PUP_MAX_OFF_X + 1);
  targetY = random(-PUP_MAX_OFF_Y, PUP_MAX_OFF_Y + 1);

  // Nächster Wechsel: 1..5 Sekunden
  nextGazeChangeAtMs = millis() + randRangeUL(1000, 5000);
}

// --------------------
// Rendering
// --------------------
void drawEyesOpen(int offX, int offY) {
  display.clearDisplay();

  // Augenringe
  display.drawCircle(L_EYE_X, EYE_Y, EYE_R, SSD1306_WHITE);
  display.drawCircle(R_EYE_X, EYE_Y, EYE_R, SSD1306_WHITE);

  // Pupillen (2D)
  display.fillCircle(L_EYE_X + offX, EYE_Y + offY, PUP_R, SSD1306_WHITE);
  display.fillCircle(R_EYE_X + offX, EYE_Y + offY, PUP_R, SSD1306_WHITE);

  display.display();
}

void drawEyesClosed() {
  display.clearDisplay();
  display.drawLine(L_EYE_X - 10, EYE_Y, L_EYE_X + 10, EYE_Y, SSD1306_WHITE);
  display.drawLine(R_EYE_X - 10, EYE_Y, R_EYE_X + 10, EYE_Y, SSD1306_WHITE);
  display.display();
}

// Pupille weich Richtung Ziel bewegen (pro Achse 1 Schritt)
bool stepPupilTowardTarget() {
  int oldX = pupX, oldY = pupY;

  if (pupX < targetX) pupX++;
  else if (pupX > targetX) pupX--;

  if (pupY < targetY) pupY++;
  else if (pupY > targetY) pupY--;

  return (pupX != oldX) || (pupY != oldY);
}

void setup() {
  Serial.begin(9600);

  // Zufall initialisieren (bei nur einem LDR am A0 ist das okay;
  // alternativ könnte man einen unbeschalteten Analogpin nehmen)
  randomSeed(analogRead(A0));

  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }

  pinMode(READ_PIN, INPUT);

  filtered = (float)analogRead(READ_PIN);

  // Start: schlafen
  drawEyesClosed();

  // Timer initialisieren
  lastBlinkChangeMs = millis();
  lastPupilStepMs   = millis();
  nextGazeChangeAtMs = millis(); // damit beim Aufwachen direkt ein Ziel gesetzt werden kann
}

void loop() {
  // 1) Messwert lesen + glätten
  int raw = analogRead(READ_PIN);
  filtered = (ALPHA * raw) + ((1.0f - ALPHA) * filtered);

  // 2) Modus per Hysterese
  if (mode == Mode::SLEEP) {
    if (filtered >= THRESH_ON) {
      mode = Mode::AWAKE;
      blinkState = BlinkState::OPEN;

      pupX = 0; pupY = 0;
      pickNewGazeTarget();

      lastBlinkChangeMs = millis();
      nextBlinkDelayMs  = pickNextBlinkDelay();

      lastPupilStepMs = millis();

      drawEyesOpen(pupX, pupY);
    }
  } else { // AWAKE
    if (filtered <= THRESH_OFF) {
      mode = Mode::SLEEP;
      blinkState = BlinkState::CLOSED;
      drawEyesClosed();
      return;
    }
  }

  // 3) Wach-Logik: Blick + Blink (nicht blockierend)
  if (mode == Mode::AWAKE) {
    unsigned long now = millis();

    // 3a) Blickziel zufällig wechseln (nur wenn Augen offen)
    if (blinkState == BlinkState::OPEN) {
      if (now >= nextGazeChangeAtMs) {
        pickNewGazeTarget();
      }

      // Pupille weich bewegen
      if (now - lastPupilStepMs >= PUP_STEP_MS) {
        lastPupilStepMs = now;

        if (stepPupilTowardTarget()) {
          drawEyesOpen(pupX, pupY);
        }
      }
    }

    // 3b) Blinken
    if (blinkState == BlinkState::OPEN) {
      if (now - lastBlinkChangeMs >= nextBlinkDelayMs) {
        blinkState = BlinkState::CLOSED;
        drawEyesClosed();
        lastBlinkChangeMs = now;
      }
    } else { // CLOSED
      if (now - lastBlinkChangeMs >= BLINK_DURATION_MS) {
        blinkState = BlinkState::OPEN;
        drawEyesOpen(pupX, pupY);
        lastBlinkChangeMs = now;
        nextBlinkDelayMs  = pickNextBlinkDelay();
      }
    }
  }
}
```
