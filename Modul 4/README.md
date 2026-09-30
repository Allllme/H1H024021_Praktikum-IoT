# Praktikum IoT Modul 4 – Komunikasi dan Pertukaran Data Dua Arah (MQTT Publish & Subscribe)

| | |
|---|---|
| **Mata Kuliah** | Praktikum Internet of Things (TK245005) |
| **Program Studi** | Teknik Komputer |
| **Tahun / Semester** | 2024 / 5 |
| **Modul** | 4 |
| **Nama Praktikan / NIM** | Alma Maida Wirastuti / H1H024021 |

---

## Tujuan Praktikum

1. Memahami konsep pertukaran data dua arah (*bidirectional*) pada sistem IoT.
2. Memahami mekanisme *subscribe* dan proses deserialisasi data JSON pada ESP8266.
3. Mengimplementasikan penerimaan perintah kendali melalui MQTT untuk menyalakan dan mematikan LED (aktuator) secara *real-time*.
4. Mengimplementasikan sistem IoT yang mempublikasikan data sensor DHT11 dan menerima perintah kendali secara bersamaan (*full duplex*) dengan pendekatan *non-blocking* menggunakan `millis()`.
5. Menganalisis mekanisme pertukaran data IoT secara menyeluruh pada sistem yang saling terhubung.

---

## Alat dan Bahan

- Board NodeMCU ESP8266 (ESP-12E) – 1 buah
- Sensor DHT11 – 1 buah
- LED – 1 buah, Resistor 220 Ω – 1 buah
- Breadboard dan kabel jumper
- Kabel USB Micro-USB
- Laptop/PC dengan Arduino IDE
- Jaringan WiFi yang terhubung ke internet
- MQTT Explorer / HiveMQ WebSocket Client (untuk publish perintah dan memantau data)
- Broker MQTT publik `broker.hivemq.com` (port 1883)

---

## Library / Dependencies

| Library | Kegunaan |
|---|---|
| `ESP8266WiFi.h` | Koneksi WiFi ESP8266 (bawaan board package ESP8266) |
| `PubSubClient` | Klien MQTT (connect, publish, subscribe, callback) |
| `ArduinoJson` (v7, memakai `JsonDocument`) | Serialisasi & deserialisasi JSON |
| `DHT sensor library` (Adafruit) | Membaca sensor DHT11 (Percobaan 4B) |
| `Adafruit Unified Sensor` | Dependensi dari DHT sensor library |

**Board:** NodeMCU 1.0 (ESP-12E Module), **Port:** COM3, **Baud rate Serial Monitor:** 115200.

---

## Konfigurasi Umum

| Parameter | Nilai |
|---|---|
| Broker | `broker.hivemq.com` |
| Port | `1883` |
| Topic perintah | `unsoed/tk245004/kelompok2/perintah` |
| Topic data (4B) | `unsoed/tk245004/kelompok2/data` |
| Topic buzzer (soal 4B no. 4) | `unsoed/tk245004/kelompok2/buzzer` |
| Format perintah | `{"perintah":"ON"}` / `{"perintah":"OFF"}` |
| Format data | `{"suhu":26.2}` |

---

# Percobaan 4A – Subscribe dan Deserialisasi JSON

## Deskripsi Percobaan

ESP8266 melakukan *subscribe* pada topic perintah. Perintah dikirim dari MQTT Explorer dalam format JSON, misalnya `{"perintah":"ON"}`. Pesan yang masuk dideserialisasi dengan ArduinoJson, lalu nilai kunci `perintah` menentukan LED menyala atau mati.

## Rangkaian

| Komponen | Kaki Komponen | Pin ESP8266 |
|---|---|---|
| LED | Anoda (+) melalui resistor 220 Ω | D1 (GPIO5) |
| LED | Katoda (–) | GND |

<img width="960" height="1280" alt="Rangkaian Percobaan 4A" src="https://github.com/user-attachments/assets/a73dd3c1-b598-420a-a08a-481f24057350" />


*Gambar 1. Rangkaian Percobaan 4A*

