# Wi---Fi-Smart-Home-Automation-
Wi-Fi enabled smart home system using ESP32 and sensors to monitor temperature, humidity, light, soil moisture, and distance, with real-time Blynk dashboard control and alerts.
#define BLYNK_TEMPLATE_ID "YOUR_TEMPLATE_ID"
#define BLYNK_TEMPLATE_NAME "Smart Home Monitor"
#define BLYNK_PRINT Serial

#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>
#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// --- Wi-Fi Credentials ---
char ssid[] = "Redmi Note 10T 5G";
char pass[] = "zxcvbnm.";
char auth[] = "YOUR_BLYNK_AUTH_TOKEN"; // From your Blynk device settings

// --- Pin Definitions ---
#define DHTPIN 4          
#define DHTTYPE DHT11     
#define LDR_PIN 34        
#define TRIG_PIN 5        
#define ECHO_PIN 18       
#define SOIL_PIN 35       
#define LED_PIN 2         
#define BUZZER_PIN 23     

// --- Initialize DHT, LCD & Blynk Timer ---
DHT dht(DHTPIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2); 
BlynkTimer timer;

// Timing variables for LCD cycle
unsigned long previousMillis = 0;
const long interval = 3000; 
int displayState = 0;

// Variables to store sensor updates
float humidity = 0;
float temperature = 0;
int ldrValue = 0;
int soilValue = 0;
float distance = 0;

// --- Control LEDs/Buzzer from Blynk App ---
BLYNK_WRITE(V5) {
  int pinValue = param.asInt();
  digitalWrite(LED_PIN, pinValue);
}

BLYNK_WRITE(V6) {
  int pinValue = param.asInt();
  digitalWrite(BUZZER_PIN, pinValue);
}

// --- Send Sensor Data to Blynk Cloud every 2 seconds ---
void sendSensorData() {
  humidity = dht.readHumidity();
  temperature = dht.readTemperature();
  ldrValue = analogRead(LDR_PIN);
  soilValue = analogRead(SOIL_PIN);

  // Trigger Ultrasonic Sensor
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  distance = duration * 0.034 / 2;

  // Send to Blynk Virtual Pins
  Blynk.virtualWrite(V0, isnan(temperature) ? 0.0 : temperature);
  Blynk.virtualWrite(V1, isnan(humidity) ? 0.0 : humidity);
  Blynk.virtualWrite(V2, ldrValue);
  Blynk.virtualWrite(V3, soilValue);
  Blynk.virtualWrite(V4, distance);
}

void setup() {
  Serial.begin(115200);

  // Initialize Pins
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(LDR_PIN, INPUT);
  pinMode(SOIL_PIN, INPUT);
  
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  // Initialize Sensors & Display
  dht.begin();
  lcd.init();
  lcd.backlight();
  
  lcd.setCursor(0, 0);
  lcd.print("Smart Home Sys");
  lcd.setCursor(0, 1);
  lcd.print("Connecting Blynk");

  // Connect to Blynk Cloud
  Blynk.begin(auth, ssid, pass);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Blynk Connected!");
  delay(2000);

  // Setup a function to run every 2 seconds
  timer.setInterval(2000L, sendSensorData);
}

void loop() {
  Blynk.run();
  timer.run();

  // --- Update I2C LCD Cyclically Locally ---
  unsigned long currentMillis = millis();
  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;
    lcd.clear();
    
    if (displayState == 0) {
      lcd.setCursor(0, 0);
      lcd.print("Temp: " + String(isnan(temperature) ? 0.0 : temperature) + "C");
      lcd.setCursor(0, 1);
      lcd.print("Humidity: " + String(isnan(humidity) ? 0.0 : humidity) + "%");
      displayState = 1;
    } else if (displayState == 1) {
      lcd.setCursor(0, 0);
      lcd.print("Light (LDR): " + String(ldrValue));
      lcd.setCursor(0, 1);
      lcd.print("Soil Moist: " + String(soilValue));
      displayState = 2;
    } else {
      lcd.setCursor(0, 0);
      lcd.print("Distance: " + String(distance) + "cm");
      lcd.setCursor(0, 1);
      lcd.print("LED:" + String(digitalRead(LED_PIN) ? "ON " : "OFF") + " BUZ:" + String(digitalRead(BUZZER_PIN) ? "ON" : "OFF"));
      displayState = 0;
    }
  }
}
