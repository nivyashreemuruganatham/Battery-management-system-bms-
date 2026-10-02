#define BLYNK_TEMPLATE_ID "TMPL3XGzfXl97"
#define BLYNK_TEMPLATE_NAME "Battery Management System"
#define BLYNK_AUTH_TOKEN "Ye1ZtyP4-vWZBQ7S66Z1qq-rdg4IGX6_"

#define BLYNK_PRINT Serial
#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

// WiFi Credentials
char ssid[] = "Wokwi-GUEST";
char pass[] = "";

// Hardware Pins
const int CELL_PINS[5] = {32, 33, 34, 35, 4};
const int CURRENT_PIN = 2;
const int DHT_PIN = 23;
const int RELAY_PIN = 26;
const int BUZZER_PIN = 13;
const int GREEN_LED = 27;
const int YELLOW_LED = 14;
const int RED_LED = 12;

#define DHTTYPE DHT22

// Objects
LiquidCrystal_I2C lcd(0x27, 16, 2);
DHT dht(DHT_PIN, DHTTYPE);
BlynkTimer timer;
WidgetTerminal terminal(V19);

// Sensor Data
float cellVoltages[5] = {0};
float prevCellVoltages[5] = {0};
float packVoltage = 0;
float maxCellV = 0;
float minCellV = 0;
float avgCellV = 0;
float cellImbalance = 0;
float batteryCurrent = 0;
float batteryPower = 0;
float temperature = 0;
float humidity = 0;

// Battery Metrics
int soc = 0;
int soh = 100;
int batteryScore = 100;
int operationMode = 0;
String aiDiagnosis = "System Initializing";
String systemStatus = "Booting";
String chargeStatus = "Idle";
String estimatedRuntime = "N/A";

// Fault Flags
bool faultOverV = false;
bool faultUnderV = false;
bool faultOverTemp = false;
bool faultImbalance = false;
bool faultRapidFluct = false;
bool faultSensorFail = false;
bool faultPackFail = false;

// Logging & Notification Flags
bool notifOverV = false;
bool notifUnderV = false;
bool notifOverTemp = false;
bool notifImbalance = false;
bool notifSensorFail = false;
bool notifPackFail = false;

bool loggedOverV = false;
bool loggedUnderV = false;
bool loggedOverTemp = false;
bool loggedImbalance = false;
bool loggedSensorFail = false;
bool loggedPackFail = false;
bool loggedRapidFluct = false;
bool prevRelayState = false;

// Fault Counter
int faultCount = 0;
String lastFault = "None";
unsigned long lastFaultTime = 0;

// State Machine
enum SystemState { STATE_NORMAL, STATE_WARNING, STATE_FAULT, STATE_RECOVERY };
SystemState currentState = STATE_NORMAL;
SystemState prevState = STATE_NORMAL;

// Timers
unsigned long recoveryStartTime = 0;
unsigned long lastBlinkTime = 0;
unsigned long lastLcdUpdate = 0;
bool blinkState = false;
int lcdPage = 0;

String formatTime(unsigned long ms) {
  unsigned long sec = ms / 1000;
  int h = sec / 3600;
  int m = (sec % 3600) / 60;
  int s = sec % 60;
  char buf[12];
  sprintf(buf, "%02d:%02d:%02d", h, m, s);
  return String(buf);
}

void logTerminal(String msg) {
  String fullMsg = "[" + formatTime(millis()) + "] " + msg;
  // Always print to Serial so we can debug even without Blynk
  Serial.println(fullMsg);
  // Only push to Blynk terminal when connected (avoids blocking)
  if (Blynk.connected()) {
    terminal.println(fullMsg);
    terminal.flush();
  }
}