## Kode Program

```cpp
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>

const char* ssid     = "rakjel";
const char* password = "********";

const char* mqttServer = "broker.hivemq.com";
const int   mqttPort   = 1883;
const char* topicPerintah = "unsoed/tk245004/kelompok2/perintah";

const int ledPin = 5;   // GPIO5 = D1

WiFiClient espClient;
PubSubClient client(espClient);

// Fungsi callback dipanggil otomatis setiap ada pesan baru masuk
void callback(char* topic, byte* payload, unsigned int length) {
  String pesan;
  for (unsigned int i = 0; i < length; i++) {
    pesan += (char)payload[i];
  }
  Serial.print("Pesan diterima [");
  Serial.print(topic);
  Serial.print("]: ");
  Serial.println(pesan);

  // Deserialisasi data JSON yang diterima
  JsonDocument doc;
  DeserializationError error = deserializeJson(doc, pesan);
  if (error) {
    Serial.print("Gagal parsing JSON: ");
    Serial.println(error.c_str());
    return;
  }

  const char* perintah = doc["perintah"];
  if (String(perintah) == "ON") {
    digitalWrite(ledPin, HIGH);
    Serial.println("Aktuator: ON");
  } else if (String(perintah) == "OFF") {
    digitalWrite(ledPin, LOW);
    Serial.println("Aktuator: OFF");
  }
}

void hubungkanWiFi() {
  WiFi.begin(ssid, password);
  Serial.print("Menghubungkan ke WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi berhasil terhubung!");
}

void hubungkanMQTT() {
  while (!client.connected()) {
    Serial.print("Menghubungkan ke broker MQTT...");
    String clientId = "ESP8266Client-" + String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      Serial.println("berhasil terhubung!");
      client.subscribe(topicPerintah);   // subscribe setelah berhasil terhubung
      Serial.print("Subscribe ke topic: ");
      Serial.println(topicPerintah);
    } else {
      Serial.print("gagal, rc=");
      Serial.print(client.state());
      Serial.println(" coba lagi dalam 2 detik");
      delay(2000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, LOW);
  hubungkanWiFi();
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback);          // daftarkan fungsi callback
}

void loop() {
  if (!client.connected()) {
    hubungkanMQTT();
  }
  client.loop();   // wajib dipanggil terus-menerus agar pesan dapat diterima
}
```

## Penjelasan Kode

### 1. Library dan Konfigurasi

| Kode | Penjelasan |
|---|---|
| `#include <ESP8266WiFi.h>` `<PubSubClient.h>` `<ArduinoJson.h>` | Memasukkan pustaka WiFi ESP8266, PubSubClient (MQTT), dan ArduinoJson untuk deserialisasi JSON. |
| `ssid`, `password` | Konfigurasi WiFi (password disensor). |
| `mqttServer`, `mqttPort` | Alamat dan port broker publik HiveMQ (1883). |
| `topicPerintah` | Topic tempat ESP8266 menerima perintah. |
| `ledPin = 5` | LED dipasang pada GPIO5 (pin D1). |
| `WiFiClient espClient;` `PubSubClient client(espClient);` | Objek `WiFiClient` dipakai `PubSubClient` sebagai jalur koneksi ke broker. |

### 2. Penjelasan Setiap Fungsi

| Fungsi | Penjelasan |
|---|---|
| `callback(topic, payload, length)` | Dipanggil otomatis setiap ada pesan baru pada topic yang di-*subscribe*. Payload (array byte) disusun menjadi `String`, ditampilkan di Serial Monitor, lalu dideserialisasi dengan `deserializeJson()`. Nilai kunci `"perintah"` dibaca untuk mengendalikan LED. |
| `hubungkanWiFi()` | Menghubungkan ESP8266 ke WiFi. Program menunggu (titik-titik di Serial Monitor) sampai status `WL_CONNECTED`, lalu menampilkan pesan berhasil. |
| `hubungkanMQTT()` | Selama belum terhubung ke broker, membuat `clientId` acak lalu `client.connect()`. Jika berhasil, langsung `client.subscribe(topicPerintah)`. Jika gagal, menampilkan kode `rc` dan mencoba lagi setelah 2 detik. |
| `setup()` | Inisialisasi Serial (115200), mengatur `ledPin` sebagai OUTPUT dan mati, menghubungkan WiFi, mengatur server broker, dan mendaftarkan fungsi callback. |
| `loop()` | Jika koneksi broker terputus, panggil `hubungkanMQTT()`. `client.loop()` dipanggil terus-menerus untuk menerima pesan dan menjaga koneksi. |

