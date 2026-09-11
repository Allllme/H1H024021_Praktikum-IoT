# Modul 2 – Konfigurasi Jaringan ESP32 (WiFi: Station, Access Point, AP+STA)

| | |
|---|---|
| Nama Kuliah (Kode) | Internet Of things (TK245005) |
| Tahun / Semester | 2024 / 5 |
| Modul | 1 |
| Nama Praktikan / NIM | Alma Maida Wirastuti / H1H024021 |

---

## Deskripsi Singkat Praktikum

Praktikum ini bertujuan memahami konfigurasi jaringan nirkabel (WiFi) pada ESP32/ESP8266 untuk aplikasi IoT, meliputi:

- Mode **Station (STA)**: ESP berperan sebagai klien yang terhubung ke jaringan WiFi yang sudah ada.
- Mode **Access Point (AP)**: ESP berperan sebagai penyedia jaringan (hotspot) mandiri tanpa memerlukan router eksternal.
- Mode **AP+STA**: gabungan keduanya, ESP dapat terhubung ke WiFi rumah sekaligus menyediakan Access Point sendiri.
- Membaca dan menganalisis parameter jaringan seperti **IP Address**, **MAC Address**, dan **RSSI (kekuatan sinyal)**.

Board yang digunakan pada praktikum ini adalah **NodeMCU (ESP8266)**, sehingga library yang dipakai adalah `ESP8266WiFi.h`. Untuk board **ESP32**, cukup ganti menjadi `#include <WiFi.h>` — seluruh fungsi (`WiFi.begin()`, `WiFi.softAP()`, dll.) tetap sama karena API-nya identik.

---

## Library / Dependencies

| Library | Board | Keterangan |
|---|---|---|
| `ESP8266WiFi.h` | ESP8266 (NodeMCU) | Menyediakan seluruh fungsi manajemen WiFi (STA, AP, status koneksi, dsb.) |
| `WiFi.h` | ESP32 | Versi setara `ESP8266WiFi.h` untuk board ESP32 |

Tidak ada library pihak ketiga tambahan yang perlu di-install, keduanya sudah termasuk dalam **Board Manager** (ESP8266/ESP32) di Arduino IDE.

**Alat dan bahan:**
- Board ESP32 DevKit / NodeMCU (ESP8266)
- Kabel USB (Micro-USB/USB-C sesuai board)
- Laptop/PC dengan Arduino IDE (sudah terpasang board manager ESP32/ESP8266)
- Jaringan WiFi (router/hotspot) beserta SSID dan password
- Smartphone/laptop untuk menguji koneksi ke Access Point
- LED (1 buah) dan resistor 220 Ω (opsional, indikator status koneksi)

---

## Skematik / Diagram Rangkaian

```
        ESP32 / NodeMCU
        ┌───────────────┐
        │           GPIO2├──[Resistor 220Ω]──►|── LED ──┐
        │               │                    (anoda) (katoda)
        │           GND ├────────────────────────────────┘
        └───────────────┘
```

**Keterangan rangkaian:**
- LED indikator dihubungkan ke pin GPIO 2.
- Resistor 220 Ω digunakan untuk membatasi arus yang mengalir ke LED.
- Katoda LED terhubung ke pin GND.
- LED berfungsi sebagai indikator visual status koneksi WiFi (menyala saat terhubung).

---

## Percobaan 2A — Mode Station (STA)

**Tujuan**: Menghubungkan ESP ke jaringan WiFi yang sudah tersedia (mode klien).

```cpp
#include <ESP8266WiFi.h>

const char* ssid     = "naye";
const char* password = "woylahbroo";
const int ledPin = 4; // LED indikator status koneksi

void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, LOW);

  // Set mode WiFi menjadi Station
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  Serial.print("Menghubungkan ke WiFi:");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  // Jika berhasil terhubung
  Serial.println();
  Serial.println("WiFi berhasil terhubung!");
  Serial.print("IP Address : ");
  Serial.println(WiFi.localIP());
  Serial.print("MAC Address : ");
  Serial.println(WiFi.macAddress());
  Serial.print("RSSI (dBm) : ");
  Serial.println(WiFi.RSSI());
  digitalWrite(ledPin, HIGH); // nyalakan LED sebagai indikator
}

void loop() {
  // Cek status koneksi setiap 5 detik
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("Status: Terhubung");
  } else {
    Serial.println("Status: Terputus");
    digitalWrite(ledPin, LOW);
  }
  delay(5000);
}
```