void readSensors() {
  for (int i = 0; i < 5; i++) {
    int adc = analogRead(CELL_PINS[i]);
    cellVoltages[i] = 2.8 + (4.2 - 2.8) * (adc / 4095.0);
  }
  
  int adcCur = analogRead(CURRENT_PIN);
  batteryCurrent = -5.0 + 10.0 * (adcCur / 4095.0);
  
  temperature = dht.readTemperature();
  humidity = dht.readHumidity();
  
  if (isnan(temperature) || isnan(humidity)) {
    faultSensorFail = true;
    temperature = 0.0;
  } else {
    faultSensorFail = false;
  }
}

void detectRapidFluctuation() {
  faultRapidFluct = false;
  for (int i = 0; i < 5; i++) {
    if (abs(cellVoltages[i] - prevCellVoltages[i]) > 0.5) {
      faultRapidFluct = true;
    }
    prevCellVoltages[i] = cellVoltages[i];
  }
}

void calculateBatteryParameters() {
  packVoltage = 0;
  maxCellV = 0;
  minCellV = 5.0;
  
  for (int i = 0; i < 5; i++) {
    packVoltage += cellVoltages[i];
    if (cellVoltages[i] > maxCellV) maxCellV = cellVoltages[i];
    if (cellVoltages[i] < minCellV) minCellV = cellVoltages[i];
  }
  avgCellV = packVoltage / 5.0;
  cellImbalance = maxCellV - minCellV;
  batteryPower = packVoltage * batteryCurrent;
}

void calculateSOC() {
  soc = map((int)(packVoltage * 100), 1400, 2100, 0, 100);
  if (soc < 0) soc = 0;
  if (soc > 100) soc = 100;
}

void calculateSOH() {
  soh = 100;
  if (cellImbalance > 0.2) soh -= 20;
  else if (cellImbalance > 0.1) soh -= 10;

  if (temperature > 60.0) soh -= 20;
  else if (temperature > 50.0) soh -= 10;

  if (currentState == STATE_FAULT) soh -= 30;
  else if (currentState == STATE_WARNING) soh -= 10;

  if (soh < 0) soh = 0;
}

void calculateBatteryScore() {
  batteryScore = soh;
  if (cellImbalance > 0.2) batteryScore -= 20;
  else if (cellImbalance > 0.1) batteryScore -= 10;

  if (temperature > 60.0) batteryScore -= 20;
  else if (temperature > 45.0) batteryScore -= 10;

  if (currentState == STATE_FAULT) batteryScore -= 30;
  else if (currentState == STATE_WARNING) batteryScore -= 10;

  if (faultCount > 0) batteryScore -= (faultCount * 2);
  if (batteryScore < 0) batteryScore = 0;
  if (batteryScore > 100) batteryScore = 100;
}

void detectFaults() {
  faultOverV = false;
  faultUnderV = false;
  faultOverTemp = false;
  faultPackFail = false;
  faultImbalance = false;
  
  for (int i = 0; i < 5; i++) {
    if (cellVoltages[i] > 4.25) faultOverV = true;
    if (cellVoltages[i] < 2.7) faultUnderV = true;
  }
  
  if (temperature > 60.0) faultOverTemp = true;
  if (packVoltage < 13.0 || packVoltage > 22.0) faultPackFail = true;
  if (cellImbalance > 0.15) faultImbalance = true; 
}

void updateStateMachine() {
  prevState = currentState;
  
  bool criticalFault = (faultOverV || faultUnderV || faultOverTemp || faultSensorFail || faultPackFail || cellImbalance > 0.3);
  bool warningCondition = (cellImbalance > 0.15 || temperature > 50.0 || faultRapidFluct);
  
  if (criticalFault) {
    currentState = STATE_FAULT;
  } else if (warningCondition) {
    currentState = STATE_WARNING;
  } else {
    if (currentState == STATE_FAULT) {
      currentState = STATE_RECOVERY;
      recoveryStartTime = millis();
    } else if (currentState == STATE_RECOVERY) {
      if (millis() - recoveryStartTime >= 5000) {
        currentState = STATE_NORMAL;
      }
    } else {
      currentState = STATE_NORMAL;
    }
  }
  
  if (currentState == STATE_RECOVERY && criticalFault) {
    currentState = STATE_FAULT;
  }
  
  switch(currentState) {
    case STATE_NORMAL: operationMode = 0; systemStatus = "NORMAL"; break;
    case STATE_WARNING: operationMode = 1; systemStatus = "WARNING"; break;
    case STATE_FAULT: operationMode = 2; systemStatus = "FAULT"; break;
    case STATE_RECOVERY: operationMode = 3; systemStatus = "RECOVERY"; break;
  }
}

