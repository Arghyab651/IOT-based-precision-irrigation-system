# IOT-based-precision-irrigation-system
#include <WiFi.h>
#include <HTTPClient.h>
#include <DallasTemperature.h>
#include <OneWire.h>
#include <DHT.h>

#define ONE_WIRE_BUS 4    
#define DHTPIN 5          
#define DHTTYPE DHT22     
#define MOISTURE_PIN 34   
#define RAIN_PIN 26       
#define RELAY_PIN 14      
#define ROTARY_PIN_1 35   
#define ROTARY_PIN_2 32   
#define ROTARY_PIN_3 33   
#define ROTARY_PIN_4 25   


OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);
DHT dht(DHTPIN, DHTTYPE);
volatile int rainCount = 0;  


const char* ssid = "YOUR_WIFI_SSID";      
const char* password = "YOUR_WIFI_PASSWORD"; 
const char* apiKey = "YOUR_OPENWEATHERMAP_API_KEY";
const char* city = "Mumbai";              
const char* countryCode = "IN";            
const int irrigationTime = 900000;        
unsigned long previousMillis = 0;
bool isIrrigating = false;

void IRAM_ATTR rainInterrupt() {
  rainCount++;  
}

void setup() {
  
  sensors.begin();
  dht.begin();
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  pinMode(RAIN_PIN, INPUT_PULLUP);
  pinMode(ROTARY_PIN_1, INPUT_PULLUP);
  pinMode(ROTARY_PIN_2, INPUT_PULLUP);
  pinMode(ROTARY_PIN_3, INPUT_PULLUP);
  pinMode(ROTARY_PIN_4, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(RAIN_PIN), rainInterrupt, FALLING);

  
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
}

void loop() {

  int moisture = analogRead(MOISTURE_PIN);
  sensors.requestTemperatures();
  float soilTemp = sensors.getTempCByIndex(0);
  float airTemp = dht.readTemperature();
  float humidity = dht.readHumidity();

 
  int threshold = 1500;  // Default
  if (digitalRead(ROTARY_PIN_1) == LOW) threshold = 1500; 
  else if (digitalRead(ROTARY_PIN_2) == LOW) threshold = 2000; 
  else if (digitalRead(ROTARY_PIN_3) == LOW) threshold = 2500;  
  else if (digitalRead(ROTARY_PIN_4) == LOW) threshold = 1000; 

  
  bool rainForecast = false;
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    String url = "http://api.openweathermap.org/data/2.5/weather?q=" + String(city) + "," + String(countryCode) + "&appid=" + String(apiKey) + "&units=metric";
    http.begin(url);
    int httpCode = http.GET();
    if (httpCode == 200) {
      String payload = http.getString();
      if (payload.indexOf("\"rain\":") != -1) {
        rainForecast = true; 
      }
    }
    http.end();
  }

  
  bool irrigate = (moisture > threshold && airTemp > 20 && humidity < 60 && rainCount == 0 && !rainForecast);
  if (irrigate && !isIrrigating) {
    digitalWrite(RELAY_PIN, HIGH); 
    isIrrigating = true;
    previousMillis = millis();
  }
  if (isIrrigating && (millis() - previousMillis >= irrigationTime)) {
    digitalWrite(RELAY_PIN, LOW);  
    isIrrigating = false;
    rainCount = 0;  
  }

  delay(5000); 
}
