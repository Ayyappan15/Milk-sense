# Milk-sense
#include <Wire.h>

#include <Adafruit_GFX.h>

#include <Adafruit_SSD1306.h>

#include <WiFi.h>

#include <OneWire.h>

#include <DallasTemperature.h>

#include <ESPAsyncWebServer.h>



#define SCREEN_WIDTH 128

#define SCREEN_HEIGHT 64

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);



const char* ssid = "milk";

const char* password = "12345678";



AsyncWebServer server(80);



const int pHpin = A0;

const int gasPin = 33;

const int tempPin = 4;



const float pHThreshold = 6.0;

const float tempThreshold = 10;

const int gasThreshold = 300;



float pHValue = 0;

float temperature = 0;

int gasValue = 0;

String milkStatus = "Unknown";



OneWire oneWire(tempPin);

DallasTemperature sensors(&oneWire);



void readPH() {

  int rawValue = analogRead(pHpin);

  float voltage = rawValue * (3.3 / 4095.0);

  pHValue = 3.3 * voltage;

}



void readTemperature() {

  sensors.requestTemperatures();

  temperature = sensors.getTempCByIndex(0);

}



void readGas() {

  gasValue = analogRead(gasPin);

}



void checkMilkQuality() {

  if (pHValue < pHThreshold || temperature > tempThreshold || gasValue > gasThreshold) {

    milkStatus = "Spoiled";

  } else {

    milkStatus = "Good";

  }

}



void bootOLEDMessage(const String& message) {

  display.clearDisplay();

  display.setTextSize(1);

  display.setTextColor(WHITE);

  display.setCursor(0, 25);

  display.println(message);

  display.display();

  delay(2000);

}



void updateOLED() {

  display.clearDisplay();

  display.setTextSize(1);

  display.setTextColor(WHITE);



  display.setCursor(0, 0);

  display.println("Milk Quality Tester");



  display.setCursor(0, 15);

  display.print("pH: ");

  display.println(pHValue, 2);



  display.setCursor(0, 25);

  display.print("Temp: ");

  display.print(temperature, 2);

  display.println(" C");



  display.setCursor(0, 35);

  display.print("Gas: ");

  display.println(gasValue);



  display.setCursor(0, 50);

  display.print("Status: ");

  display.println(milkStatus);



  display.display();

}



String sensorData() {

  String data = "{";

  data += "\"pH\":" + String(pHValue, 2) + ",";

  data += "\"temperature\":" + String(temperature, 2) + ",";

  data += "\"gas\":" + String(gasValue) + ",";

  data += "\"status\":\"" + milkStatus + "\"";

  data += "}";

  return data;

}



void setup() {

  Serial.begin(115200);



  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {

    Serial.println("SSD1306 allocation failed");

    for (;;);

  }



  bootOLEDMessage("Milk Quality Tester");

  bootOLEDMessage("Initializing...");



  WiFi.begin(ssid, password);

  for (int i = 0; i < 10; i++) {

    if (WiFi.status() == WL_CONNECTED) {

      break;

    }

    bootOLEDMessage("Connecting to Wi-Fi...");

  }



  if (WiFi.status() == WL_CONNECTED) {

    bootOLEDMessage("IP: " + WiFi.localIP().toString());

  } else {

    bootOLEDMessage("Wi-Fi Offline");

  }



  sensors.begin();



  server.on("/", HTTP_GET, [](AsyncWebServerRequest* request) {

    request->send(200, "text/html", R"rawliteral(

<!DOCTYPE html>

<html lang="en">

<head>

  <meta charset="UTF-8">

  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <title>Milk Quality Tester</title>

  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">

  <style>

    body { background-color: #f8f9fa; }

    .card { margin-top: 20px; }

  </style>

</head>

<body>

  <div class="container">

    <div class="card">

      <div class="card-header">

        <h3 class="text-center">Milk Quality Tester</h3>

      </div>

      <div class="card-body">

        <p><strong>pH:</strong> <span id="ph">Loading...</span></p>

        <p><strong>Temperature:</strong> <span id="temperature">Loading...</span> °C</p>

        <p><strong>Gas:</strong> <span id="gas">Loading...</span></p>

        <p><strong>Status:</strong> <span id="status">Loading...</span></p>

      </div>

    </div>

  </div>

  <script>

    async function updateData() {

      const response = await fetch("/sensor");

      const data = await response.json();

      document.getElementById("ph").textContent = data.pH;

      document.getElementById("temperature").textContent = data.temperature;

      document.getElementById("gas").textContent = data.gas;

      document.getElementById("status").textContent = data.status;

    }

    setInterval(updateData, 2000);

  </script>

</body>

</html>

    )rawliteral");

  });



  server.on("/sensor", HTTP_GET, [](AsyncWebServerRequest* request) {

    request->send(200, "application/json", sensorData());

  });



  server.begin();

}



void loop() {

  readPH();

  readTemperature();

  readGas();

  checkMilkQuality();

  updateOLED();

}
