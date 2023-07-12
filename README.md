# Health-monitoring-system

Components Required:

Arduino Uno
Wi-Fi Module (ESP8266)
Blood Pressure Sensor
Glucose Sensor
Heart Rate Sensor
LCD Display (optional for local display)
Breadboard and jumper wires

Arduino cose:

#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BMP280.h>
#include <HeartRate.h>

// Replace with your Wi-Fi credentials
const char* ssid = "Your_SSID";
const char* password = "Your_PASSWORD";

// Replace with your web server IP address
const char* serverIP = "192.168.0.100";
const int serverPort = 80;

// Blood Pressure Sensor Pins
const int bpSensorPin = A0;

// Glucose Sensor Pins
const int glucoseSensorPin = A1;

// Heart Rate Sensor Pins
const int heartRateSensorPin = A2;

// LCD Display Pins (optional)
const int lcdAddress = 0x27;
const int lcdCols = 16;
const int lcdRows = 2;

// Variables
Adafruit_BMP280 bmp;
HeartRate heartRate;

ESP8266WebServer server(80);

void setup() {
  Serial.begin(9600);

  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }

  Serial.println("Connected to WiFi");

  // Start the server
  server.begin();
  Serial.println("Server started");

  // Initialize sensors
  if (!bmp.begin()) {
    Serial.println("Could not find a valid BMP280 sensor, check wiring!");
    while (1);
  }

  // Register handler for updating vital data
  server.on("/update", handleUpdate);
}

void loop() {
  // Read sensor data
  float pressure = bmp.readPressure() / 100.0;  // Convert Pa to hPa
  int glucoseLevel = analogRead(glucoseSensorPin);
  int heartRateValue = getHeartRate();

  // Display data on LCD (optional)
  displayDataOnLCD(pressure, glucoseLevel, heartRateValue);

  // Send data to the server
  sendToServer(pressure, glucoseLevel, heartRateValue);

  server.handleClient();
  delay(1000);
}

int getHeartRate() {
  int heartRateValue = heartRate.getHeartRate();

  if (heartRateValue == 0) {
    Serial.println("Finger not detected or improper placement");
  } else {
    Serial.print("Heart Rate: ");
    Serial.println(heartRateValue);
  }

  return heartRateValue;
}

void displayDataOnLCD(float pressure, int glucoseLevel, int heartRateValue) {
  // Display data on LCD (optional)
  Wire.begin();
  lcd.begin(lcdCols, lcdRows);
  lcd.setBacklight(LOW);
  lcd.setCursor(0, 0);
  lcd.print("Pressure: ");
  lcd.print(pressure);
  lcd.setCursor(0, 1);
  lcd.print("Glucose: ");
  lcd.print(glucoseLevel);
  lcd.setCursor(10, 1);
  lcd.print("HR: ");
  lcd.print(heartRateValue);
}

void sendToServer(float pressure, int glucoseLevel, int heartRateValue) {
  WiFiClient client;

  if (client.connect(serverIP, serverPort)) {
    // Send HTTP POST request to update vital data
    client.println("POST /update-vitals HTTP/1.1");
    client.println("Host: " + String(serverIP));
    client.println("Content-Type: application/x-www-form-urlencoded");
    client.print("pressure=");
    client.println(pressure);
    client.print("glucose=");
    client.println(glucoseLevel);
    client.print("heartRate=");
    client.println(heartRateValue);
    client.println("Connection: close");
    client.println();
    delay(500);
    client.stop();
  }
}

void handleUpdate() {
  if (server.method() == HTTP_POST) {
    float pressure = server.arg("pressure").toFloat();
    int glucoseLevel = server.arg("glucose").toInt();
    int heartRateValue = server.arg("heartRate").toInt();

    // Process received vital data as per your requirements
    // For example, store in a database or perform analysis

    server.send(200, "text/plain", "Vital data updated");
  } else {
    server.send(400, "text/plain", "Invalid request");
  }
}