void generateAIDiagnosis() {
  if (currentState == STATE_FAULT) {
    if (faultPackFail) aiDiagnosis = "Pack Failure";
    else if (faultSensorFail) aiDiagnosis = "Sensor Failure";
    else if (faultOverV) aiDiagnosis = "Over Voltage Detected";
    else if (faultUnderV) aiDiagnosis = "Under Voltage Detected";
    else if (faultOverTemp) aiDiagnosis = "Over Temperature";
    else if (cellImbalance > 0.3) aiDiagnosis = "Critical Cell Imbalance";
  } else if (currentState == STATE_RECOVERY) {
    aiDiagnosis = "Recovery Successful";
  } else if (currentState == STATE_WARNING) {
    if (cellImbalance > 0.2) aiDiagnosis = "Battery Performance Degrading";
    else if (cellImbalance > 0.15) aiDiagnosis = "Minor Cell Imbalance";
    else if (faultRapidFluct) aiDiagnosis = "Battery Requires Inspection";
    else aiDiagnosis = "Battery Requires Inspection";
  } else {
    if (batteryCurrent > 0.5) aiDiagnosis = "Charging Normally";
    else if (batteryCurrent < -0.5) aiDiagnosis = "Discharging Normally";
    else aiDiagnosis = "Battery Healthy";
  }
}

void updateRelay() {
  bool relayOn = (currentState == STATE_NORMAL || currentState == STATE_WARNING);
  digitalWrite(RELAY_PIN, relayOn ? HIGH : LOW);
}

void updateLEDs() {
  unsigned long currentMillis = millis();
  if (currentMillis - lastBlinkTime > 500) {
    lastBlinkTime = currentMillis;
    blinkState = !blinkState;
  }
  
  switch(currentState) {
    case STATE_NORMAL:
      digitalWrite(GREEN_LED, HIGH);
      digitalWrite(YELLOW_LED, LOW);
      digitalWrite(RED_LED, LOW);
      break;
    case STATE_WARNING:
      digitalWrite(GREEN_LED, LOW);
      digitalWrite(YELLOW_LED, blinkState ? HIGH : LOW);
      digitalWrite(RED_LED, LOW);
      break;
    case STATE_FAULT:
      digitalWrite(GREEN_LED, LOW);
      digitalWrite(YELLOW_LED, LOW);
      digitalWrite(RED_LED, HIGH);
      break;
    case STATE_RECOVERY:
      digitalWrite(GREEN_LED, blinkState ? HIGH : LOW);
      digitalWrite(YELLOW_LED, blinkState ? LOW : HIGH);
      digitalWrite(RED_LED, LOW);
      break;
  }
}

void updateBuzzer() {
  digitalWrite(BUZZER_PIN, HIGH);
}

