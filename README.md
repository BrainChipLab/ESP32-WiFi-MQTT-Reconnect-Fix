# ESP32 Keeps Disconnecting from WiFi & MQTT — Auto-Reconnect Code (Core 3.x)

Code snippets and a complete non-blocking template for diagnosing WiFi drops (reason codes & RSSI) and keeping both WiFi and MQTT (`PubSubClient`) connected 24/7 on `arduino-esp32` core 3.x.

---

### 1. Print the WiFi Disconnect Reason (`WiFi.onEvent`)
Put `onWiFiEvent` above `setup()` to see the exact reason code when your ESP32 drops:
* `200` (`BEACON_TIMEOUT`) — Weak signal / router stopped responding
* `201` (`NO_AP_FOUND`) — Wrong SSID, 5 GHz-only band, or out of range
* `202` (`AUTH_FAIL`) / `204` (`HANDSHAKE_TIMEOUT`) — Password/security mismatch or power dip during handshake
* `8` (`ASSOC_LEAVE`) — Router kicked the client or changed channel

```cpp
#include <WiFi.h>

void onWiFiEvent(WiFiEvent_t event, WiFiEventInfo_t info) {
  if (event == ARDUINO_EVENT_WIFI_STA_DISCONNECTED) {
    uint8_t r = info.wifi_sta_disconnected.reason;
    Serial.printf("WiFi lost. Reason %u: %s\n", r,
                  WiFi.disconnectReasonName((wifi_err_reason_t)r));
  } else if (event == ARDUINO_EVENT_WIFI_STA_GOT_IP) {
    Serial.print("Online, IP: ");
    Serial.println(WiFi.localIP());
  }
}

void setup() {
  Serial.begin(115200);
  WiFi.onEvent(onWiFiEvent);
  WiFi.mode(WIFI_STA);
  WiFi.setAutoReconnect(true);   // already the default on core 3.x
  WiFi.begin(ssid, pass);
  WiFi.setSleep(false);          // Fix 4: plugged-in boards only, no Bluetooth
}
```

---

### 2. Non-Blocking WiFi Safety Net in `loop()`
Checks connection status every 10 seconds using `millis()`, forces `WiFi.reconnect()` after 30 seconds offline, and restarts after 10 minutes as a last resort:

```cpp
unsigned long lastCheck = 0, offlineSince = 0;

void loop() {
  unsigned long now = millis();
  if (now - lastCheck >= 10000) {                 // every 10 s
    lastCheck = now;
    if (WiFi.status() == WL_CONNECTED) {
      offlineSince = 0;
      Serial.printf("RSSI: %d dBm\n", WiFi.RSSI());
    } else {
      if (offlineSince == 0) offlineSince = now;
      if (now - offlineSince > 30000)  WiFi.reconnect();  // 30 s offline
      if (now - offlineSince > 600000) ESP.restart();     // 10 min: last resort
    }
  }
  // ... your code here, no long delay() ...
}
```

---

### 3. Non-Blocking MQTT Reconnect + Unique Client ID (`PubSubClient`)
* Generates a unique Client ID from the MAC address so two ESP32 boards never kick each other off the broker.
* Retries every 5 seconds without blocking `loop()`, re-subscribes on reconnect, and calls `mqtt.loop()` continuously to prevent keep-alive timeouts.

```cpp
#include <PubSubClient.h>

WiFiClient net;
PubSubClient mqtt(net);
String clientId;                 // in setup(), after WiFi.begin():
                                 // clientId = "esp32-" + WiFi.macAddress();
                                 // clientId.replace(":", "");
                                 // mqtt.setServer("192.168.1.10", 1883);
unsigned long lastMqttTry = 0;

// Inside loop():
  if (WiFi.status() == WL_CONNECTED) {
    if (!mqtt.connected()) {
      if (now - lastMqttTry >= 5000) {            // try every 5 s, don't block
        lastMqttTry = now;
        if (mqtt.connect(clientId.c_str())) {
          mqtt.subscribe("home/esp32/cmd");       // subscribe again!
        } else {
          Serial.printf("MQTT failed, state %d\n", mqtt.state());
        }
      }
    } else {
      mqtt.loop();                                // keep-alive: call it often
    }
  }
// state(): -3 connection lost, -4 timeout, 4 bad user/password, 5 not authorized
```

---

### 4. Complete Combined Sketch (WiFi Safety Net + MQTT Auto-Reconnect)
Ready-to-upload template combining all snippets above:

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Your_2.4GHz_SSID";
const char* pass = "Your_Password";
const char* mqtt_server = "192.168.1.10";

WiFiClient net;
PubSubClient mqtt(net);
String clientId;

unsigned long lastCheck = 0, offlineSince = 0, lastMqttTry = 0;

void onWiFiEvent(WiFiEvent_t event, WiFiEventInfo_t info) {
  if (event == ARDUINO_EVENT_WIFI_STA_DISCONNECTED) {
    uint8_t r = info.wifi_sta_disconnected.reason;
    Serial.printf("WiFi lost. Reason %u: %s\n", r,
                  WiFi.disconnectReasonName((wifi_err_reason_t)r));
  } else if (event == ARDUINO_EVENT_WIFI_STA_GOT_IP) {
    Serial.print("Online, IP: ");
    Serial.println(WiFi.localIP());
  }
}

void setup() {
  Serial.begin(115200);
  WiFi.onEvent(onWiFiEvent);
  WiFi.mode(WIFI_STA);
  WiFi.setAutoReconnect(true);
  WiFi.begin(ssid, pass);
  WiFi.setSleep(false); // plugged-in boards only, no Bluetooth

  clientId = "esp32-" + WiFi.macAddress();
  clientId.replace(":", "");
  mqtt.setServer(mqtt_server, 1883);
}

void loop() {
  unsigned long now = millis();

  // WiFi safety net (every 10 s)
  if (now - lastCheck >= 10000) {
    lastCheck = now;
    if (WiFi.status() == WL_CONNECTED) {
      offlineSince = 0;
      Serial.printf("RSSI: %d dBm\n", WiFi.RSSI());
    } else {
      if (offlineSince == 0) offlineSince = now;
      if (now - offlineSince > 30000)  WiFi.reconnect();
      if (now - offlineSince > 600000) ESP.restart();
    }
  }

  // Non-blocking MQTT reconnect & keep-alive
  if (WiFi.status() == WL_CONNECTED) {
    if (!mqtt.connected()) {
      if (now - lastMqttTry >= 5000) {
        lastMqttTry = now;
        if (mqtt.connect(clientId.c_str())) {
          Serial.println("MQTT connected!");
          mqtt.subscribe("home/esp32/cmd");
        } else {
          Serial.printf("MQTT failed, state %d\n", mqtt.state());
        }
      }
    } else {
      mqtt.loop();
    }
  }
}
```
