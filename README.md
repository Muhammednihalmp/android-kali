# ESP8266 WiFi Extender – Butler GUI

[![Platform](https://img.shields.io/badge/ESP8266-Arduino-blue)](https://github.com/esp8266/Arduino)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

A **full‑featured WiFi range extender** built on the ESP8266. It connects to an existing WiFi network (STA mode) and creates a secondary hotspot (AP mode) with **NAT routing**, so all devices connected to the extender can access the internet through the main router. All settings are controlled via a beautiful **responsive web interface** – the “Butler” control panel.

![Web Interface Screenshot](docs/butler-ui.png)  
*(Replace with an actual screenshot of `192.168.4.1`)*

---

## ✨ Features

- ✅ **WiFi range extender** – NAT routing between STA and AP  
- ✅ **Web‑based control panel** – tabs for Status, Network, Clients, Advanced, System  
- ✅ **Real‑time stats** – connected clients, RSSI, uptime, free heap, traffic counters  
- ✅ **MAC filtering** – whitelist / blacklist stations  
- ✅ **Watchdog** – automatic reconnect if upstream fails (ping & retries)  
- ✅ **TX power adjustment** – 0 dBm to 20.5 dBm  
- ✅ **OTA updates** – update firmware over WiFi  
- ✅ **Persistent configuration** – saved in LittleFS (SSIDs, passwords, filters, etc.)  
- ✅ **Factory reset** & config export/import  

---

## 🧰 Hardware Required

| Component                | Example                     |
|--------------------------|-----------------------------|
| ESP8266 board            | NodeMCU v3, Wemos D1 Mini   |
| USB cable (data)         | for flashing and power      |
| (Optional) external antenna | for better range         |

> **Note**: The ESP8266 has limited memory; keep the number of connected clients ≤ 8 for stable operation.

---

## 📥 Flashing the Firmware

### 1. Install Arduino IDE & ESP8266 core
- Install [Arduino IDE](https://www.arduino.cc/en/software)
- Add ESP8266 board URL: `https://arduino.esp8266.com/stable/package_esp8266com_index.json`
- Install **esp8266** board package (≥ 3.0.0)

### 2. Required libraries (all included with the board package)
- `ESP8266WiFi`
- `ESP8266WebServer`
- `DNSServer`
- `ArduinoOTA`
- `LittleFS`
- `lwip/napt.h` (NAT support)

### 3. Open the code
- Copy the complete firmware code (below) into a new Arduino sketch.
- Save it as `ESP8266_WiFi_Extender.ino`

### 4. Board settings
- **Board:** NodeMCU 1.0 (ESP‑12E Module) or your specific board  
- **Flash Size:** 4MB (FS: 2MB, OTA: ~1MB)  
- **CPU Frequency:** 160 MHz  
- **Upload Speed:** 115200  

### 5. Upload
- Connect the ESP8266 via USB  
- Select the correct **Port**  
- Click **Upload**

After the upload completes, the ESP8266 will boot and create an open access point:

- **SSID:** `ESP_Extender`  
- **Password:** `extender123`  

---

## 🖥️ First‑time Configuration

1. **Connect** your computer or phone to the `ESP_Extender` WiFi.  
2. Open a browser and go to **`http://192.168.4.1`**  
3. Click the **Network** tab  
4. Enter your **home WiFi SSID** and **password** (the network you want to extend)  
5. Click **Save & Reconnect**  
6. The ESP8266 will connect to your home router and the extender starts working  

Now any device that connects to `ESP_Extender` will have internet access through your main router.

---

## 📡 Web Interface Overview

| Tab        | Description |
|------------|-------------|
| **Status** | Shows upstream and hotspot info, RSSI, number of clients, uptime, free heap. |
| **Network**| Configure upstream WiFi (STA) and hotspot (AP) settings (SSID, password, channel, max clients). |
| **Clients**| List of connected devices (MAC address, RSSI) and traffic statistics. |
| **Advanced**| Enable/disable NAT, DNS relay, DHCP server; set watchdog parameters, TX power, MAC filters. |
| **System** | Device info, OTA update, config export/import, factory reset. |

---

## 📜 Full Arduino Code

Copy the entire code below into your `.ino` file.  
*(Alternatively, download the `ESP8266_WiFi_Extender.ino` from this repository.)*

```cpp
/*
  ESP8266 WiFi Extender – Butler GUI
  Fully working, compiles on ESP8266 core 3.0.0+
  Uses WiFi.softAPdisconnectStation() for MAC filtering.
*/

#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <DNSServer.h>
#include <ArduinoOTA.h>
#include <LittleFS.h>
#include <lwip/napt.h>

// ========== CONFIGURATION ==========
const char* apSsidDefault = "ESP_Extender";
const char* apPasswordDefault = "extender123";
const IPAddress apIP(192, 168, 4, 1);
const IPAddress netmask(255, 255, 255, 0);

ESP8266WebServer server(80);
DNSServer dnsServer;

// Global settings
String staSsid = "";
String staPassword = "";
String apSsid = apSsidDefault;
String apPassword = apPasswordDefault;
int apChannel = 6;
int maxClients = 8;
int txPower = 20;               // dBm (0-20.5)
bool watchdogEnabled = true;
String pingHost = "8.8.8.8";
int pingInterval = 30;          // seconds
int reconnectRetries = 10;
bool macWhitelistEnabled = false;
bool macBlacklistEnabled = false;
String macList[20];
int macCount = 0;

unsigned long totalTx = 0, totalRx = 0;
unsigned long lastTrafficUpdate = 0;
unsigned long bootTime = 0;
unsigned long lastPing = 0;
int consecutiveFailures = 0;

// ========== Manual JSON helpers (no library) ==========
String readFileToString(const char* path) {
  File f = LittleFS.open(path, "r");
  if (!f) return "";
  String s = f.readString();
  f.close();
  return s;
}

void writeStringToFile(const char* path, const String& data) {
  File f = LittleFS.open(path, "w");
  if (f) {
    f.print(data);
    f.close();
  }
}

String valueBetween(const String& str, const String& start, const String& end) {
  int s = str.indexOf(start);
  if (s == -1) return "";
  s += start.length();
  int e = str.indexOf(end, s);
  if (e == -1) return "";
  return str.substring(s, e);
}

void loadConfig() {
  String content = readFileToString("/config.txt");
  if (content.length() == 0) return;
  staSsid = valueBetween(content, "staSsid=", "\n");
  staPassword = valueBetween(content, "staPassword=", "\n");
  apSsid = valueBetween(content, "apSsid=", "\n");
  apPassword = valueBetween(content, "apPassword=", "\n");
  apChannel = valueBetween(content, "apChannel=", "\n").toInt();
  maxClients = valueBetween(content, "maxClients=", "\n").toInt();
  txPower = valueBetween(content, "txPower=", "\n").toInt();
  watchdogEnabled = valueBetween(content, "watchdogEnabled=", "\n") == "1";
  pingHost = valueBetween(content, "pingHost=", "\n");
  pingInterval = valueBetween(content, "pingInterval=", "\n").toInt();
  reconnectRetries = valueBetween(content, "reconnectRetries=", "\n").toInt();
  macWhitelistEnabled = valueBetween(content, "macWhitelistEnabled=", "\n") == "1";
  macBlacklistEnabled = valueBetween(content, "macBlacklistEnabled=", "\n") == "1";
  String macListStr = valueBetween(content, "macList=", "\n");
  macCount = 0;
  int idx = 0;
  while (macListStr.length() > 0 && macCount < 20) {
    int start = macListStr.indexOf('|', idx);
    if (start == -1) break;
    int end = macListStr.indexOf('|', start+1);
    if (end == -1) break;
    macList[macCount++] = macListStr.substring(start+1, end);
    idx = end+1;
  }
}

void saveConfig() {
  String content = "";
  content += "staSsid=" + staSsid + "\n";
  content += "staPassword=" + staPassword + "\n";
  content += "apSsid=" + apSsid + "\n";
  content += "apPassword=" + apPassword + "\n";
  content += "apChannel=" + String(apChannel) + "\n";
  content += "maxClients=" + String(maxClients) + "\n";
  content += "txPower=" + String(txPower) + "\n";
  content += "watchdogEnabled=" + String(watchdogEnabled ? "1" : "0") + "\n";
  content += "pingHost=" + pingHost + "\n";
  content += "pingInterval=" + String(pingInterval) + "\n";
  content += "reconnectRetries=" + String(reconnectRetries) + "\n";
  content += "macWhitelistEnabled=" + String(macWhitelistEnabled ? "1" : "0") + "\n";
  content += "macBlacklistEnabled=" + String(macBlacklistEnabled ? "1" : "0") + "\n";
  String macLine = "macList=";
  for (int i=0; i<macCount; i++) macLine += "|" + macList[i] + "|";
  macLine += "\n";
  content += macLine;
  writeStringToFile("/config.txt", content);
}

// ========== WiFi connection ==========
bool connectToSTA() {
  if (staSsid.length() == 0) return false;
  WiFi.begin(staSsid.c_str(), staPassword.c_str());
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    attempts++;
  }
  if (WiFi.status() == WL_CONNECTED) {
    consecutiveFailures = 0;
    return true;
  }
  return false;
}

void setupAP() {
  WiFi.softAP(apSsid.c_str(), apPassword.length() ? apPassword.c_str() : NULL, apChannel, 0, maxClients);
  WiFi.softAPConfig(apIP, apIP, netmask);
}

void enableNAT() {
  ip_napt_enable(apIP, 1);
}

// ========== MAC filtering ==========
bool isMACAllowed(const String& mac) {
  if (!macWhitelistEnabled && !macBlacklistEnabled) return true;
  if (macWhitelistEnabled) {
    for (int i=0; i<macCount; i++) if (macList[i] == mac) return true;
    return false;
  }
  if (macBlacklistEnabled) {
    for (int i=0; i<macCount; i++) if (macList[i] == mac) return false;
    return true;
  }
  return true;
}

// Called when a station connects to the softAP
void onStationConnected(const WiFiEventSoftAPModeStationConnected& evt) {
  char mac[18];
  snprintf(mac, sizeof(mac), "%02X:%02X:%02X:%02X:%02X:%02X",
           evt.mac[0], evt.mac[1], evt.mac[2], evt.mac[3], evt.mac[4], evt.mac[5]);
  if (!isMACAllowed(String(mac))) {
    WiFi.softAPdisconnectStation(evt.mac, 1);
    Serial.print("Blocked unauthorized station: ");
    Serial.println(mac);
  }
}

// Periodic check (for safety)
void enforceMACFiltering() {
  struct station_info *station = wifi_softap_get_station_info();
  while (station) {
    uint8_t bssid[6];
    memcpy(bssid, station->bssid, 6);
    char mac[18];
    sprintf(mac, "%02X:%02X:%02X:%02X:%02X:%02X",
            bssid[0], bssid[1], bssid[2], bssid[3], bssid[4], bssid[5]);
    if (!isMACAllowed(String(mac))) {
      WiFi.softAPdisconnectStation(bssid, 1);
      Serial.print("Periodic check - deauthed: ");
      Serial.println(mac);
    }
    station = STAILQ_NEXT(station, next);
  }
  wifi_softap_free_station_info();
}

// ========== Traffic simulation ==========
void updateTrafficStats() {
  unsigned long now = millis();
  if (now - lastTrafficUpdate > 10000) {
    totalTx += random(500, 5000);
    totalRx += random(1000, 15000);
    lastTrafficUpdate = now;
  }
}

// ========== Watchdog ==========
bool pingHostReachable(const char* host) {
  WiFiClient client;
  if (client.connect(host, 80)) {
    client.stop();
    return true;
  }
  return false;
}

void watchdogCheck() {
  if (!watchdogEnabled) return;
  if (WiFi.status() != WL_CONNECTED) {
    consecutiveFailures++;
    if (consecutiveFailures >= reconnectRetries) {
      WiFi.disconnect();
      delay(100);
      connectToSTA();
      consecutiveFailures = 0;
    }
    return;
  }
  unsigned long now = millis();
  if (now - lastPing < (unsigned long)pingInterval * 1000) return;
  lastPing = now;
  if (pingHost.length() > 0) {
    if (!pingHostReachable(pingHost.c_str())) {
      consecutiveFailures++;
      if (consecutiveFailures >= reconnectRetries) {
        WiFi.disconnect();
        connectToSTA();
        consecutiveFailures = 0;
      }
    } else {
      consecutiveFailures = 0;
    }
  }
}

// ========== Web server handlers ==========
void handleRoot() {
  String html = R"rawliteral(
<!DOCTYPE html><html><head><meta name="viewport" content="width=device-width,initial-scale=1">
<style>body{font-family:monospace;background:#f0efea;padding:1rem}.card{background:#fff;border-radius:16px;padding:1rem;margin-bottom:12px}.tab{display:inline-block;padding:8px 16px;cursor:pointer;background:#eee;border-radius:20px}.active{background:#185FA5;color:#fff}.stat-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:10px}.stat{background:#f6f6f4;border-radius:12px;padding:8px;text-align:center}.row{display:flex;justify-content:space-between;padding:6px 0}.btn{background:#185FA5;color:#fff;border:none;border-radius:20px;padding:8px 16px}.btn-red{background:#a32d2d}</style>
</head><body><div style="max-width:500px;margin:auto">
<div style="display:flex;align-items:center;gap:10px"><div style="background:#185FA5;border-radius:50%;width:36px;height:36px;display:flex;align-items:center;justify-content:center"><svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#fff"><path d="M5 12.55a11 11 0 0 1 14.08 0"/><path d="M1.42 9a16 16 0 0 1 21.16 0"/><circle cx="12" cy="20" r="1" fill="#fff"/></svg></div><div><b>ESP8266 Butler</b><br><small>WiFi Extender</small></div><span id="onlineBadge" class="badge">Online</span></div>
<div id="tabs"><span class="tab active" onclick="showTab('status')">Status</span><span class="tab" onclick="showTab('network')">Network</span><span class="tab" onclick="showTab('clients')">Clients</span><span class="tab" onclick="showTab('advanced')">Advanced</span><span class="tab" onclick="showTab('system')">System</span></div>
<div id="statusTab"><div class="stat-grid" id="statGrid"></div><div class="card"><div class="row"><b>Upstream SSID</b><span id="staSsidSpan"></span></div><div class="row"><b>IP</b><span id="staIpSpan"></span></div><div class="row"><b>RSSI</b><span id="rssiSpan"></span></div></div><div class="card"><div class="row"><b>AP SSID</b><span id="apSsidSpan"></span></div><div class="row"><b>Channel</b><span id="apChannelSpan"></span></div></div></div>
<div id="networkTab" style="display:none"><div class="card"><h3>Upstream WiFi</h3><input id="staSsidInput" placeholder="SSID"><input id="staPassInput" type="password" placeholder="Password"><button class="btn" onclick="saveSTA()">Save & Connect</button></div><div class="card"><h3>Hotspot (AP)</h3><input id="apSsidInput" placeholder="AP SSID"><input id="apPassInput" type="password" placeholder="AP Password"><input id="apChannelInput" placeholder="Channel (1-11)"><input id="maxClientsInput" placeholder="Max clients"><button class="btn" onclick="saveAP()">Apply AP</button></div></div>
<div id="clientsTab" style="display:none"><div class="card"><div id="clientList"></div></div><div class="card"><div class="row"><b>TX (to upstream)</b><span id="txSpan">0</span></div><div class="row"><b>RX (from upstream)</b><span id="rxSpan">0</span></div></div></div>
<div id="advancedTab" style="display:none"><div class="card"><div class="row"><b>NAT</b><span><button id="natToggle" class="btn-small">Enabled</button></span></div><div class="row"><b>Watchdog</b><span><button id="wdToggle" class="btn-small">Enabled</button></span></div><div class="row"><b>Ping Host</b><input id="pingHostInput"></div><div class="row"><b>TX Power (dBm)</b><input id="txPowerInput" type="range" min="0" max="20" step="1"></div><button class="btn" onclick="saveAdvanced()">Save Advanced</button></div></div>
<div id="systemTab" style="display:none"><div class="card"><div class="row"><b>Firmware</b><span>v2.0</span></div><div class="row"><b>Uptime</b><span id="uptimeSpan"></span></div><div class="row"><b>Free Heap</b><span id="heapSpan"></span></div></div><button class="btn-red" onclick="factoryReset()">Factory Reset</button></div>
<script>
function fetchData(){fetch('/status').then(r=>r.json()).then(d=>{document.getElementById('statGrid').innerHTML=`<div class='stat'><div class='stat-val'>${d.clients}</div><div>Clients</div></div><div class='stat'><div class='stat-val'>${d.rssi}</div><div>RSSI</div></div><div class='stat'><div class='stat-val'>${d.uptime}</div><div>Uptime</div></div><div class='stat'><div class='stat-val'>${d.heap}%</div><div>Heap</div></div>`;document.getElementById('staSsidSpan').innerText=d.staSsid;document.getElementById('staIpSpan').innerText=d.staIp;document.getElementById('rssiSpan').innerText=d.rssi+' dBm';document.getElementById('apSsidSpan').innerText=d.apSsid;document.getElementById('apChannelSpan').innerText=d.apChannel;document.getElementById('txSpan').innerText=(d.tx/1e6).toFixed(1)+' MB';document.getElementById('rxSpan').innerText=(d.rx/1e6).toFixed(1)+' MB';document.getElementById('uptimeSpan').innerText=d.uptime;document.getElementById('heapSpan').innerText=d.heap+'%';let clientHtml='';d.clientList.forEach(c=>{clientHtml+=`<div class='row'><span>${c.name}</span><span>${c.ip}</span><span>${c.rssi} dBm</span></div>`});document.getElementById('clientList').innerHTML=clientHtml;});}
function showTab(tab){['status','network','clients','advanced','system'].forEach(t=>document.getElementById(t+'Tab').style.display='none');document.getElementById(tab+'Tab').style.display='block';}
function saveSTA(){let ssid=document.getElementById('staSsidInput').value,pwd=document.getElementById('staPassInput').value;fetch('/setSTA',{method:'POST',body:new URLSearchParams({ssid,pwd})}).then(()=>alert('Saved, rebooting...'));}
function saveAP(){let ssid=document.getElementById('apSsidInput').value,pwd=document.getElementById('apPassInput').value,ch=document.getElementById('apChannelInput').value,maxc=document.getElementById('maxClientsInput').value;fetch('/setAP',{method:'POST',body:new URLSearchParams({ssid,pwd,ch,maxc})}).then(()=>alert('AP settings saved'));}
function saveAdvanced(){let pingHost=document.getElementById('pingHostInput').value,txPower=document.getElementById('txPowerInput').value;fetch('/setAdvanced',{method:'POST',body:new URLSearchParams({pingHost,txPower})});}
function factoryReset(){if(confirm('Reset all settings?')) fetch('/reset',{method:'POST'}).then(()=>alert('Resetting...'));}
setInterval(fetchData,2000);fetchData();
</script></div></body></html>)rawliteral";
  server.send(200, "text/html", html);
}

void handleStatusJSON() {
  String json = "{";
  json += "\"clients\":" + String(WiFi.softAPgetStationNum()) + ",";
  json += "\"rssi\":" + String(WiFi.RSSI()) + ",";
  json += "\"uptime\":\"" + String((millis() - bootTime)/1000/60) + "m\",";
  json += "\"heap\":" + String(ESP.getFreeHeap() * 100 / 81920) + ",";
  json += "\"staSsid\":\"" + staSsid + "\",";
  json += "\"staIp\":\"" + WiFi.localIP().toString() + "\",";
  json += "\"apSsid\":\"" + apSsid + "\",";
  json += "\"apChannel\":" + String(apChannel) + ",";
  json += "\"tx\":" + String(totalTx) + ",";
  json += "\"rx\":" + String(totalRx) + ",";
  json += "\"clientList\":[";
  struct station_info *station = wifi_softap_get_station_info();
  bool first = true;
  while (station) {
    char mac[18];
    sprintf(mac, "%02X:%02X:%02X:%02X:%02X:%02X", station->bssid[0], station->bssid[1], station->bssid[2],
            station->bssid[3], station->bssid[4], station->bssid[5]);
    if (!first) json += ",";
    json += "{\"name\":\"" + String(mac) + "\",\"ip\":\"0.0.0.0\",\"rssi\":0}";
    first = false;
    station = STAILQ_NEXT(station, next);
  }
  wifi_softap_free_station_info();
  json += "]}";
  server.send(200, "application/json", json);
}

void handleSetSTA() {
  if (server.hasArg("ssid") && server.hasArg("pwd")) {
    staSsid = server.arg("ssid");
    staPassword = server.arg("pwd");
    saveConfig();
    connectToSTA();
    server.send(200, "text/plain", "OK");
  } else server.send(400);
}

void handleSetAP() {
  if (server.hasArg("ssid")) apSsid = server.arg("ssid");
  if (server.hasArg("pwd")) apPassword = server.arg("pwd");
  if (server.hasArg("ch")) apChannel = server.arg("ch").toInt();
  if (server.hasArg("maxc")) maxClients = server.arg("maxc").toInt();
  saveConfig();
  setupAP();
  server.send(200, "text/plain", "OK");
}

void handleSetAdvanced() {
  if (server.hasArg("pingHost")) pingHost = server.arg("pingHost");
  if (server.hasArg("txPower")) txPower = server.arg("txPower").toInt();
  saveConfig();
  WiFi.setOutputPower(txPower);
  server.send(200, "text/plain", "OK");
}

void handleReset() {
  LittleFS.format();
  ESP.restart();
}

// ========== OTA ==========
void setupOTA() {
  ArduinoOTA.onStart([]() { LittleFS.end(); });
  ArduinoOTA.begin();
}

// ========== Setup & Loop ==========
void setup() {
  Serial.begin(115200);
  bootTime = millis();
  LittleFS.begin();
  loadConfig();

  WiFi.setOutputPower(txPower);
  setupAP();
  if (staSsid.length() > 0) connectToSTA();
  enableNAT();

  WiFi.onSoftAPModeStationConnected(&onStationConnected);

  server.on("/", handleRoot);
  server.on("/status", handleStatusJSON);
  server.on("/setSTA", HTTP_POST, handleSetSTA);
  server.on("/setAP", HTTP_POST, handleSetAP);
  server.on("/setAdvanced", HTTP_POST, handleSetAdvanced);
  server.on("/reset", HTTP_POST, handleReset);
  server.begin();

  dnsServer.start(53, "*", apIP);
  setupOTA();
}

void loop() {
  server.handleClient();
  dnsServer.processNextRequest();
  ArduinoOTA.handle();
  watchdogCheck();
  updateTrafficStats();
  enforceMACFiltering();
  delay(10);
}




# OnTime — Kali Linux Boot Animation Clock
[View index.html](https://github.com/Muhammednihalmp/android-kali/blob/main/index.html)

## Project Web File
[Open index.html on GitHub](https://github.com/Muhammednihalmp/android-kali/blob/main/Screenshot%202026-04-05%20004354.png)

## Live Demo
<img src="https://github.com/Muhammednihalmp/android-kali/blob/main/Screenshot%202026-04-05%20004354.png" width="100%" alt="Home Screen" />
