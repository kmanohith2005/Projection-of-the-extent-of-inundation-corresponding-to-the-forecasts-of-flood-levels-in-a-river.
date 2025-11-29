#include <WiFi.h>
#include <HTTPClient.h>
#include <LiquidCrystal_I2C.h>
#include "time.h"  // NTP Time

// Wi-Fi Credentials
const char* ssid = "Bunny";
const char* password = "12345678";

// ThingSpeak API Info
String apiKey = "X99C0MT40DI1VB1Y";
const char* server = "http://api.thingspeak.com/update";

// Ultrasonic Sensor Pins
#define TRIG_PIN 5
#define ECHO_PIN 18

// Output Pins
#define BUZZER_PIN 19
#define RED_LED 4
#define GREEN_LED 15

// SIM900A
#define SIM_RX 26  // Receive data from SIM900A (connect to SIM TX)
#define SIM_TX 27  // Transmit data to SIM900A (connect to SIM RX)

// LCD I2C
LiquidCrystal_I2C lcd(0x27, 16, 2);

// SMS Flags
bool alertSent50 = false;
bool alertSent100 = false;

// NTP Time Settings
const char* ntpServer = "pool.ntp.org";
const long gmtOffset_sec = 19800; // IST
const int daylightOffset_sec = 0;

// Timer variables
unsigned long previousMillis = 0;
const unsigned long interval = 10000;  // 10 seconds

void setup() {
  Serial.begin(115200);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);

  lcd.init();
  lcd.backlight();

  // Connect to WiFi
  WiFi.begin(ssid, password);
  lcd.setCursor(0, 0);
  lcd.print("Connecting WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("WiFi Connected");
  delay(1000);
  lcd.clear();

  // Initialize NTP
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) {
    Serial.println("Waiting for NTP time...");
    delay(1000);
  }

  // Start SIM900A Serial
  Serial2.begin(9600, SERIAL_8N1, SIM_RX, SIM_TX);  // RX, TX

}

float getDistanceCM() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duration = pulseIn(ECHO_PIN, HIGH);
  float distance = duration * 0.034 / 2;
  return distance;
}

String getFormattedTime() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    return "Time N/A";
  }
  char buffer[30];
  strftime(buffer, sizeof(buffer), "%Y-%m-%d %H:%M:%S", &timeinfo);
  return String(buffer);
}

void sendToThingSpeak(float distance) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    String url = String(server) + "?api_key=" + apiKey + "&field1=" + String(distance);
    http.begin(url);
    int httpResponseCode = http.GET();
    http.end();
    Serial.println("Data sent to ThingSpeak: " + String(httpResponseCode));
  }
}

void sendSMS(String message) {
  Serial2.println("AT+CMGF=1");
  delay(1000);
  Serial2.println("AT+CMGS=\"+919030028438\"");  // 🔁 Replace with your number
  delay(1000);
  Serial2.print(message);
  Serial2.write(26);  // Ctrl+Z to send
  delay(3000);
}

void loop() {
  unsigned long currentMillis = millis();

  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;

    float distance = getDistanceCM();
    Serial.print("Distance: ");
    Serial.println(distance);

    lcd.setCursor(0, 1);
    lcd.print("Dist: ");
    lcd.print(distance, 1);
    lcd.print(" cm     ");

    // Flood Detection Logic
    if (distance > 90) {
      digitalWrite(GREEN_LED, HIGH);
      digitalWrite(RED_LED, LOW);
      digitalWrite(BUZZER_PIN, LOW);
      lcd.setCursor(0, 0);
      lcd.print("No Flood Detected ");
      alertSent50 = false;
      alertSent100 = false;
    } else if (distance <= 90 && distance > 50) {
      digitalWrite(GREEN_LED, LOW);
      digitalWrite(RED_LED, LOW);
      digitalWrite(BUZZER_PIN, HIGH);
      lcd.setCursor(0, 0);
      lcd.print("Flood Incoming... ");

      if (!alertSent100) {
        String timeStr = getFormattedTime();
        String message = "Flood Incoming at " + timeStr +
                         ". Water Level: " + String(distance, 1) + " cm.";
        sendSMS(message);
        alertSent100 = true;
      }
    } else if (distance <= 50) {
      digitalWrite(GREEN_LED, LOW);
      digitalWrite(RED_LED, HIGH);
      digitalWrite(BUZZER_PIN, HIGH);
      lcd.setCursor(0, 0);
      lcd.print("Flood Detected!   ");

      if (!alertSent50) {
        String timeStr = getFormattedTime();
        String message = "ALERT! Flood Detected at " + timeStr +
                         ". Water Level: " + String(distance, 1) + " cm.";
        sendSMS(message);
        alertSent50 = true;
      }
    }

    // Send data to ThingSpeak
    sendToThingSpeak(distance);
  }

  }