void updateLCD() {
  if (currentState == STATE_RECOVERY) {
    int countdown = 5 - ((millis() - recoveryStartTime) / 1000);
    if (countdown < 0) countdown = 0;
    static int lastCountdown = -1;
    if (countdown != lastCountdown) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Recovery Mode");
      lcd.setCursor(0, 1);
      lcd.print("Recovery in ");
      lcd.print(countdown);
      lastCountdown = countdown;
    }
    return;
  }
  
  if (millis() - lastLcdUpdate < 3000) return;
  lastLcdUpdate = millis();
  lcd.clear();
  
  switch(lcdPage) {
    case 0:
      lcd.setCursor(0,0); lcd.print("Pack: "); lcd.print(packVoltage, 2); lcd.print("V");
      lcd.setCursor(0,1); lcd.print("SOC:  "); lcd.print(soc); lcd.print("%");
      break;
    case 1:
      lcd.setCursor(0,0); lcd.print("Temp: "); lcd.print(temperature, 1); lcd.print("C");
      lcd.setCursor(0,1); lcd.print("Curr: "); lcd.print(batteryCurrent, 2); lcd.print("A");
      break;
    case 2:
      lcd.setCursor(0,0); lcd.print("Health: "); lcd.print(soh); lcd.print("%");
      lcd.setCursor(0,1); lcd.print("Score:  "); lcd.print(batteryScore); lcd.print("%");
      break;
    case 3:
      lcd.setCursor(0,0); lcd.print("Max: "); lcd.print(maxCellV, 3); lcd.print("V");
      lcd.setCursor(0,1); lcd.print("Min: "); lcd.print(minCellV, 3); lcd.print("V");
      break;
    case 4:
      lcd.setCursor(0,0); lcd.print("Imbal: "); lcd.print(cellImbalance, 3); lcd.print("V");
      lcd.setCursor(0,1); lcd.print("Pwr: "); lcd.print(batteryPower, 1); lcd.print("W");
      break;
    case 5:
      lcd.setCursor(0,0); lcd.print("AI Diagnosis:");
      lcd.setCursor(0,1); lcd.print(aiDiagnosis.substring(0, 16));
      break;
    case 6:
      lcd.setCursor(0,0); lcd.print("Faults: "); lcd.print(faultCount);
      lcd.setCursor(0,1); lcd.print("Last: "); lcd.print(lastFault.substring(0, 10));
      break;
  }
  lcdPage = (lcdPage + 1) % 7;
}

void updateBlynk() {
  if (!Blynk.connected()) return;   // Guard: no Blynk I/O when disconnected
  
  Blynk.virtualWrite(V0, packVoltage);
  Blynk.virtualWrite(V1, temperature);
  Blynk.virtualWrite(V2, batteryCurrent);
  Blynk.virtualWrite(V3, batteryPower);
  Blynk.virtualWrite(V4, soc);
  Blynk.virtualWrite(V5, soh);
  Blynk.virtualWrite(V6, maxCellV);
  Blynk.virtualWrite(V7, minCellV);
  Blynk.virtualWrite(V8, avgCellV);
  Blynk.virtualWrite(V9, cellImbalance);
  
  String modeStr = "";
  switch(operationMode) {
    case 0: modeStr = "NORMAL"; break;
    case 1: modeStr = "WARNING"; break;
    case 2: modeStr = "FAULT"; break;
    case 3: modeStr = "RECOVERY"; break;
  }
  Blynk.virtualWrite(V10, modeStr);
  
  Blynk.virtualWrite(V11, faultOverV ? 1 : 0);
  Blynk.virtualWrite(V12, faultUnderV ? 1 : 0);
  Blynk.virtualWrite(V13, faultRapidFluct ? 1 : 0);
  Blynk.virtualWrite(V14, faultSensorFail ? 1 : 0);
  Blynk.virtualWrite(V15, digitalRead(RELAY_PIN) == HIGH ? 1 : 0);
  Blynk.virtualWrite(V16, digitalRead(BUZZER_PIN) == HIGH ? 1 : 0);
  Blynk.virtualWrite(V17, systemStatus);
  Blynk.virtualWrite(V18, aiDiagnosis);
  
  Blynk.virtualWrite(V25, batteryScore);
  Blynk.virtualWrite(V26, faultCount);
  Blynk.virtualWrite(V27, lastFault + " @ " + formatTime(lastFaultTime));
  
  if (currentState == STATE_RECOVERY) {
    int countdown = 5 - ((millis() - recoveryStartTime) / 1000);
    if (countdown < 0) countdown = 0;
    Blynk.virtualWrite(V28, "Recovery in " + String(countdown));
  } else {
    Blynk.virtualWrite(V28, "N/A");
  }
  
  Blynk.virtualWrite(V29, formatTime(millis()));
  
  if (batteryCurrent > 0.5) {
    chargeStatus = "Charging";
    float timeToFull = (1.0 - (soc / 100.0)) * 100.0 / batteryCurrent;
    estimatedRuntime = String(timeToFull, 1) + "h to Full";
  } else if (batteryCurrent < -0.5) {
    chargeStatus = "Discharging";
    float timeToEmpty = (soc / 100.0) * 100.0 / abs(batteryCurrent);
    estimatedRuntime = String(timeToEmpty, 1) + "h to Empty";
  } else {
    chargeStatus = "Idle";
    estimatedRuntime = "N/A";
  }
  Blynk.virtualWrite(V30, chargeStatus);
  Blynk.virtualWrite(V31, estimatedRuntime);
}

