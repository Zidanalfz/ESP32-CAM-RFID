// 📚 IMPORT LIBRARY - seperti "mengunduh alat-alat" yang dibutuhkan
#include <WebServer.h>    // Untuk membuat web server
#include <WiFi.h>         // Untuk koneksi WiFi
#include <SPI.h>          // Untuk komunikasi dengan RFID
#include <MFRC522.h>      // Untuk mengontrol modul RFID
#include <esp32cam.h>     // Untuk mengontrol kamera

// 📡 KONFIGURASI WiFi - seperti "alamat dan password WiFi"
const char* WIFI_SSID = "Public";      // Nama WiFi yang akan disambungi
const char* WIFI_PASS = "12345678";    // Password WiFi

// 🔌 SETUP PIN RFID - seperti "menentukan colokan kabel"
#define SS_PIN 13      // GPIO13 → Connected ke pin SDA RFID
#define RST_PIN 15     // GPIO15 → Connected ke pin RST RFID

/* 
 * 🎯 DIAGRAM KONEKSI ESP32-CAM ke RFID:
 * 3.3V  → 3.3V     (Daya positif)
 * GND   → GND      (Daya negatif/ground)
 * GPIO13 → SDA     (Data RFID)
 * GPIO15 → RST     (Reset RFID)
 * GPIO14 → SCK     (Clock sinyal)
 * GPIO2  → MOSI    (Data keluar ESP → RFID)
 * GPIO12 → MISO    (Data masuk RFID → ESP)
 */

// 🔧 BUAT OBJECT RFID - seperti "menyiapkan alat RFID"
MFRC522 mfrc522(SS_PIN, RST_PIN);

// 🌐 BUAT WEB SERVER - seperti "membuat restoran kecil di port 80"
WebServer server(80);

// 💾 VARIABLE PENYIMPANAN - seperti "buku catatan"
String lastRFID = "Tidak ada kartu terdeteksi";  // Menyimpan ID kartu terakhir

// ⚙️ FUNGSI SETUP - dijalankan SEKALI saat device nyala
void setup() {
  Serial.begin(115200);  // Mulai komunikasi serial untuk debugging
  
  // 🔄 INISIALISASI RFID
  SPI.begin(14, 12, 2, 13);  // Mulai komunikasi SPI dengan pin yang ditentukan
  mfrc522.PCD_Init();        // Hidupkan modul RFID
  delay(4);                  // Tunggu sebentar untuk stabil
  
  // 🖨️ CETAK INFO RFID ke Serial Monitor
  Serial.println("RFID Reader Initialized:");
  Serial.print("SS Pin: GPIO"); Serial.println(SS_PIN);
  Serial.print("RST Pin: GPIO"); Serial.println(RST_PIN);
  mfrc522.PCD_DumpVersionToSerial();  // Tampilkan versi firmware RFID
  
  // 📷 SETUP KAMERA
  setupCamera();
  
  // 📡 KONEKSI WiFi
  connectWiFi();
  
  // 🛣️ SETUP RUTE WEB SERVER
  setupWebServer();
  
  // ✅ SISTEM SIAP
  Serial.println("\n=== Sistem ESP32-CAM + RFID Ready ===");
  Serial.println("Tempelkan kartu RFID ke reader...");
}

// 📷 FUNGSI SETUP KAMERA
void setupCamera() {
  using namespace esp32cam;
  
  // ⚙️ KONFIGURASI KAMERA
  Config config;
  config.setPins(pins::AiThinker);          // Gunakan pin untuk board AiThinker
  config.setResolution(Resolution::find(800, 600));  // Set resolusi 800x600 pixel
  config.setBufferCount(2);                 // Sediakan 2 buffer untuk gambar
  config.setJpeg(80);                       // Kualitas JPEG 80%
  
  // 🚀 HIDUPKAN KAMERA
  bool ok = Camera.begin(config);
  if (!ok) {
    Serial.println("Gagal inisialisasi kamera");
    delay(5000);
    ESP.restart();  // Restart device jika kamera gagal
  }
  Serial.println("Kamera berhasil diinisialisasi");
}

// 📡 FUNGSI KONEKSI WiFi
void connectWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASS);  // Mulai koneksi ke WiFi
  Serial.print("Menghubungkan ke WiFi");
  
  // ⏳ TUNGGU SAMPAI TERHUBUNG (maksimal 20 detik)
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(1000);
    Serial.print(".");
    attempts++;
  }
  
  // ✅ CEK HASIL KONEKSI
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println();
    Serial.println("WiFi terhubung!");
    Serial.print("IP Address: ");
    Serial.println(WiFi.localIP());  // Tampilkan alamat IP device
  } else {
    Serial.println("\nGagal terhubung ke WiFi");
  }
}

