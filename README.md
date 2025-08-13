#include <Wire.h>
#include <SPI.h>
#include <Adafruit_BME280.h>
#include <Adafruit_SSD1306.h>
#include <SD.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <TinyGPSPlus.h>
#include <LoRa.h>
#include <SoftwareSerial.h>

// === Déclarations capteurs ===
Adafruit_BME280 bme;
Adafruit_SSD1306 display(128, 64, &Wire, -1);
TinyGPSPlus gps;

// Config réseau
const char* ssid = "VOTRE_SSID";
const char* pass = "VOTRE_PASS";

// API Cloud
String apiUrl = "http://votre_serveur/api/data";

void setup() {
  Serial.begin(115200);
  Wire.begin();

  // Init BME280
  if (!bme.begin(0x76)) {
    Serial.println("Erreur BME280");
    while (1);
  }

  // Init OLED
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();

  // Init microSD
  if (!SD.begin(5)) {
    Serial.println("Erreur microSD");
  }

  // Init WiFi
  WiFi.begin(ssid, pass);
  while (WiFi.status() != WL_CONNECTED) delay(500);

  // Init LoRa (exemple fréquence EU868)
  LoRa.begin(868E6);
}

void loop() {
  // Lecture capteurs
  float temp = bme.readTemperature();
  float hum  = bme.readHumidity();
  float pres = bme.readPressure() / 100.0F;

  // GPS (si dispo)
  float lat = gps.location.lat();
  float lon = gps.location.lng();

  // Affichage OLED
  display.clearDisplay();
  display.setCursor(0,0);
  display.printf("T: %.2fC\nH: %.2f%%\nP: %.2fhPa", temp, hum, pres);
  display.display();

  // Stockage microSD
  File file = SD.open("/log.csv", FILE_APPEND);
  if (file) {
    file.printf("%.2f,%.2f,%.2f,%.6f,%.6f\n", temp, hum, pres, lat, lon);
    file.close();
  }

  // Transmission WiFi
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(apiUrl);
    http.addHeader("Content-Type", "application/json");
    String payload = "{\"temp\":" + String(temp) +
                     ",\"hum\":" + String(hum) +
                     ",\"pres\":" + String(pres) + "}";
    http.POST(payload);
    http.end();
  } else {
    // Transmission LoRa
    LoRa.beginPacket();
    LoRa.printf("%.2f,%.2f,%.2f", temp, hum, pres);
    LoRa.endPacket();
  }

  delay(60000); // 1 min
}