### Penjelasan Fungsi

| Fungsi | Penjelasan |
|---|---|
| `WiFi.mode(WIFI_STA)` | Mengatur mode operasi WiFi ESP menjadi Station, yaitu sebagai klien yang mencari dan terhubung ke access point yang sudah ada. |
| `WiFi.begin(ssid, password)` | Memulai proses koneksi ke jaringan WiFi dengan SSID dan password yang ditentukan. |
| `WiFi.status()` | Mengembalikan status koneksi saat ini, contoh nilai `WL_CONNECTED` jika sudah terhubung. |
| `WiFi.localIP()` | Mengembalikan alamat IP yang diberikan oleh DHCP server kepada ESP. |
| `WiFi.macAddress()` | Mengembalikan alamat MAC unik dari modul ESP. |
| `WiFi.RSSI()` | Mengembalikan nilai kekuatan sinyal WiFi (RSSI) dalam satuan dBm. |
| `void loop()` | Fungsi yang berjalan berulang untuk memonitor status koneksi setiap 5 detik. |

### Penjelasan Percabangan / Conditional

- **`while (WiFi.status() != WL_CONNECTED)`**: loop ini akan terus berjalan (blocking) selama ESP belum berhasil terhubung ke WiFi. Setiap 500 ms, program mencetak tanda titik (`.`) sebagai indikator proses. Loop ini tidak memiliki timeout, sehingga jika SSID/password salah, program akan terjebak selamanya di loop ini.
- **`if (WiFi.status() == WL_CONNECTED) { ... } else { ... }`** pada `loop()`: mengecek status koneksi setiap 5 detik. Jika masih terhubung → mencetak "Status: Terhubung". Jika terputus → mencetak "Status: Terputus" dan mematikan LED indikator (`digitalWrite(ledPin, LOW)`).

### Hasil Pengamatan

```
WiFi berhasil terhubung!
IP Address  : 10.212.205.171
MAC Address : 84:F3:EB:B7:01:91
RSSI (dBm)  : -45
Status: Terhubung
Status: Terhubung
Status: Terhubung
```

**Analisis:**
- ESP8266 berhasil terhubung ke SSID "naye" tanpa error.
- IP `10.212.205.171` diperoleh dari DHCP server, menunjukkan ESP berada pada jaringan lokal dengan prefix `10.x.x.x`.
- MAC Address `84:F3:EB:B7:01:91` adalah identitas unik perangkat di lapisan data link.
- RSSI -45 dBm tergolong sangat baik (koneksi stabil). Referensi umum kekuatan sinyal:

  | RSSI | Kualitas |
  |---|---|
  | -30 s/d -50 dBm | Sangat baik (stabil) |
  | -50 s/d -70 dBm | Baik (cukup stabil) |
  | -70 s/d -90 dBm | Lemah (tidak stabil) |
  | < -90 dBm | Sangat lemah (sering putus) |

- Loop monitoring berhasil menampilkan status "Terhubung" tiap 5 detik selama praktikum berlangsung, menandakan koneksi tetap stabil.

---

## Percobaan 2A (Modifikasi) — Auto Reconnect

**Tujuan**: Menambahkan mekanisme **timeout** dan **auto-reconnect** agar program tidak terjebak infinite loop bila SSID/password salah, dan agar ESP dapat menyambung ulang otomatis bila koneksi terputus.

```cpp
#include <ESP8266WiFi.h> // ESP32: #include <WiFi.h>

const char* ssid     = "naye";
const char* password = "woylahbroo";
const int maxAttempts = 20; // Batas maksimum percobaan koneksi

void setup() {
  Serial.begin(115200);
  connectToWiFi(); // Panggil fungsi koneksi
}

void loop() {
  // Cek status koneksi setiap 5 detik
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("[INFO] Koneksi WiFi terputus! Mencoba reconnect...");
    connectToWiFi(); // Panggil fungsi reconnect
  } else {
    Serial.print("[INFO] Terhubung | RSSI: ");
    Serial.println(WiFi.RSSI());
  }
  delay(5000);
}

// Fungsi untuk menghubungkan ke WiFi dengan timeout
void connectToWiFi() {
  Serial.print("[INFO] Menghubungkan ke WiFi");
  WiFi.begin(ssid, password);

  int attempts = 0;

  // Loop dengan batasan percobaan (timeout)
  while (WiFi.status() != WL_CONNECTED && attempts < maxAttempts) {
    delay(500);
    Serial.print(".");
    attempts++;
  }

  Serial.println();

  // Cek apakah berhasil atau gagal
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("[SUKSES] WiFi berhasil terhubung!");
    Serial.print("[INFO] IP Address  : ");
    Serial.println(WiFi.localIP());
    Serial.print("[INFO] MAC Address : ");
    Serial.println(WiFi.macAddress());
    Serial.print("[INFO] RSSI (dBm)  : ");
    Serial.println(WiFi.RSSI());
  } else {
    Serial.println("[GAGAL] Tidak dapat terhubung ke WiFi!");
    Serial.println("[INFO] Periksa SSID, password, atau jangkauan sinyal.");
  }
}
```

