#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_TSL2561_U.h>

// ─────────────────────────────────────────
// BROCHES
// ─────────────────────────────────────────
#define PIR_PIN    4
#define MOSFET_PIN 6
#define SDA_PIN    21
#define SCL_PIN    20

// ─────────────────────────────────────────
// SEUILS
// ─────────────────────────────────────────
#define LUX_JOUR  500
#define LUX_NUIT  50

// ─────────────────────────────────────────
// TIMING
// ─────────────────────────────────────────
#define DELAI_EXTINCTION  10000  // 10 secondes
#define DUREE_FADE        5000   // fade 5 secondes
#define DELAI_MANUEL      1800000

// ─────────────────────────────────────────
// TSL2561
// ─────────────────────────────────────────
Adafruit_TSL2561_Unified tsl =
  Adafruit_TSL2561_Unified(TSL2561_ADDR_FLOAT, 12345);

// ─────────────────────────────────────────
// VARIABLES
// ─────────────────────────────────────────
bool   presence           = false;
bool   enExtinction       = false;
bool   modeManuel         = false;
int    intensite          = 0;
int    intensiteAvantFade = 0;
int    heureSimulee       = 14;
int    dernierCompte      = -1;

unsigned long dernierMouvement = 0;
unsigned long debutFade        = 0;
unsigned long dernierManuel    = 0;

// ─────────────────────────────────────────
// SET LED
// ─────────────────────────────────────────
void setLED(int percent) {
  percent   = constrain(percent, 0, 100);
  intensite = percent;
  ledcWrite(MOSFET_PIN, map(percent, 0, 100, 0, 255));
}

// ─────────────────────────────────────────
// NUIT ?
// ─────────────────────────────────────────
bool isNuit() {
  return (heureSimulee >= 22 || heureSimulee < 7);
}

// ─────────────────────────────────────────
// AFFICHE COMMANDES
// ─────────────────────────────────────────
void afficherCommandes() {
  Serial.println("════════════════════════════");
  Serial.println("      COMMANDES SÉRIE       ");
  Serial.println("════════════════════════════");
  Serial.println("ON       → lampe ON 80%");
  Serial.println("OFF      → lampe OFF");
  Serial.println("AUTO     → mode automatique");
  Serial.println("50       → luminosité 50%");
  Serial.println("welcome  → scène bienvenue");
  Serial.println("night    → scène nuit");
  Serial.println("cleaning → scène nettoyage");
  Serial.println("vacancy  → scène vacant");
  Serial.println("════════════════════════════");
}

// ─────────────────────────────────────────
// TRAITEMENT COMMANDE SÉRIE
// ─────────────────────────────────────────
void traiterCommande(String cmd) {
  cmd.trim();
  cmd.toUpperCase();

  if (cmd == "ON") {
    modeManuel    = true;
    dernierManuel = millis();
    setLED(80);
    Serial.println("✓ Lampe ON → 80%");
  }
  else if (cmd == "OFF") {
    modeManuel = true;
    setLED(0);
    Serial.println("✓ Lampe OFF");
  }
  else if (cmd == "AUTO") {
    modeManuel       = false;
    dernierMouvement = millis();
    Serial.println("✓ Mode automatique activé");
  }
  else if (cmd == "WELCOME") {
    modeManuel    = true;
    dernierManuel = millis();
    setLED(80);
    Serial.println("✓ Scène Welcome → 80%");
  }
  else if (cmd == "NIGHT") {
    modeManuel    = true;
    dernierManuel = millis();
    setLED(20);
    Serial.println("✓ Scène Nuit → 20%");
  }
  else if (cmd == "CLEANING") {
    modeManuel    = true;
    dernierManuel = millis();
    setLED(100);
    Serial.println("✓ Scène Nettoyage → 100%");
  }
  else if (cmd == "VACANCY") {
    modeManuel = true;
    setLED(0);
    Serial.println("✓ Scène Vacancy → OFF");
  }
  else {
    int val = cmd.toInt();
    if (val >= 0 && val <= 100) {
      modeManuel    = true;
      dernierManuel = millis();
      setLED(val);
      Serial.print("✓ Luminosité → ");
      Serial.print(val);
      Serial.println("%");
    } else {
      Serial.println("❌ Commande inconnue");
      afficherCommandes();
    }
  }
}