### 3. Penjelasan Percabangan / Conditional

| Percabangan | Penjelasan |
|---|---|
| `if (error) { ...; return; }` | Jika parsing JSON gagal, tampilkan pesan kesalahan (`error.c_str()`) lalu keluar dari callback sehingga LED tidak berubah. |
| `if (String(perintah) == "ON")` | Perintah `ON` → `digitalWrite(ledPin, HIGH)`, LED menyala. |
| `else if (String(perintah) == "OFF")` | Perintah `OFF` → `digitalWrite(ledPin, LOW)`, LED mati. |
| (tanpa `else`) | Nilai perintah lain diabaikan, status LED tidak berubah. |
| `while (WiFi.status() != WL_CONNECTED)` | Perulangan menunggu WiFi terhubung. |
| `while (!client.connected())` / `if (client.connect(...))` | Perulangan hingga terhubung ke broker; jika berhasil subscribe, jika gagal coba lagi. |
| `if (!client.connected())` di `loop()` | Menyambungkan ulang ke broker bila koneksi terputus. |

## Hasil Pengamatan

Program diunggah ke board NodeMCU 1.0 (ESP-12E Module) pada port COM3 dengan baud rate 115200. Serial Monitor menampilkan proses koneksi WiFi, koneksi ke broker, dan *subscribe* ke topic perintah.

<img width="1600" height="302" alt="Serial Monitor 4A" src="https://github.com/user-attachments/assets/aed7ef7f-4b7f-461a-95f9-0e45c63a313b" />


*Gambar 2. Serial Monitor Percobaan 4A (koneksi WiFi, broker, dan subscribe)*

## Jawaban Pertanyaan Praktikum 4A

### 1. Flowchart proses penerimaan dan pemrosesan pesan pada callback

<img width="1063" height="1487" alt="flowchart_callback" src="https://github.com/user-attachments/assets/f61f477b-f27c-41d6-9f6e-db34a3144e46" />

*Gambar 3. Diagram alur fungsi callback pada Percobaan 4A*

### 2. Apa yang terjadi jika pesan bukan JSON yang valid?

`deserializeJson()` mengembalikan status kesalahan (misalnya `InvalidInput`), sehingga kondisi `if (error)` bernilai benar. Program menampilkan `"Gagal parsing JSON: "` diikuti keterangan kesalahan di Serial Monitor, lalu keluar dari fungsi callback dengan `return`. Nilai `"perintah"` tidak dibaca sehingga status LED tidak berubah dan program tetap berjalan normal.

### 3. Mengapa `client.subscribe()` dipanggil di `hubungkanMQTT()`, bukan di `setup()`?

*Subscribe* hanya dapat dilakukan setelah ESP8266 terhubung ke broker, sedangkan pada `setup()` koneksi ke broker belum terbentuk (baru `client.setServer()`, belum `client.connect()`). Selain itu, langganan (*subscription*) tidak dipertahankan ketika koneksi terputus, sehingga setiap kali koneksi dibuat ulang oleh `hubungkanMQTT()`, ESP8266 harus melakukan *subscribe* lagi. Dengan meletakkannya di dalam blok koneksi berhasil, *subscribe* selalu dijalankan pada koneksi yang baru terbentuk.

### 4. Modifikasi: intensitas LED dengan PWM

Format pesan: `{"perintah": "ON", "intensitas": 200}`

**Tambahan di `setup()`:**