void updateCharts() {
  if (!Blynk.connected()) return;   // Guard
  Blynk.virtualWrite(V20, cellVoltages[0]);
  Blynk.virtualWrite(V21, cellVoltages[1]);
  Blynk.virtualWrite(V22, cellVoltages[2]);
  Blynk.virtualWrite(V23, cellVoltages[3]);
  Blynk.virtualWrite(V24, cellVoltages[4]);
}

void sendNotifications() {
  if (!Blynk.connected()) return;   // Guard: logEvent must not run disconnected
  
  if (currentState == STATE_FAULT) {
    if (faultOverV && !notifOverV) { Blynk.logEvent("over_voltage", "Over Voltage Detected!"); notifOverV = true; }
    if (faultUnderV && !notifUnderV) { Blynk.logEvent("under_voltage", "Under Voltage Detected!"); notifUnderV = true; }
    if (faultOverTemp && !notifOverTemp) { Blynk.logEvent("over_temp", "Over Temperature Detected!"); notifOverTemp = true; }
    if (cellImbalance > 0.3 && !notifImbalance) { Blynk.logEvent("critical_imbalance", "Critical Cell Imbalance!"); notifImbalance = true; }
    if (faultSensorFail && !notifSensorFail) { Blynk.logEvent("sensor_failure", "Sensor Failure Detected!"); notifSensorFail = true; }
    if (faultPackFail && !notifPackFail) { Blynk.logEvent("pack_failure", "Pack Failure Detected!"); notifPackFail = true; }
  } else {
    notifOverV = false;
    notifUnderV = false;
    notifOverTemp = false;
    notifImbalance = false;
    notifSensorFail = false;
    notifPackFail = false;
  }
}