### Penjelasan Baris Kode Tambahan

| Baris | Penjelasan |
|---|---|
| `const int maxAttempts = 20;` | Variabel batas maksimum percobaan koneksi (20 × 500 ms = 10 detik timeout), mencegah program terjebak selamanya jika SSID/password salah. |
| `void connectToWiFi() { ... }` | Fungsi terpisah (modular) yang membungkus seluruh logika koneksi WiFi, sehingga bisa dipanggil ulang baik dari `setup()` maupun saat `loop()` mendeteksi koneksi terputus. |
| `int attempts = 0;` | Counter untuk menghitung jumlah percobaan koneksi yang sudah dilakukan. |
| `while (WiFi.status() != WL_CONNECTED && attempts < maxAttempts)` | Percabangan gabungan (AND) — loop akan berhenti bila **sudah terhubung** ATAU **jumlah percobaan mencapai batas maksimum**, mana pun yang terjadi lebih dulu. Ini mengganti loop lama yang tidak memiliki batas (infinite loop). |
| `if (WiFi.status() == WL_CONNECTED) { ... } else { ... }` (dalam `connectToWiFi`) | Mengecek hasil akhir setelah loop timeout selesai: jika status sudah `WL_CONNECTED` → cetak info sukses beserta IP/MAC/RSSI; jika belum → cetak pesan gagal beserta saran troubleshooting. |
| `if (WiFi.status() != WL_CONNECTED) { ... } else { ... }` (dalam `loop()`) | Percabangan utama monitoring: jika koneksi **terputus** → panggil ulang `connectToWiFi()` untuk reconnect otomatis; jika **masih terhubung** → cetak nilai RSSI terkini. |

### Kelebihan Modifikasi
- Program tidak akan berhenti selamanya jika SSID/password salah (ada timeout).
- ESP dapat kembali terhubung secara otomatis jika koneksi terputus (auto-reconnect).
- Memberikan informasi status yang lebih jelas (`[INFO]`, `[SUKSES]`, `[GAGAL]`).
- Kode lebih modular karena logika koneksi dipisah ke dalam fungsi `connectToWiFi()`.

---

## Percobaan 2B — Mode Access Point (AP)

**Tujuan**: Mengimplementasikan ESP sebagai Access Point (hotspot) mandiri yang dapat diakses langsung oleh perangkat lain tanpa router eksternal.

```cpp
#include <ESP8266WiFi.h> // Library untuk fungsi WiFi

// Konfigurasi Access Point
const char* ap_ssid     = "ESP32_AccessPoint"; // SSID yang akan muncul
const char* ap_password = "12345678";          // Password (minimal 8 karakter)

void setup() {
  Serial.begin(115200);

  // Set mode WiFi menjadi Access Point
  WiFi.mode(WIFI_AP);

  // Membuat Access Point dengan SSID dan password
  WiFi.softAP(ap_ssid, ap_password);

  // Mendapatkan IP Address dari Access Point
  IPAddress apIP = WiFi.softAPIP();

  Serial.println("=== ACCESS POINT AKTIF ===");
  Serial.print("SSID       : ");
  Serial.println(ap_ssid);
  Serial.print("Password   : ");
  Serial.println(ap_password);
  Serial.print("IP Address : ");
  Serial.println(apIP);
  Serial.println("============================");
  Serial.println();
}

void loop() {
  // Menampilkan jumlah perangkat yang terhubung setiap 5 detik
  int jumlahClient = WiFi.softAPgetStationNum();
  Serial.print("Jumlah perangkat terhubung: ");
  Serial.println(jumlahClient);

  delay(5000);
}
```

### Penjelasan Fungsi