```cpp
analogWriteRange(255);      // ubah rentang PWM 0-1023 menjadi 0-255
analogWrite(ledPin, 0);     // pastikan LED mati saat awal
```

**Callback setelah modifikasi:**

```cpp
  const char* perintah = doc["perintah"];
  int intensitas = doc["intensitas"] | 255;        // default 255 bila kunci tidak ada
  intensitas = constrain(intensitas, 0, 255);      // batasi 0-255

  if (String(perintah) == "ON") {
    analogWrite(ledPin, intensitas);
    Serial.print("Aktuator: ON, intensitas = ");
    Serial.println(intensitas);
  } else if (String(perintah) == "OFF") {
    analogWrite(ledPin, 0);
    Serial.println("Aktuator: OFF");
  }
```

| Kode yang ditambahkan | Penjelasan |
|---|---|
| `int intensitas = doc["intensitas"] \| 255;` | Membaca nilai kunci `"intensitas"` dari JSON. Operator `\|` memberi nilai bawaan 255 bila kunci tidak ada, sehingga pesan lama `{"perintah":"ON"}` tetap menyalakan LED terang penuh. |
| `intensitas = constrain(intensitas, 0, 255);` | Membatasi nilai pada rentang 0–255 agar nilai di luar rentang tidak menghasilkan PWM yang salah. |
| `analogWrite(ledPin, intensitas);` (pada `ON`) | Kecerahan LED diatur dengan PWM sesuai nilai intensitas (menggantikan `digitalWrite HIGH`), dan nilainya ditampilkan di Serial Monitor. |
| `analogWrite(ledPin, 0);` (pada `OFF`) | PWM diatur 0 sehingga LED mati (menggantikan `digitalWrite LOW`). |
| `analogWriteRange(255);` (di `setup()`) | Mengubah rentang PWM ESP8266 dari bawaan 0–1023 menjadi 0–255 agar sesuai contoh intensitas 200. |
| `analogWrite(ledPin, 0);` (di `setup()`) | Memastikan LED mati saat awal. |

---

# Percobaan 4B – Publish dan Subscribe Bersamaan (Full Duplex)

## Deskripsi Percobaan

Program 4A dikembangkan agar ESP8266 juga mempublikasikan data suhu dari DHT11 setiap 5 detik ke topic data, sambil tetap menerima perintah LED dari topic perintah. Pengaturan waktu publish memakai `millis()` (non-blocking) sehingga `client.loop()` terus berjalan tanpa terhambat `delay()`.

## Skematik Rangkaian

| Komponen | Kaki Komponen | Pin ESP8266 |
|---|---|---|
| LED | Anoda (+) melalui resistor 220 Ω | D1 (GPIO5) |
| LED | Katoda (–) | GND |
| DHT11 | VCC | 3V3 |
| DHT11 | DATA | D2 (GPIO4) |
| DHT11 | GND | GND |

<img width="960" height="1280" alt="Rangkaian Percobaan 4B" src="https://github.com/user-attachments/assets/f0cd5d16-315c-4586-adf1-a28d1e9011de" />

*Gambar 4. Rangkaian Percobaan 4B*

## Kode Program

Bagian `hubungkanWiFi()` dan `hubungkanMQTT()` sama seperti Percobaan 4A (versi tanpa pesan detail), termasuk *subscribe* ke topic perintah.

