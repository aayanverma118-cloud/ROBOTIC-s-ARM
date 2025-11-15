#include <ESP8266WiFi.h>
#include <Servo.h>

const char* ssid = "Your Wifi Name";
const char* password = "Your Wifi Password";

WiFiServer server(80);

// Servos
Servo baseS, shoulderS, elbowS, wristS, gripS;

int baseA = 90, shoulderA = 90, elbowA = 90, wristA = 90, gripA = 0;

void setup() {
  Serial.begin(115200);

  baseS.attach(D1);
  shoulderS.attach(D2);
  elbowS.attach(D3);
  wristS.attach(D4);
  gripS.attach(D5);

  WiFi.begin(ssid, password);
  Serial.print("Connecting");

  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }

  Serial.println("\nConnected!");
  Serial.println(WiFi.localIP());
  server.begin();
}

void loop() {
  WiFiClient client = server.available();
  if (!client) return;

  String req = client.readStringUntil('\n');

  // ---- FAST AJAX servo control ----
  if (req.indexOf("/set?") >= 0) {
    baseA     = getVal(req, "b");
    shoulderA = getVal(req, "s");
    elbowA    = getVal(req, "e");
    wristA    = getVal(req, "w");
    gripA     = getVal(req, "g");

    baseS.write(baseA);
    shoulderS.write(shoulderA);
    elbowS.write(elbowA);
    wristS.write(wristA);
    gripS.write(gripA);

    client.println("HTTP/1.1 200 OK\n");
    client.stop();
    return;
  }

  // --- Send webpage ---
  client.println("HTTP/1.1 200 OK");
  client.println("Content-Type: text/html\n");
  client.println(pageHTML());
  client.stop();
}

int getVal(String r, String key) {
  int i = r.indexOf(key + "=");
  if (i < 0) return 0;
  int si = i + key.length() + 1;
  int ei = r.indexOf("&", si);
  if (ei < 0) ei = r.length();
  return r.substring(si, ei).toInt();
}

String pageHTML() {
  String p =
  "<html><head>"
  "<meta name='viewport' content='width=device-width, initial-scale=1.0'>"
  "<style>"
  "body{background:#111;color:#0f0;font-family:Arial;text-align:center;}"
  "h2{color:#0f0;}"
  ".slider{width:90%;margin:12px;}"
  "</style>"

  "<script>"
  "function send(){"
  "var url='/set?b='+base.value"
  "+'&s='+shoulder.value"
  "+'&e='+elbow.value"
  "+'&w='+wrist.value"
  "+'&g='+grip.value;"
  "fetch(url);"
  "}"
  "</script>"

  "</head><body>"
  "<h2>ESP8266 Robot Arm</h2>";

  p += slider("Base", "base", baseA, 25, 180);
  p += slider("Shoulder", "shoulder", shoulderA, 0, 180);
  p += slider("Elbow", "elbow", elbowA, 0, 180);
  p += slider("Wrist", "wrist", wristA, 0, 180);
  p += slider("Gripper", "grip", gripA, 0, 90);

  p += "</body></html>";
  return p;
}

String slider(String label, String id, int val, int minV, int maxV) {
  String h = "";
  h += "<h3>" + label + " (" + val + ")</h3>";
  h += "<input class='slider' type='range' id='" + id + 
       "' min='" + minV + "' max='" + maxV + "' value='" + val + 
       "' oninput='send()'>";
  return h;
}