| Fungsi | Penjelasan |
|---|---|
| `WiFi.mode(WIFI_AP)` | Mengatur mode WiFi ESP menjadi **Access Point** (penyedia jaringan). ESP akan membuat jaringan WiFi sendiri. |
| `WiFi.softAP(ap_ssid, ap_password)` | Membuat Access Point dengan SSID dan password tertentu. Fungsi ini membuat ESP bertindak seperti router WiFi mini. |
| `WiFi.softAPIP()` | Mengembalikan alamat IP dari Access Point yang dibuat (default `192.168.4.1`). |
| `WiFi.softAPgetStationNum()` | Mengembalikan jumlah perangkat (station) yang saat ini terhubung ke Access Point ESP. |

### Penjelasan Percabangan / Conditional
Percobaan 2B tidak memiliki percabangan `if/else` — alurnya bersifat sekuensial (setup AP sekali di `setup()`, lalu `loop()` hanya membaca dan mencetak jumlah client tiap 5 detik). Tidak ada logika kondisional karena tujuannya murni monitoring.

### Hasil Pengamatan

**Analisis:**
- ESP berhasil membuat jaringan WiFi baru dengan SSID `ESP32_AccessPoint`, terdeteksi oleh perangkat lain di sekitarnya.
- IP default **192.168.4.1** berada dalam subnet `192.168.4.0/24` dan berfungsi sebagai gateway bagi client yang terhubung.
- ESP secara otomatis menjalankan DHCP server sederhana yang memberi IP ke client.
- `softAPgetStationNum()` berhasil menampilkan jumlah perangkat yang terhubung, bertambah setiap kali ada client baru.
- Password minimal 8 karakter sesuai standar keamanan WiFi (WPA2).
- Keterbatasan: Access Point ESP hanya mendukung sekitar 4–5 client sekaligus karena keterbatasan memori dan sumber daya.

---

## Percobaan 2B (Modifikasi) — Mode AP+STA

**Tujuan**: ESP terhubung ke WiFi rumah (STA) **sekaligus** menyediakan Access Point sendiri (AP) secara bersamaan — berguna untuk skenario *provisioning* perangkat IoT.

```cpp
#include <ESP8266WiFi.h> // ESP32: #include <WiFi.h>

// Konfigurasi Station (koneksi ke WiFi rumah)
const char* sta_ssid     = "naye";        // SSID WiFi rumah
const char* sta_password = "woylahbroo";  // Password WiFi rumah

// Konfigurasi Access Point
const char* ap_ssid     = "ESP32_AP_STA"; // SSID Access Point
const char* ap_password = "12345678";     // Password AP (min 8 karakter)

void setup() {
  Serial.begin(115200);

  // Mengatur mode AP+STA (keduanya aktif secara bersamaan)
  WiFi.mode(WIFI_AP_STA);

  // 1. Membuat Access Point
  WiFi.softAP(ap_ssid, ap_password);
  Serial.println("=== ACCESS POINT AKTIF ===");
  Serial.print("SSID AP : ");
  Serial.println(ap_ssid);
  Serial.print("IP AP   : ");
  Serial.println(WiFi.softAPIP());
  Serial.println("============================");

  // 2. Menghubungkan ke WiFi rumah (Station)
  Serial.print("Menghubungkan ke WiFi rumah");
  WiFi.begin(sta_ssid, sta_password);

  // Timeout untuk koneksi station
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 30) {
    delay(500);
    Serial.print(".");
    attempts++;
  }

  // Tampilkan status koneksi station
  Serial.println();
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("WiFi rumah terhubung!");
    Serial.print("IP Station : ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("Gagal terhubung ke WiFi rumah!");
    Serial.println("(Access Point tetap aktif untuk konfigurasi)");
  }
  Serial.println();
}

void loop() {
  // Menampilkan status kedua mode jaringan
  Serial.println("=== STATUS JARINGAN ===");

  // Informasi Access Point
  Serial.print("Client AP : ");
  Serial.print(WiFi.softAPgetStationNum());
  Serial.println(" perangkat");

  // Informasi Station
  if (WiFi.status() == WL_CONNECTED) {
    Serial.print("Status STA : Terhubung");
    Serial.print(" | IP: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("Status STA : Terputus");
    // Coba reconnect ke WiFi rumah
    Serial.println("Mencoba reconnect ke WiFi rumah...");
    WiFi.begin(sta_ssid, sta_password);
  }

  Serial.println("=========================");
  delay(5000);
}
```

### Penjelasan Baris Kode

