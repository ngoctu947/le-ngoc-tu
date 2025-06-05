```cpp
#include <ESP8266WiFi.h>
#include <Firebase_ESP_Client.h>
#include <Wire.h>
#include <BH1750.h>
#include <DHT.h>

// --- WiFi & Firebase ---
#define WIFI_SSID "YOUR_WIFI_SSID"
#define WIFI_PASSWORD "YOUR_WIFI_PASSWORD"
#define API_KEY "YOUR_FIREBASE_API_KEY"
#define DATABASE_URL "https://your-project-id.firebaseio.com/"  // Phải kết thúc bằng dấu "/"
#define USER_EMAIL "YOUR_EMAIL@gmail.com"
#define USER_PASSWORD "YOUR_PASSWORD"

// --- Cảm biến ---
#define DHTPIN D5
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);
BH1750 lightMeter;

// --- Firebase Object ---
FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

unsigned long lastMillis = 0;

// --- Động cơ ---
#define IN1 D1
#define IN2 D2

void setup() {
  Serial.begin(115200);

  // Cài đặt chân động cơ
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);

  // Kết nối WiFi
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("OK");

  // Khởi động Firebase
  config.api_key = API_KEY;
  config.database_url = DATABASE_URL;
  auth.user.email = USER_EMAIL;
  auth.user.password = USER_PASSWORD;

  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);

  // Khởi động cảm biến
  dht.begin();
  Wire.begin();
  lightMeter.begin();
}

void loop() {
  // Gửi dữ liệu cảm biến mỗi 5 giây
  if (millis() - lastMillis > 5000) {
    lastMillis = millis();
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    float lux = lightMeter.readLightLevel();

    if (!isnan(temp) && !isnan(hum)) {
      Firebase.RTDB.setFloat(&fbdo, "/sensor/temperature", temp);
      Firebase.RTDB.setFloat(&fbdo, "/sensor/humidity", hum);
    }
    Firebase.RTDB.setFloat(&fbdo, "/sensor/lux", lux);
    Serial.println("Đã gửi dữ liệu cảm biến");
  }

  // Đọc lệnh điều khiển từ Firebase
  if (Firebase.RTDB.getString(&fbdo, "/command/move")) {
    String command = fbdo.stringData();
    Serial.println("Lệnh từ Firebase: " + command);
    controlMotor(command);
  }
}

void controlMotor(String cmd) {
  if (cmd == "forward") {
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
  } else if (cmd == "backward") {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
  } else if (cmd == "stop") {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
  }
}
```