```cpp
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <DHT.h>

const char* ssid     = "rakjel";
const char* password = "********";

const char* mqttServer = "broker.hivemq.com";
const int   mqttPort   = 1883;
const char* topicData     = "unsoed/tk245004/kelompok2/data";
const char* topicPerintah = "unsoed/tk245004/kelompok2/perintah";

#define DHTPIN 4          // GPIO4 = D2
#define DHTTYPE DHT11
const int ledPin = 5;     // GPIO5 = D1

DHT dht(DHTPIN, DHTTYPE);
WiFiClient espClient;
PubSubClient client(espClient);

unsigned long waktuTerakhirPublish = 0;
const long intervalPublish = 5000;   // publish setiap 5 detik (non-blocking)

void callback(char* topic, byte* payload, unsigned int length) {
  String pesan;
  for (unsigned int i = 0; i < length; i++) pesan += (char)payload[i];

  JsonDocument doc;
  if (deserializeJson(doc, pesan)) return;   // abaikan jika parsing gagal

  const char* perintah = doc["perintah"];
  digitalWrite(ledPin, String(perintah) == "ON" ? HIGH : LOW);
  Serial.print("Perintah diterima -> Aktuator: ");
  Serial.println(perintah);
}

void hubungkanWiFi() {
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.println("WiFi berhasil terhubung!");
}

void hubungkanMQTT() {
  while (!client.connected()) {
    String clientId = "ESP8266Client-" + String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      client.subscribe(topicPerintah);
      Serial.println("Terhubung dan subscribe topic perintah");
    } else {
      delay(2000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);
  dht.begin();
  hubungkanWiFi();
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) hubungkanMQTT();
  client.loop();   // memproses pesan masuk secara terus-menerus

  // Publish data sensor secara berkala tanpa memblokir proses subscribe
  if (millis() - waktuTerakhirPublish > intervalPublish) {
    waktuTerakhirPublish = millis();

    float suhu = dht.readTemperature();
    if (!isnan(suhu)) {
      JsonDocument doc;
      doc["suhu"] = suhu;

      char buffer[128];
      serializeJson(doc, buffer);
      client.publish(topicData, buffer);

      Serial.print("Data terkirim: ");
      Serial.println(buffer);
    }
  }
}
```

## Penjelasan Kode

### 1. Tambahan dibanding Percobaan 4A

| Kode | Penjelasan |
|---|---|
| `#include <DHT.h>` | Pustaka sensor DHT. |
| `topicData` | Topic untuk mempublikasikan data sensor (selain topic perintah). |
| `DHTPIN 4`, `DHTTYPE DHT11`, `DHT dht(...)` | Sensor DHT11 pada GPIO4 (pin D2). |
| `waktuTerakhirPublish`, `intervalPublish` (5000 ms) | Variabel pengaturan publish secara non-blocking. |
| `dht.begin()` di `setup()` | Mengaktifkan sensor DHT11. |

### 2. Penjelasan Setiap Fungsi

| Fungsi | Penjelasan |
|---|---|
| `callback()` | Versi ringkas: pesan dideserialisasi, jika parsing gagal pesan diabaikan. LED dinyalakan bila perintah `"ON"` dan dimatikan untuk nilai lainnya. |
| `hubungkanWiFi()` | Sama seperti 4A. |
| `hubungkanMQTT()` | Sama seperti 4A (tanpa pesan detail), termasuk *subscribe* ke topic perintah. |
| `setup()` | Sama seperti 4A ditambah `dht.begin()`. |
| `loop()` | `client.loop()` dipanggil setiap putaran tanpa `delay()`, sehingga perintah LED tetap dapat diterima. Data sensor dikirim dengan `millis()` hanya bila selisih waktu melebihi 5 detik. |

### 3. Penjelasan Percabangan / Conditional

| Percabangan | Penjelasan |
|---|---|
| `if (deserializeJson(doc, pesan)) return;` | Jika parsing gagal (nilai tidak nol), callback berhenti dan pesan diabaikan. |
| `String(perintah) == "ON" ? HIGH : LOW` | Operator ternary: `"ON"` → LED HIGH, selain itu → LOW. |
| `if (!client.connected()) hubungkanMQTT();` | Menyambung ulang ke broker jika terputus. |
| `if (millis() - waktuTerakhirPublish > intervalPublish)` | Kondisi non-blocking: blok publish hanya dijalankan bila sudah lewat 5 detik. |
| `if (!isnan(suhu))` | Data hanya dikirim jika pembacaan DHT11 valid (bukan `NaN`). |

## Hasil Pengamatan

Setelah terhubung dan *subscribe* ke topic perintah, ESP8266 mempublikasikan data suhu ke topic data secara berkala. Data pertama bernilai 26,7 °C, sedangkan data berikutnya relatif konstan pada 26,2 °C.