| Baris | Penjelasan |
|---|---|
| `WiFi.mode(WIFI_AP_STA)` | Mengaktifkan **dua mode WiFi sekaligus**: Access Point dan Station berjalan bersamaan pada satu modul WiFi ESP. |
| `WiFi.softAP(ap_ssid, ap_password)` | Membuat jaringan Access Point sendiri dengan SSID `ESP32_AP_STA`, agar perangkat lain tetap bisa terhubung langsung ke ESP. |
| `WiFi.begin(sta_ssid, sta_password)` | Memulai koneksi ke jaringan WiFi rumah (mode Station), berjalan paralel dengan AP yang sudah aktif. |
| `while (WiFi.status() != WL_CONNECTED && attempts < 30)` | Percabangan loop dengan timeout (maksimal 30 percobaan × 500 ms = 15 detik) — mencegah `setup()` macet jika WiFi rumah gagal terhubung. |
| `if (WiFi.status() == WL_CONNECTED) { ... } else { ... }` (di `setup()`) | Menentukan pesan yang ditampilkan setelah proses koneksi STA selesai: sukses (tampilkan IP) atau gagal (AP tetap aktif sebagai fallback). |
| `if (WiFi.status() == WL_CONNECTED) { ... } else { ... }` (di `loop()`) | Percabangan monitoring utama: jika STA masih terhubung → tampilkan IP; jika terputus → cetak status "Terputus" dan panggil ulang `WiFi.begin()` untuk mencoba reconnect. |
| `WiFi.softAPgetStationNum()` | Tetap dipanggil di setiap siklus `loop()` untuk memantau jumlah client yang terhubung ke AP, terlepas dari status koneksi STA. |

### Kelebihan Mode AP+STA
1. **Provisioning**: ESP dapat tetap menyediakan AP untuk konfigurasi, sambil terhubung ke internet melalui STA.
2. **Fault Tolerance**: Jika koneksi STA terputus, AP tetap aktif sehingga pengguna masih dapat mengakses ESP.
3. **Dual Function**: ESP dapat berkomunikasi dengan internet (via STA) dan dengan perangkat lokal (via AP) secara bersamaan.

---

## Jawaban Pertanyaan Praktikum

### Pertanyaan Praktikum Percobaan 2A

**1. Gambarkan diagram alur (flowchart) proses koneksi ESP32 ke jaringan WiFi pada program di atas!**
<img width="1741" height="5623" alt="deepseek_mermaid_20260909_c6039f" src="https://github.com/user-attachments/assets/c7a16a0c-5896-473b-9051-42022e693dd5" />


**2. Apa fungsi dari perintah WiFi.mode(WIFI_STA) pada program tersebut?**
Fungsi ini mengatur mode operasi WiFi ESP menjadi Station Mode, yaitu sebagai klien yang akan terhubung ke jaringan WiFi yang sudah ada. Dalam mode ini, ESP akan mencari access point dengan SSID yang sesuai, melakukan autentikasi menggunakan password, lalu mendapatkan alamat IP dari DHCP server. Pada program dasar, fungsi ini sebenarnya tidak wajib dipanggil eksplisit karena `WiFi.begin()` secara default sudah menggunakan mode Station.

**3. Jelaskan apa yang terjadi apabila SSID atau password yang dimasukkan salah!**
- ESP tidak akan pernah mendapatkan status `WL_CONNECTED`.
- Program akan terus berada dalam loop `while (WiFi.status() != WL_CONNECTED)`.
- Serial Monitor akan menampilkan titik-titik (`...`) terus-menerus tanpa henti.
- Informasi jaringan (IP, MAC, RSSI) tidak akan pernah ditampilkan.
- Tidak ada pesan error eksplisit karena program dasar tidak memiliki mekanisme timeout atau penanganan error.
Untuk mengatasi hal ini, sebaiknya ditambahkan mekanisme timeout agar program dapat memberi tahu pengguna bahwa koneksi gagal.

