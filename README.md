#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>

// ===== Ultrasonic =====
#define TRIG_PIN D1   // GPIO5
#define ECHO_PIN D2   // GPIO4

#define TANK_HEIGHT 35.0     // ความสูงถัง (cm)
#define SENSOR_OFFSET 0.0   // ระยะ offset (cm)

ESP8266WebServer server(80);

// ---------- อ่าน Ultrasonic ----------
long readUltrasonic() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // timeout 30ms
  if (duration == 0) return -1;

  return duration * 0.0343 / 2;
}

// ---------- หน้าเว็บ ----------
void handleRoot() {
  long distance = readUltrasonic();
  String page = "<!DOCTYPE html><html><head>";
  page += "<meta name='viewport' content='width=device-width, initial-scale=1'>";
  page += "<meta http-equiv='refresh' content='1'>"; // รีเฟรชทุก 1 วิ
  page += "<title>Food Level</title>";
  page += "<style>";
  page += "body{font-family:Arial;text-align:center;background:#f4f4f4;}";
  page += ".card{background:white;padding:20px;margin:20px;border-radius:10px;}";
  page += ".big{font-size:40px;}";
  page += "</style></head><body>";

  page += "<h2>Food Level Monitor</h2>";
  page += "<div class='card'>";

  if (distance < 0) {
    page += "<p class='big'>Sensor Error</p>";
  } else {
    float fullDist  = SENSOR_OFFSET;
    float emptyDist = SENSOR_OFFSET + TANK_HEIGHT;
    distance = constrain(distance, fullDist, emptyDist);

    int foodPercent =
      (emptyDist - distance) * 100 /
      (emptyDist - fullDist);


    page += "<p>Food Remaining</p>";
    page += "<p class='big'>" + String(foodPercent) + " %</p>";
  }

  page += "</div></body></html>";

  server.send(200, "text/html", page);
}

void setup() {
  Serial.begin(115200);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  digitalWrite(TRIG_PIN, LOW);

  // ===== เปิด WiFi เอง =====
  WiFi.softAP("FoodMonitor");  // ชื่อ WiFi
  Serial.println("WiFi started");
  Serial.print("IP: ");
  Serial.println(WiFi.softAPIP()); // ปกติคือ 192.168.4.1

  // ===== เว็บเซิร์ฟเวอร์ =====
  server.on("/", handleRoot);
  server.begin();
}

void loop() {
  server.handleClient();
}