```
Terhubung dan subscribe topic perintah
Data terkirim: {"suhu":26.7}
Data terkirim: {"suhu":26.2}
Data terkirim: {"suhu":26.2}
...
```

<img width="493" height="413" alt="Serial Monitor 4B" src="https://github.com/user-attachments/assets/c863a5c8-427a-477a-8135-433370874342" />

*Gambar 5. Serial Monitor Percobaan 4B (data suhu terkirim berkala)*

## Jawaban Pertanyaan Praktikum 4B

### 1. Mengapa `delay()` yang lama sebaiknya dihindari?

`delay()` bersifat *blocking*: selama jeda berlangsung seluruh program berhenti sehingga `client.loop()` tidak dipanggil. Akibatnya pesan perintah yang masuk tidak segera diproses (LED terlambat merespons) dan koneksi ke broker tidak dijaga. Jika jeda lebih lama dari waktu *keep-alive* (bawaan PubSubClient 15 detik), broker dapat memutus koneksi. Proses publish juga ikut tertahan oleh jeda tersebut.

### 2. Cara kerja non-blocking dengan `millis()`

Variabel `waktuTerakhirPublish` menyimpan waktu publish terakhir. Pada setiap putaran `loop()`, program memeriksa apakah `millis() - waktuTerakhirPublish` sudah lebih besar dari `intervalPublish` (5000 ms). Jika belum, program langsung melanjutkan putaran berikutnya tanpa menunggu, sehingga `client.loop()` terus dipanggil. Jika sudah, `waktuTerakhirPublish` diperbarui, suhu dibaca, diserialisasi ke JSON, lalu dipublikasikan. Dengan cara ini publish berkala dan penerimaan perintah berjalan bersamaan.

### 3. Jika `client.loop()` jarang dipanggil (misalnya 10 detik sekali)?

Pesan perintah yang masuk baru diproses saat `client.loop()` dipanggil, sehingga LED dapat terlambat merespons hingga sekitar 10 detik. Selain itu paket *keep-alive* ke broker juga baru diproses saat `client.loop()` dipanggil; bila jaraknya mendekati atau melebihi *keep-alive* (15 detik), broker dapat menganggap perangkat terputus sehingga koneksi harus dibuat ulang.

### 4. Modifikasi: topic perintah baru untuk buzzer

**Tambahan variabel global:**

```cpp
const char* topicBuzzer = "unsoed/tk245004/kelompok2/buzzer";
const int buzzerPin = 14;   // GPIO14 = D5
```

**Callback yang membedakan topic:**

```cpp
void callback(char* topic, byte* payload, unsigned int length) {
  String pesan;
  for (unsigned int i = 0; i < length; i++) pesan += (char)payload[i];

  JsonDocument doc;
  if (deserializeJson(doc, pesan)) return;

  const char* perintah = doc["perintah"];
  bool aktif = (String(perintah) == "ON");

  if (strcmp(topic, topicPerintah) == 0) {
    digitalWrite(ledPin, aktif ? HIGH : LOW);
  } else if (strcmp(topic, topicBuzzer) == 0) {
    digitalWrite(buzzerPin, aktif ? HIGH : LOW);
  }
}
```

**Di `hubungkanMQTT()`** (setelah subscribe topic perintah):

```cpp
client.subscribe(topicBuzzer);
```

**Di `setup()`:**

```cpp
pinMode(buzzerPin, OUTPUT);
digitalWrite(buzzerPin, LOW);
```

