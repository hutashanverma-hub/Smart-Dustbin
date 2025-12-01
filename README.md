# Smart-Dustbin
/*
  Smart Wet/Dry Dustbin - Updated
  - Ultrasonic strict cutoff <= 15 cm
  - moisture == 0 => treated as DRY (fallback)
  - Edge-detect (acts only when object first appears)
  - Debounce + averaging for sensors
  - Smooth servo movement, robust close (overshoot + seat)
  - Manual Serial override: 'L' (left/wet), 'R' (right/dry), 'C' (close)
*/

#include <Servo.h>
Servo flap;

// PINS
const int TRIG = 7;
const int ECHO = 6;
const int SERVO = 9;
const int MOIST = A0;

// TUNABLE PARAMETERS
const int DETECT_DISTANCE = 15;    // STRICT cutoff in cm (<= 15cm = present)
const int CONSECUTIVE_NEEDED = 5;  // confirmations required
int moistureDeltaNeeded = 120;     // delta above baseline => WET (tune if needed)
const unsigned long REARM_MILLIS = 1400; // cooldown after an action (ms)

// SERVO ANGLES (adjust to your mechanical mount)
const int LEFT_ANGLE  = 150;  // WET -> left container
const int RIGHT_ANGLE = 18;   // DRY -> right container
int CLOSED_ANGLE      = 78;   // final closed (tune 60..100)
const int OPEN_SMALL  = 100;  // small open before drop

// INTERNAL STATE
int baselineMoist = 0;
int consecutiveCount = 0;
bool lastObjectPresent = false;
unsigned long lastActionTime = 0;
int currentAngle = CLOSED_ANGLE;

// ---------------- helper functions ----------------
long singleDistanceRead() {
  digitalWrite(TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG, LOW);
  long dur = pulseIn(ECHO, HIGH, 30000);
  if (dur == 0) return 999;
  return dur / 58;
}

long avgDistance(int n = 4) {
  long s = 0;
  for (int i = 0; i < n; ++i) {
    s += singleDistanceRead();
    delay(16);
  }
  return s / n;
}

int avgMoisture(int n = 8) {
  long s = 0;
  for (int i = 0; i < n; ++i) {
    s += analogRead(MOIST);
    delay(12);
  }
  return int(s / n);
}

// move smoothly only when target differs; attach/detach servo
void smoothMoveTo(int target) {
  if (target == currentAngle) {
    Serial.print("[SERVO] already at ");
    Serial.println(target);
    return;
  }
  Serial.print("[SERVO] moving from ");
  Serial.print(currentAngle);
  Serial.print(" to ");
  Serial.println(target);
  flap.attach(SERVO);
  int step = (target > currentAngle) ? 1 : -1;
  for (int a = currentAngle; a != target; a += step) {
    flap.write(a);
    delay(6);
  }
  flap.write(target);
  delay(120);
  currentAngle = target;
  flap.detach();
  Serial.println("[SERVO] move done & detached");
}

// robust close: overshoot then return to seat
void robustClose(int finalAngle) {
  Serial.print("[CLOSE] closing to ");
  Serial.println(finalAngle);
  int overshoot = 6; // tweak if needed
  int overAngle = finalAngle + (finalAngle > currentAngle ? overshoot : -overshoot);

  flap.attach(SERVO);
  int step = (overAngle > currentAngle) ? 1 : -1;
  for (int a = currentAngle; a != overAngle; a += step) {
    flap.write(a);
    delay(6);
  }
  flap.write(overAngle);
  delay(110);

  step = (finalAngle > overAngle) ? 1 : -1;
  for (int a = overAngle; a != finalAngle; a += step) {
    flap.write(a);
    delay(8);
  }
  flap.write(finalAngle);
  delay(130);
  currentAngle = finalAngle;
  flap.detach();
  Serial.println("[CLOSE] seated & detached");
}

// ---------------- setup ----------------
void setup() {
  Serial.begin(9600);
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  pinMode(MOIST, INPUT);

  // initial closed position (detach to prevent jitter)
  flap.attach(SERVO);
  flap.write(CLOSED_ANGLE);
  delay(350);
  flap.detach();
  currentAngle = CLOSED_ANGLE;

  // measure stable baseline (ensure pad EMPTY & DRY)
  Serial.println("--- Measuring baseline moisture: keep pad EMPTY & DRY ---");
  baselineMoist = avgMoisture(60);
  Serial.print("Baseline moisture = ");
  Serial.println(baselineMoist);
  Serial.println("System READY. Place waste on pad to test. Manual override: send L/R/C");
  delay(600);
}