**Modifikasi program agar ESP32 mencoba menghubungkan ulang (reconnect) secara otomatis apabila koneksi WiFi terputus, dan berikan penjelasan di setiap baris kode yang ditambahkan dalam bentuk README.md!**
lihat kode dan penjelasan lengkap pada bagian [Percobaan 2A (Modifikasi) — Auto Reconnect](#percobaan-2a-modifikasi--auto-reconnect) di atas.

### Pertanyaan Praktikum Percobaan 2B

**1. Mengapa alamat IP default Access Point pada ESP32 umumnya bernilai 192.168.4.1?**
Karena ini adalah alamat default yang ditetapkan oleh library ESP untuk mode Access Point, dipilih agar tidak berbenturan dengan rentang IP default router rumah yang umum digunakan (`192.168.1.1` atau `192.168.0.1`).

**2. Apa perbedaan mendasar antara mode Station dan mode Access Point pada ESP32?**

| Mode Station | Mode Access Point |
|---|---|
| ESP berperan sebagai **client** | ESP berperan sebagai **server** |
| Terhubung ke jaringan yang sudah ada | Membuat jaringan sendiri |
| Mendapat IP dari router (DHCP client) | Memberi IP ke client (DHCP server) |

**3. Jelaskan risiko keamanan apabila password Access Point tidak diberikan atau terlalu sederhana!**
- Akses tidak sah ke perangkat oleh pihak yang tidak berwenang.
- Data yang dikirim/diterima dapat disadap (*eavesdropping*).
- Perangkat berpotensi diretas atau dikendalikan oleh orang lain (*unauthorized control*).

**Modifikasi program agar ESP32 berjalan pada mode AP+STA (terhubung ke WiFi rumah sekaligus menyediakan Access Point), dan berikan penjelasan di setiap baris kode nya dalam bentuk README.md!** 
lihat kode dan penjelasan lengkap pada bagian [Percobaan 2B (Modifikasi) — Mode AP+STA](#percobaan-2b-modifikasi--mode-apsta) di atas.

### Pertanyaan Praktikum Analisis Umum

**1. Uraikan hasil tugas pada praktikum yang telah dilakukan pada setiap percobaan!**
- **2A**: ESP8266 berhasil terhubung ke WiFi "naye", memperoleh IP `10.212.205.171`, dengan RSSI **-45 dBm** (sangat baik).
- **2B**: ESP8266 berhasil membuat Access Point "ESP32_AccessPoint" dengan IP default `192.168.4.1`, dan dapat memonitor jumlah client yang terhubung.

**2. Bagaimana pengaruh kekuatan sinyal (RSSI) terhadap kestabilan koneksi WiFi pada perangkat IoT?**
RSSI -45 dBm menandakan sinyal sangat baik dan koneksi stabil. Semakin rendah nilai RSSI (semakin mendekati -90 dBm), semakin lemah sinyalnya, sehingga koneksi menjadi tidak stabil dan lebih sering terputus — hal ini penting diperhatikan pada perangkat IoT yang butuh koneksi kontinu untuk mengirim data sensor secara real-time.

**3. Bagaimana cara kerja ESP32 dalam membedakan peran sebagai klien (Station) dan sebagai penyedia jaringan (Access Point)?**
ESP membedakan peran melalui fungsi `WiFi.mode()` dengan parameter `WIFI_STA`, `WIFI_AP`, atau `WIFI_AP_STA`, dan menggunakan antarmuka jaringan (network interface) yang terpisah secara internal untuk masing-masing peran, sehingga keduanya (pada mode AP+STA) dapat berjalan bersamaan tanpa saling mengganggu.

**4. Bagaimana kombinasi mode Station dan Access Point (AP+STA) dapat dimanfaatkan dalam skenario nyata sistem IoT, misalnya pada proses konfigurasi awal perangkat (provisioning)?**
Mode AP+STA sangat berguna untuk *provisioning* perangkat IoT: ESP dapat terhubung ke internet (via STA) untuk operasional normal, sambil tetap menyediakan Access Point (AP) yang memungkinkan pengguna mengubah konfigurasi (misalnya SSID/password baru) melalui halaman web tanpa perlu memprogram ulang firmware.

---

## Foto Proses Praktikum/Dokumentasi

| Percobaan | Rangkaian | Serial Monitor |
|---|---|---|
| 2A | `<img width="3024" height="4032" alt="Rangkaian 2A dan 2B" src="https://github.com/user-attachments/assets/7a614981-673f-416d-a9de-f4a2cb6024f6" />
` | `<img width="1600" height="673" alt="Serial Monitor 2A" src="https://github.com/user-attachments/assets/cad74676-945b-4ba0-a8bc-de781a6a4bbf" />
` |
| 2B | `<img width="3024" height="4032" alt="Rangkaian 2A dan 2B" src="https://github.com/user-attachments/assets/e8ea0c13-9867-43a4-82d6-0903cd77288d" />
` | `<img width="1600" height="948" alt="Seial Monitor 2B" src="https://github.com/user-attachments/assets/88cadd24-c4fe-46ad-bb23-ad97d34d4780" />
` |

---