| Kode yang ditambahkan | Penjelasan |
|---|---|
| `const char* topicBuzzer = ...;` `const int buzzerPin = 14;` | Topic khusus perintah buzzer (terpisah dari topic LED) dan pin buzzer pada GPIO14 (D5). Pin disesuaikan dengan rangkaian. |
| `bool aktif = (String(perintah) == "ON");` | Mengubah nilai perintah menjadi `true`/`false` agar dapat dipakai bersama oleh kedua aktuator. |
| `if (strcmp(topic, topicPerintah) == 0) { ... } else if (strcmp(topic, topicBuzzer) == 0) { ... }` | Callback membedakan topic penerima pesan dengan `strcmp()` (bernilai 0 jika teks sama). Pesan dari topic perintah menggerakkan LED, pesan dari topic buzzer menggerakkan buzzer. |
| `client.subscribe(topicBuzzer);` | Mendaftarkan ESP8266 ke topic buzzer setiap koneksi ke broker terbentuk (termasuk setelah koneksi ulang). |
| `pinMode(buzzerPin, OUTPUT);` `digitalWrite(buzzerPin, LOW);` | Mengatur pin buzzer sebagai keluaran dan memastikan buzzer mati saat program mulai. |

---

# Pertanyaan Analisis

### 1. Uraian hasil tiap percobaan

- **Percobaan 4A:** ESP8266 (NodeMCU ESP-12E) berhasil terhubung ke WiFi dan broker HiveMQ, lalu melakukan *subscribe* ke topic `unsoed/tk245004/kelompok2/perintah`.
- **Percobaan 4B:** ESP8266 mempublikasikan data suhu dari DHT11 ke topic data secara berkala; nilai awal yang terkirim 26,7 °C dan selanjutnya sekitar 26,2 °C.

### 2. Perbandingan komunikasi satu arah dan dua arah

Pada komunikasi satu arah, ESP8266 hanya berperan sebagai *publisher* yang mengirim data sensor ke broker; perangkat tidak menerima apa pun sehingga program cukup memanggil `client.publish()` dan tidak membutuhkan callback. Pada komunikasi dua arah, ESP8266 juga menjadi *subscriber*: perangkat mendaftar ke topic perintah dengan `client.subscribe()`, menerima pesan melalui fungsi callback, dan mendeserialisasi JSON untuk menggerakkan aktuator. Program menjadi lebih kompleks karena `client.loop()` harus dipanggil terus-menerus dan pengiriman data harus non-blocking.

Sistem satu arah hanya dapat memantau, sedangkan sistem dua arah dapat memantau sekaligus mengendalikan perangkat dari jarak jauh secara *real-time*.

### 3. Mengapa non-blocking (`millis()`) lebih sesuai daripada blocking (`delay()`)?

`delay()` menghentikan seluruh program selama jeda sehingga `client.loop()` tidak dipanggil. Pesan perintah yang masuk terlambat diproses dan koneksi ke broker dapat terputus bila jeda melebihi waktu *keep-alive*. Dengan `millis()`, program hanya membandingkan selisih waktu sekarang dengan waktu publish terakhir; data dikirim saat interval tercapai, sedangkan pada putaran lainnya `client.loop()` tetap berjalan. Dengan demikian perintah kendali direspons segera tanpa mengganggu pengiriman data sensor berkala.

### 4. Contoh penerapan komunikasi dua arah pada IoT nyata

**Smart farming.** Sensor kelembapan tanah dan suhu mempublikasikan data ke topic data, sedangkan pompa irigasi menerima perintah ON/OFF dari aplikasi melalui topic perintah dan mengirim kembali status pompa.

Manfaat dibanding sistem satu arah:
- Petani tidak hanya melihat kondisi lahan tetapi dapat langsung menyiram dari jarak jauh.
- Penyiraman dapat diotomatisasi berdasarkan data sensor.
- Status pompa dapat dikonfirmasi sehingga penggunaan air lebih efisien.

---

# Dokumentasi

## Foto Proses Praktikum / Perangkaian

| Percobaan 4A | Percobaan 4B |
|---|---|
| <img width="960" height="1280" alt="Rangkaian Percobaan 4A" src="https://github.com/user-attachments/assets/1fe5d0f9-f8b8-463d-9207-f78b754a899e" />
 | <img width="493" height="413" alt="Serial Monitor 4B" src="https://github.com/user-attachments/assets/cec8c4f3-be80-41b2-a3c7-90716a2881c6" />
 |
