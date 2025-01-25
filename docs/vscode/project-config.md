#include <WiFi.h>
#include <HTTPClient.h>
#include <TinyGPS++.h>
#include <SoftwareSerial.h>

// ThingSpeak settings
const String thingSpeakAPIKey = "YOUR_THINGSPEAK_API_KEY"; // Replace with your ThingSpeak API Key
const String thingSpeakURL = "https://api.thingspeak.com/update";

// Wi-Fi settings
const char* ssid = "YOUR_SSID"; 
const char* password = "YOUR_PASSWORD"; 

// GPS settings
SoftwareSerial ss(4, 3);  // RX, TX
TinyGPSPlus gps;

// Variables to store data
float latitude = 0.0;
float longitude = 0.0;
int batteryLevel = 100;  // Initial battery level
String accessLog = "Access Granted";  // Simulated access log
String alert = "No Alerts";  // Default alert

// Geofencing area (for example, room 101 with a center and radius)
float geofenceLatitude = 35.6895; 
float geofenceLongitude = 139.6917;
float geofenceRadius = 0.01;  // 0.01 degrees, approx 1km radius

void setup() {
  Serial.begin(115200);
  ss.begin(9600);  // GPS module baud rate
  
  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi!");
}

void loop() {
  // Reading GPS data
  while (ss.available() > 0) {
    gps.encode(ss.read());
    if (gps.location.isUpdated()) {
      latitude = gps.location.lat();
      longitude = gps.location.lng();
      checkGeofence();
    }
  }

  // Simulate battery level (decreases randomly)
  batteryLevel = random(20, 100);

  // Simulate access log (randomly grant or deny access)
  if (random(0, 2) == 0) {
    accessLog = "Access Granted";
  } else {
    accessLog = "Access Denied";
  }

  // Send data to ThingSpeak
  sendDataToThingSpeak();

  // Delay between updates
  delay(2000);
}

// Check if the bracelet is within the geofencing area
void checkGeofence() {
  float distance = calculateDistance(latitude, longitude, geofenceLatitude, geofenceLongitude);
  if (distance < geofenceRadius) {
    alert = "Inside Geofence: Hotel Area";
  } else {
    alert = "Outside Geofence: Hotel Area";
  }
}

// Calculate the distance between two GPS coordinates (Haversine formula)
float calculateDistance(float lat1, float lon1, float lat2, float lon2) {
  float radlat1 = deg2rad(lat1);
  float radlat2 = deg2rad(lat2);
  float theta = lon1 - lon2;
  float radtheta = deg2rad(theta);
  float dist = sin(radlat1) * sin(radlat2) + cos(radlat1) * cos(radlat2) * cos(radtheta);
  dist = acos(dist);
  dist = rad2deg(dist);
  dist = dist * 60 * 1.1515; // Convert to miles
  return dist * 1.609344; // Convert to kilometers
}

// Convert degrees to radians
float deg2rad(float deg) {
  return (deg * 3.141593 / 180.0);
}

// Convert radians to degrees
float rad2deg(float rad) {
  return (rad * 180.0 / 3.141593);
}

// Send data to ThingSpeak
void sendDataToThingSpeak() {
  HTTPClient http;
  String url = thingSpeakURL + "?api_key=" + thingSpeakAPIKey + "&field1=" + String(batteryLevel) + 
               "&field2=" + String(latitude, 6) + "&field3=" + String(longitude, 6) + 
               "&field4=" + accessLog + "&field5=" + alert;
  
  http.begin(url);  // Connect to ThingSpeak
  int httpCode = http.GET();  // Send GET request
  if (httpCode > 0) {
    Serial.println("Data sent successfully!");
  } else {
    Serial.println("Error sending data");
  }
  http.end();
}