// ─────────────────────────────────────────
// SETUP
// ─────────────────────────────────────────
void setup() {
  Serial.begin(115200);
  delay(1000);

  pinMode(PIR_PIN, INPUT_PULLDOWN);
  ledcAttach(MOSFET_PIN, 5000, 8);
  ledcWrite(MOSFET_PIN, 0);

  Wire.begin(SDA_PIN, SCL_PIN);
  if (!tsl.begin()) {
    Serial.println("ERREUR TSL2561 !");
    while (true) delay(1000);
  }
  tsl.enableAutoRange(true);
  tsl.setIntegrationTime(TSL2561_INTEGRATIONTIME_101MS);

  dernierMouvement = millis();
  Serial.println("Système démarré ✓");
  afficherCommandes();
}

// ─────────────────────────────────────────
// LOOP
// ─────────────────────────────────────────
void loop() {

  // Lecture commande série
  if (Serial.available()) {
    String cmd = Serial.readStringUntil('\n');
    traiterCommande(cmd);
    dernierCompte = -1;
  }

  // Lecture capteurs
  bool pir = digitalRead(PIR_PIN);
  sensors_event_t event;
  tsl.getEvent(&event);
  float lux = event.light ? event.light : 0;

  if (pir) {
    presence         = true;
    dernierMouvement = millis();
    enExtinction     = false;
    dernierCompte    = -1;
    Serial.println("✅ Mouvement détecté !");
  }

  unsigned long timeSansMouvement = millis() - dernierMouvement;

  // Auto OFF manuel après 30 min
  if (modeManuel && intensite > 0 &&
      millis() - dernierManuel > DELAI_MANUEL) {
    modeManuel = false;
    Serial.println("⚡ Auto OFF — 30min sans activité");
  }

  // Mode manuel → skip automatisme
  if (modeManuel) {
    delay(1000);
    return;
  }

  // ─────────────────────────────────────────
  // COUNTDOWN — affiché seulement si présence
  // et pas encore en extinction
  // ─────────────────────────────────────────
  if (presence && !enExtinction && intensite > 0) {
    int secondesRestantes = (DELAI_EXTINCTION - timeSansMouvement) / 1000;
    secondesRestantes = constrain(secondesRestantes, 0, 10);

    // Affiche seulement si le chiffre a changé
    if (secondesRestantes != dernierCompte) {
      dernierCompte = secondesRestantes;
      if (secondesRestantes <= 10 && secondesRestantes > 0) {
        Serial.print("⏱ Extinction dans : ");
        Serial.print(secondesRestantes);
        Serial.println(" sec");
      }
    }
  }

  // Déclenche extinction après 10s
  if (presence && !enExtinction &&
      timeSansMouvement > DELAI_EXTINCTION) {
    enExtinction       = true;
    debutFade          = millis();
    intensiteAvantFade = intensite;
    Serial.println("🔅 Extinction progressive...");
  }

  // Fin extinction
  if (enExtinction && millis() - debutFade > DUREE_FADE) {
    presence      = false;
    enExtinction  = false;
    dernierCompte = -1;
    setLED(0);
    Serial.println("⬛ Lampe éteinte — personne absente");
  }

  int maxBright = isNuit() ? 30 : 100;

  // SCÉNARIO 1 — Absent
  if (!presence && !enExtinction) {
    setLED(0);
  }

  // SCÉNARIO 5 — Extinction progressive
  else if (enExtinction) {
    float prog = (float)(millis() - debutFade) / DUREE_FADE;
    int val    = (int)(intensiteAvantFade * (1.0 - prog));
    setLED(constrain(val, 0, 100));
  }

  // SCÉNARIOS 2/3/4/6 — Présence + Lux
  else if (presence) {
    int brightness = 0;

    if (lux >= LUX_JOUR) {
      brightness = 0;
    }
    else if (lux <= LUX_NUIT) {
      brightness = maxBright;
    }
    else {
      brightness = map((int)lux, LUX_JOUR, LUX_NUIT, 0, maxBright);
      brightness = constrain(brightness, 0, maxBright);
    }

    if (isNuit() && brightness > 30) brightness = 30;
    setLED(brightness);
  }

  delay(1000);
}