void logEvents() {
  if (currentState != prevState) {
    if (currentState == STATE_FAULT) {
      faultCount++;
      lastFault = aiDiagnosis;
      lastFaultTime = millis();
      logTerminal("Fault Detected: " + aiDiagnosis);
    } else if (currentState == STATE_RECOVERY) {
      logTerminal("Recovery Started");
    } else if (currentState == STATE_NORMAL && prevState == STATE_RECOVERY) {
      logTerminal("Recovery Completed");
    } else if (currentState == STATE_NORMAL) {
      logTerminal("Battery Healthy");
    }
  }

  bool relayOn = (currentState == STATE_NORMAL || currentState == STATE_WARNING);
  if (relayOn && !prevRelayState) logTerminal("Relay Activated");
  if (!relayOn && prevRelayState) logTerminal("Relay OFF");
  prevRelayState = relayOn;

  if (faultOverV && !loggedOverV) {
    for(int i=0; i<5; i++) if(cellVoltages[i] > 4.25) logTerminal("Cell " + String(i+1) + " Over Voltage");
    loggedOverV = true;
  } else if (!faultOverV) loggedOverV = false;

  if (faultUnderV && !loggedUnderV) {
    for(int i=0; i<5; i++) if(cellVoltages[i] < 2.7) logTerminal("Cell " + String(i+1) + " Under Voltage");
    loggedUnderV = true;
  } else if (!faultUnderV) loggedUnderV = false;

  if (faultOverTemp && !loggedOverTemp) { logTerminal("Over Temperature Detected"); loggedOverTemp = true; } 
  else if (!faultOverTemp) loggedOverTemp = false;

  if (faultSensorFail && !loggedSensorFail) { logTerminal("Sensor Failure Detected"); loggedSensorFail = true; } 
  else if (!faultSensorFail) loggedSensorFail = false;

  if (faultPackFail && !loggedPackFail) { logTerminal("Pack Failure Detected"); loggedPackFail = true; } 
  else if (!faultPackFail) loggedPackFail = false;

  if (faultRapidFluct && !loggedRapidFluct) { logTerminal("Rapid Fluctuation Detected"); loggedRapidFluct = true; } 
  else if (!faultRapidFluct) loggedRapidFluct = false;

  if (faultImbalance && !loggedImbalance) { logTerminal("Cell Imbalance Detected"); loggedImbalance = true; } 
  else if (!faultImbalance) loggedImbalance = false;
}

void periodicTasks() {
  readSensors();
  detectRapidFluctuation();
  calculateBatteryParameters();
  calculateSOC();
  calculateSOH();
  calculateBatteryScore();
  detectFaults();
  updateStateMachine();
  generateAIDiagnosis();
  updateRelay();
  updateLEDs();
  updateBuzzer();
  updateLCD();
  updateBlynk();
  updateCharts();
  sendNotifications();
  logEvents();
}

// Blynk connection callbacks — for diagnostics
BLYNK_CONNECTED() {
  Serial.println(F("[BLYNK] Connected to server"));
  terminal.clear();
  logTerminal("Blynk Connected");
}

void setup() {
  Serial.begin(115200);
  delay(200);
  Serial.println(F("\n==== Smart BMS Booting ===="));

  analogReadResolution(12);
  Serial.println(F("[INIT] ADC resolution = 12-bit"));

  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(GREEN_LED, LOW);
  digitalWrite(YELLOW_LED, LOW);
  digitalWrite(RED_LED, LOW);
  Serial.println(F("[INIT] GPIO configured (relay, buzzer, LEDs)"));

  // Explicit Wire.begin() — some LCD libs don't call it internally
  Wire.begin();
  Serial.println(F("[INIT] I2C bus started"));
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0,0);
  lcd.print("Smart BMS Booting");
  Serial.println(F("[INIT] LCD initialized"));
  
  dht.begin();
  Serial.println(F("[INIT] DHT22 started"));

  // ---- WiFi (non-blocking) ----
  Serial.print(F("[WIFI] Connecting to "));
  Serial.println(ssid);
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, pass);

  // ---- Blynk config (does NOT block) ----
  // Blynk.config() registers the token; Blynk.run() in loop() will
  // establish the connection asynchronously. This avoids the blocking
  // behavior of Blynk.begin() that was hanging the simulation.
  Blynk.config(BLYNK_AUTH_TOKEN);
  Serial.println(F("[BLYNK] Configured (will connect in background)"));

  // ---- Timer ----
  timer.setInterval(1000L, periodicTasks);
  Serial.println(F("[INIT] 1s periodic timer started"));

  Serial.println(F("[INIT] System Started — entering main loop"));
  logTerminal("System Started");
}

void loop() {
  // Blynk.run() handles (re)connection attempts internally when
  // Blynk.config() has been called. It returns quickly if disconnected.
  Blynk.run();
  // timer.run() MUST execute every iteration — this drives periodicTasks()
  timer.run();
}