// 🛣️ FUNGSI SETUP RUTE WEB SERVER
void setupWebServer() {
  // Tentukan "halaman-halaman" yang tersedia:
  server.on("/", handleRoot);           // Halaman utama
  server.on("/capture.jpg", handleCapture);  // Halaman untuk ambil foto
  server.on("/rfid", handleRFID);       // Halaman untuk cek RFID
  server.on("/stream", handleStream);   // Halaman untuk live stream
  server.on("/info", handleInfo);       // Halaman informasi system
  
  server.begin();  // Mulai web server
  Serial.println("Web server started");
}

// 🔄 FUNGSI LOOP - dijalankan BERULANG-ULANG selamanya
void loop() {
  server.handleClient();  // Terima kunjungan dari browser
  checkRFID();            // Cek apakah ada kartu RFID
  delay(500);             // Tunggu 0.5 detik sebelum ulangi
}

// 🔍 FUNGSI CEK RFID
void checkRFID() {
  // 👀 CEK APAKAH ADA KARTU BARU
  if (!mfrc522.PICC_IsNewCardPresent()) {
    return;  // Jika tidak ada kartu, keluar dari fungsi
  }
  
  // 📖 BACA ID KARTU
  if (!mfrc522.PICC_ReadCardSerial()) {
    return;  // Jika gagal baca, keluar dari fungsi
  }
  
  // 🔢 KONVERSI ID KARTU KE STRING
  String content = "";
  byte letter;
  for (byte i = 0; i < mfrc522.uid.size; i++) {
    if(mfrc522.uid.uidByte[i] < 0x10) {
      content += "0";  // Tambah "0" di depan jika angka < 16
    }
    content += String(mfrc522.uid.uidByte[i], HEX);  // Konversi ke hexadecimal
  }
  content.toUpperCase();  // Ubah ke huruf besar
  
  // 🖨️ TAMPILKAN ID KARTU
  Serial.print("RFID Terdeteksi: ");
  Serial.println(content);
  
  lastRFID = content;  // Simpan ID kartu terakhir
  
  // ⏹️ HENTIKAN PEMBACAAN
  mfrc522.PICC_HaltA();
}

// 🏠 FUNGSI HALAMAN UTAMA (HTML)
void handleRoot() {
  String html = R"rawliteral(
<!DOCTYPE html><html><head>
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>ESP32 CAM + RFID</title>
<style>
/* 🎨 STYLING HALAMAN WEB */
body { 
  font-family: Arial, sans-serif; 
  text-align: center; 
  margin: 20px; 
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
}
.container { 
  max-width: 800px; 
  margin: 0 auto; 
  background: white;
  padding: 30px;
  border-radius: 15px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.2);
}
/* ... (styling lainnya) ... */
</style>
</head>
<body>
<div class="container">
  <h1>🚀 ESP32 CAM + RFID System</h1>
  
  <div class="pin-info">
    <strong>Pin Configuration:</strong><br>
    RFID SDA → GPIO13 | RFID RST → GPIO15<br>
    RFID SCK → GPIO14 | RFID MOSI → GPIO2 | RFID MISO → GPIO12
  </div>
  
  <div class="rfid-info">
    <h3>🔍 Status RFID</h3>
    <div id="rfid">RFID Terakhir: )rawliteral" + lastRFID + R"rawliteral(</div>
  </div>
  
  <div class="button-group">
    <button onclick="captureImage()">📷 Ambil Foto</button>
    <button onclick="checkRFID()">🔄 Refresh RFID</button>
    <button onclick="startStream()">📹 Live Stream</button>
    <button onclick="showInfo()">ℹ️ System Info</button>
  </div>
  
  <div id="status">Status: Sistem Ready ✅</div>
  
  <div id="imageContainer">
    <img id="image" src="" style="display:none;">
  </div>
  
  <div class="stream-container" id="streamContainer" style="display:none;">
    <h3>Live Stream</h3>
    <img id="stream" src="" style="max-width: 100%;">
  </div>
</div>

<script>
// 📜 JAVASCRIPT UNTUK INTERAKSI BROWSER

function captureImage() {
  document.getElementById('status').innerHTML = 'Status: 📷 Mengambil foto...';
  document.getElementById('streamContainer').style.display = 'none';
  
  var img = document.getElementById('image');
  img.src = '/capture.jpg?' + new Date().getTime();  // Minta foto ke ESP32
  img.style.display = 'block';
  
  img.onload = function() {
    document.getElementById('status').innerHTML = 'Status: ✅ Foto berhasil diambil ' + new Date().toLocaleTimeString();
  }
}

function startStream() {
  document.getElementById('status').innerHTML = 'Status: 📹 Memulai live stream...';
  document.getElementById('image').style.display = 'none';
  
  var streamContainer = document.getElementById('streamContainer');
  var streamImg = document.getElementById('stream');
  
  streamContainer.style.display = 'block';
  streamImg.src = '/stream?' + new Date().getTime();  // Minta stream ke ESP32
}

function checkRFID() {
  document.getElementById('status').innerHTML = 'Status: 🔄 Memeriksa RFID...';
  
  fetch('/rfid')  // Minta data RFID terbaru ke ESP32
    .then(response => response.text())
    .then(data => {
      document.getElementById('rfid').innerHTML = 'RFID Terakhir: ' + data;
      document.getElementById('status').innerHTML = 'Status: ✅ RFID diperbarui ' + new Date().toLocaleTimeString();
    })
}

// 🔄 AUTO-REFRESH: RFID setiap 5 detik, stream setiap 2 detik
setInterval(checkRFID, 5000);
setInterval(function() {
  var streamContainer = document.getElementById('streamContainer');
  if (streamContainer.style.display !== 'none') {
    var streamImg = document.getElementById('stream');
    streamImg.src = '/stream?' + new Date().getTime();
  }
}, 2000);
</script>
</body></html>
)rawliteral";
  
  server.send(200, "text/html", html);  // Kirim halaman HTML ke browser
}