// ---------------- main loop ----------------
void loop() {
  // manual serial test (safe)
  if (Serial.available()) {
    char c = Serial.read();
    if (c == 'L' || c == 'l') {
      Serial.println("[MANUAL] LEFT");
      smoothMoveTo(OPEN_SMALL); delay(250);
      smoothMoveTo(LEFT_ANGLE); delay(700);
      robustClose(CLOSED_ANGLE);
      lastActionTime = millis();
    } else if (c == 'R' || c == 'r') {
      Serial.println("[MANUAL] RIGHT");
      smoothMoveTo(OPEN_SMALL); delay(250);
      smoothMoveTo(RIGHT_ANGLE); delay(700);
      robustClose(CLOSED_ANGLE);
      lastActionTime = millis();
    } else if (c == 'C' || c == 'c') {
      Serial.println("[MANUAL] CLOSE");
      robustClose(CLOSED_ANGLE);
    }
    while (Serial.available()) Serial.read(); // clear serial buffer
  }

  unsigned long now = millis();
  if (now - lastActionTime < REARM_MILLIS) {
    // cooling period - sample but do not act
    long d = avgDistance(2);
    int m = avgMoisture(3);
    Serial.print("[COOL] Dist:"); Serial.print(d);
    Serial.print(" cm  Moist:"); Serial.println(m);
    delay(160);
    return;
  }

  long dist = avgDistance(4);
  int moist = avgMoisture(6);

  // STRICT ultrasonic detection: present only <= DETECT_DISTANCE
  bool present = (dist <= DETECT_DISTANCE);

  // debug print
  Serial.print("[READ] Dist:"); Serial.print(dist);
  Serial.print("cm  Moist:"); Serial.print(moist);
  Serial.print("  base:"); Serial.print(baselineMoist);
  Serial.print("  present:"); Serial.print(present ? "YES" : "NO");
  Serial.print("  consec:"); Serial.println(consecutiveCount);

  // Debounce & edge logic:
  if (present) {
    consecutiveCount++;
  } else {
    consecutiveCount = 0;
    lastObjectPresent = false; // reset so next object triggers
  }

  // Only act on rising edge: when previously not present and now confirmed present
  if (!lastObjectPresent && consecutiveCount >= CONSECUTIVE_NEEDED) {
    Serial.println("[EDGE] rising edge -> object JUST appeared");

    // Decision: treat 0 as DRY fallback
    int delta = moist - baselineMoist;
    Serial.print("[DECIDE] delta = "); Serial.println(delta);

    bool isWet = false;
    if (moist == 0) {
      Serial.println("[DECIDE] moisture==0 -> fallback: TREAT AS DRY");
      isWet = false;
    } else {
      if (delta >= moistureDeltaNeeded) isWet = true;
      else isWet = false;
    }

    Serial.print("[DECIDE] final => "); Serial.println(isWet ? "WET (LEFT)" : "DRY (RIGHT)");

    // final stability re-check (abort if huge change)
    int recheck = avgMoisture(6);
    Serial.print("[CHECK] recheck = "); Serial.println(recheck);
    if (abs(recheck - moist) > 200) {
      Serial.println("[ABORT] unstable moisture - ignoring this event");
      consecutiveCount = 0;
      lastObjectPresent = true; // keep high until removed
      delay(200);
    } else {
      // move: tiny open, then left/right, then robust close
      smoothMoveTo(OPEN_SMALL);
      delay(300);
      if (isWet) smoothMoveTo(LEFT_ANGLE);
      else smoothMoveTo(RIGHT_ANGLE);
      delay(900);
      robustClose(CLOSED_ANGLE);
      lastActionTime = millis();
      consecutiveCount = 0;
      lastObjectPresent = true;
    }
  }

  // update lastObjectPresent when object disappears so we can re-trigger next time
  if (!present && lastObjectPresent) {
    Serial.println("[EDGE] object removed -> ready for next");
    lastObjectPresent = false;
  }

  delay(140);
}