// 📸 FUNGSI AMBIL FOTO
void handleCapture() {
  auto frame = esp32cam::capture();  // Ambil gambar dari kamera
  if (frame == nullptr) {
    Serial.println("CAPTURE FAIL");
    server.send(503, "", "");  // Kirim error ke browser
    return;
  }
  
  // 🖨️ LOG FOTO YANG DIAMBIL
  Serial.printf("CAPTURE OK %dx%d %db\n", frame->getWidth(), frame->getHeight(), 
                static_cast<int>(frame->size()));
  
  // 📨 KIRIM GAMBAR KE BROWSER
  server.setContentLength(frame->size());
  server.send(200, "image/jpeg");
  WiFiClient client = server.client();
  frame->writeTo(client);  // Stream gambar ke browser
}

// 📹 FUNGSI LIVE STREAM
void handleStream() {
  auto frame = esp32cam::capture();  // Ambil frame dari kamera
  if (frame == nullptr) {
    Serial.println("STREAM FAIL");
    server.send(503, "", "");
    return;
  }
  
  // 📨 KIRIM FRAME KE BROWSER
  server.setContentLength(frame->size());
  server.send(200, "image/jpeg");
  WiFiClient client = server.client();
  frame->writeTo(client);
}

// 🔍 FUNGSI CEK STATUS RFID
void handleRFID() {
  server.send(200, "text/plain", lastRFID);  // Kirim ID RFID terakhir ke browser
}

// ℹ️ FUNGSI INFORMASI SISTEM
void handleInfo() {
  String info = "<html><head><title>System Info</title></head><body>";
  info += "<h1>ESP32-CAM + RFID System Info</h1>";
  info += "<h2>Pin Configuration:</h2>";
  info += "<table border='1' style='border-collapse: collapse;'>";
  info += "<tr><th>ESP32-CAM</th><th>RFID RC522</th><th>GPIO Pin</th></tr>";
  info += "<tr><td>3.3V</td><td>3.3V</td><td>-</td></tr>";
  info += "<tr><td>GND</td><td>GND</td><td>-</td></tr>";
  info += "<tr><td>GPIO13</td><td>SDA</td><td>13</td></tr>";
  // ... (tabel koneksi lengkap)
  info += "</table>";
  info += "<h2>Current Status:</h2>";
  info += "<p>Last RFID: " + lastRFID + "</p>";
  info += "<p>WiFi Status: " + String(WiFi.status() == WL_CONNECTED ? "Connected" : "Disconnected") + "</p>";
  info += "<p>IP Address: " + WiFi.localIP().toString() + "</p>";
  info += "<p><a href='/'>Kembali ke Home</a></p>";
  info += "</body></html>";
  
  server.send(200, "text/html", info);  // Kirim halaman info ke browser
}
