# Modul 3 – Protokol Komunikasi IoT 

| | |
|---|---|
| Nama Mata Kuliah (Kode) | Internet Of things (TK245005) |
| Tahun / Semester | 2024 / 5 |
| Modul | 3 |
| Nama Praktikan / NIM | Alma Maida Wirastuti / H1H024021 |

Repository ini berisi dokumentasi dan source code hasil praktikum Modul 3 mata kuliah Praktikum Sistem Mikrokontroler/IoT, Membahas implementasi protokol **HTTP (POST)** dan **MQTT (publish-subscribe)** untuk mengirim data sensor dalam format **JSON** dari mikrokontroler ke server/broker.


## Tujuan Praktikum

1. Memahami konsep dasar protokol komunikasi pada sistem IoT.
2. Memahami karakteristik dan perbedaan protokol HTTP dan MQTT dalam konteks IoT.
3. Mengimplementasikan pengiriman data dari mikrokontroler ke server menggunakan protokol HTTP (metode POST).
4. Mengimplementasikan pertukaran data dari mikrokontroler ke broker MQTT menggunakan pola publish-subscribe dengan format JSON.
5. Menganalisis kelebihan dan kekurangan masing-masing protokol untuk berbagai skenario aplikasi IoT.

## Alat dan Bahan

| Alat/Bahan | Keterangan |
|---|---|
| NodeMCU 1.0 (ESP-12E, ESP8266) | Pengganti ESP32 DevKit |
| Kabel USB Micro-USB | Untuk pemrograman & power |
| Laptop/PC + Arduino IDE 2.3.10 | Sudah terpasang board manager ESP8266 |
| Jaringan WiFi (terhubung internet) | SSID: `rakjel` |
| MQTT Explorer | Client untuk verifikasi data MQTT |
| Broker MQTT publik | `broker.hivemq.com` port `1883` |
| Endpoint uji HTTP | `https://httpbin.org/post` |

## Library / Dependencies

Diinstal melalui Library Manager pada Arduino IDE:

| Library | Fungsi |
|---|---|
| `ESP8266WiFi.h` | Bawaan board package ESP8266 — mengelola koneksi WiFi |
| `ESP8266HTTPClient.h` | Bawaan board package ESP8266 — membuat request HTTP (GET/POST) |
| `WiFiClientSecure.h` | Bawaan board package ESP8266 — membuat koneksi TLS/HTTPS |
| `PubSubClient` by Nick O'Leary | Komunikasi protokol MQTT (connect, publish, subscribe) |
| `ArduinoJson` by Benoit Blanchon | Membuat & mengurai (parse) data berformat JSON |

Board package yang digunakan: **esp8266 by ESP8266 Community**, board **NodeMCU 1.0 (ESP-12E Module)**.

---

## Percobaan 3A — Komunikasi HTTP

### Gambar Rangkaian

<img width="3024" height="4032" alt="Rangkaian 3A   3B" src="https://github.com/user-attachments/assets/f69d78a7-0bea-4fa3-85c1-1e15ba3e1962" />

NodeMCU terhubung ke laptop hanya melalui kabel USB (power + serial), lalu terhubung secara nirkabel ke WiFi untuk mengakses internet dan mengirim data ke endpoint HTTP.

### Penjelasan Code

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <WiFiClientSecure.h>
#include <ArduinoJson.h>
```
Mengimpor pustaka yang dibutuhkan: koneksi WiFi, klien HTTP, klien TLS (karena endpoint memakai HTTPS), dan pembuatan JSON.

```cpp
const char* ssid     = "rakjel";
const char* password = "entersaja";
const char* serverUrl = "https://httpbin.org/post";
```
Konstanta konfigurasi: nama & sandi jaringan WiFi, serta alamat endpoint tujuan pengiriman data.

### Penjelasan Setiap Fungsi

**`setup()`** — dijalankan sekali saat board menyala/reset.
- `Serial.begin(115200)` mengaktifkan komunikasi serial untuk debugging di Serial Monitor.
- `WiFi.begin(ssid, password)` memulai proses koneksi ke access point.
- Loop `while` menunggu hingga status WiFi menjadi `WL_CONNECTED` sebelum melanjutkan, sambil mencetak `.` sebagai indikator progres.

**`loop()`** — dijalankan berulang-ulang selama board menyala.
- Membuat objek `WiFiClientSecure` dan memanggil `setInsecure()` agar koneksi TLS tidak memverifikasi sertifikat server (hanya untuk keperluan pengujian).
- Membuat objek `HTTPClient`, memulai koneksi ke `serverUrl` melalui `http.begin(client, serverUrl)`, lalu menambahkan header `Content-Type: application/json` agar server tahu body yang dikirim berformat JSON.
- Membuat `JsonDocument`, mengisi key `suhu` dan `kelembaban`, lalu mengubahnya menjadi string dengan `serializeJson(doc, requestBody)`.
- Mengirim `requestBody` sebagai body request melalui `http.POST(requestBody)`, yang mengembalikan kode status HTTP.
- Mencetak kode status dan isi respons (`http.getString()`) ke Serial Monitor.
- Menutup koneksi dengan `http.end()`.

### Penjelasan Percabangan/Conditional

| Kondisi | Fungsi |
|---|---|
| `while (WiFi.status() != WL_CONNECTED)` | Menahan eksekusi program di `setup()` sampai koneksi WiFi benar-benar berhasil, mencegah program lanjut ke `loop()` tanpa jaringan. |
| `if (WiFi.status() == WL_CONNECTED)` | Memastikan proses pengiriman HTTP POST hanya dijalankan jika WiFi masih terhubung, mencegah error jika koneksi terputus di tengah jalan. |
| `if (httpResponseCode > 0) { ... } else { ... }` | Membedakan penanganan antara pengiriman yang berhasil mendapat respons dari server (`httpResponseCode > 0`, misalnya 200) dengan kegagalan pengiriman (kode negatif menandakan error koneksi/timeout). |

### Hasil Pengamatan

Board berhasil terhubung ke WiFi, mengirim data `{"suhu":28.5,"kelembaban":65}` ke `https://httpbin.org/post`, dan menerima kode respons HTTP 200 (OK). Isi respons mengembalikan (echo) data JSON yang sama persis pada field `"json"`, sebagai bukti data diterima dengan benar oleh server. Proses berulang setiap 10 detik dengan hasil yang konsisten.
<img width="1600" height="940" alt="Serial Monitor 3A" src="https://github.com/user-attachments/assets/bec93788-e3c9-45be-bd94-77bf433b46a2" />

Hasil ini sesuai dengan spesifikasi yang diharapkan pada modul: WiFi berhasil terhubung, data terkirim dalam format JSON, kode status dan isi respons tampil di Serial Monitor, dan tidak terjadi error saat kompilasi maupun pengiriman.

### Jawaban Pertanyaan Praktikum 3A

**1. Gambarkan diagram alur (flowchart) proses pengiriman data melalui HTTP POST pada program di atas!**

<img width="910" height="3717" alt="Flowchart 3A" src="https://github.com/user-attachments/assets/062bcbd9-446f-4c92-989e-e887f5d7d75d" />

**2. Apa fungsi dari perintah http.addHeader("Content-Type", "application/json") pada program tersebut?**

Perintah ini menambahkan header HTTP yang memberi tahu server bahwa **body request yang dikirim berformat JSON**. Dengan header ini, server (httpbin.org) dapat mem-parsing body sebagai objek JSON yang valid (muncul pada field `"json"` di respons), bukan diperlakukan sebagai teks biasa atau form data.

**3. Jelaskan arti dari kode response HTTP 200 dan sebutkan salah satu contoh kode response HTTP lain beserta artinya!**

HTTP 200 (OK) berarti request berhasil diproses server dan server mengembalikan response 
yang diminta. Ini kode standar untuk request sukses. 
Contoh kode lain: 

• 404 (Not Found): Resource/endpoint tidak ditemukan di server. 

• 500 (Internal Server Error): Kesalahan internal pada server. 

• 400 (Bad Request): Request klien tidak valid.

**4. Modifikasi program agar ESP32 dapat mengirimkan data tambahan berupa waktu (dalam milidetik sejak dinyalakan menggunakan millis()) ke dalam JSON yang dikirim, dan berikan penjelasan di setiap baris kode yang ditambahkan dalam bentuk README.md**
```
void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    WiFiClientSecure client;
    client.setInsecure();

    HTTPClient http;
    http.begin(client, serverUrl);
    http.addHeader("Content-Type", "application/json");

    JsonDocument doc;
    doc["suhu"] = 28.5;              // Data suhu (℃)
    doc["kelembaban"] = 65.0;        // Data kelembaban (%)
    doc["waktu_ms"] = millis();      // BARIS BARU: waktu sejak NodeMCU menyala (ms)

    String requestBody;
    serializeJson(doc, requestBody);

    Serial.print("Mengirim data: ");
    Serial.println(requestBody);

    int httpResponseCode = http.POST(requestBody);

    if (httpResponseCode > 0) {
      Serial.print("Kode Response HTTP: ");
      Serial.println(httpResponseCode);
      Serial.println("Isi Response:");
      Serial.println(http.getString());
    } else {
      Serial.print("Pengiriman gagal, kode error: ");
      Serial.println(httpResponseCode);
    }

    http.end();
  }

  delay(10000);
}
```

```cpp
doc["waktu_ms"] = millis();
```

Penjelasan: `millis()` mengembalikan nilai `unsigned long` berupa jumlah milidetik sejak board terakhir kali menyala/reset. Nilai ini ditambahkan sebagai key baru `"waktu_ms"` ke dalam `JsonDocument` yang sama dengan `suhu` dan `kelembaban`, sehingga ikut diserialisasi menjadi bagian dari `requestBody` yang dikirim melalui HTTP POST. Data ini berguna sebagai penanda waktu **relatif** (bukan waktu nyata/real-world clock, karena ESP8266/ESP32 tidak memiliki RTC bawaan) untuk memverifikasi interval dan urutan pengiriman data pada sisi server.

---

## Percobaan 3B — Komunikasi MQTT

### Gambar Rangkaian

<img width="3024" height="4032" alt="Rangkaian 3A   3B" src="https://github.com/user-attachments/assets/73cad033-55b8-4bd5-b250-948263ff4d45" />

NodeMCU bertindak sebagai **publisher** yang mengirim data ke **broker** MQTT publik, sedangkan MQTT Explorer bertindak sebagai **subscriber** yang mengikuti topic yang sama untuk menerima data tersebut.

### Penjelasan Code

```cpp
const char* mqttServer = "broker.hivemq.com";
const int   mqttPort   = 1883;
const char* mqttTopic  = "unsoed/tk245004/kelompokAnda/sensor";

WiFiClient espClient;
PubSubClient client(espClient);
```
Konfigurasi alamat & port broker, nama topic (dibuat unik per kelompok), serta objek `PubSubClient` yang dibangun di atas koneksi TCP `WiFiClient`.

### Penjelasan Setiap Fungsi

**`hubungkanWiFi()`** — menghubungkan board ke jaringan WiFi, sama seperti pada Percobaan 3A, dipanggil sekali dari `setup()`.

**`hubungkanMQTT()`** — mencoba menghubungkan client ke broker MQTT:
- Membuat `clientId` acak (`"ESP8266Client-" + hex random`) agar tidak bentrok dengan client lain yang terhubung ke broker publik yang sama.
- Memanggil `client.connect(clientId.c_str())`; jika berhasil mencetak pesan sukses, jika gagal mencetak kode error (`client.state()`) lalu menunggu 2 detik sebelum mencoba lagi.

**`setup()`** — memanggil `hubungkanWiFi()` lalu `client.setServer(mqttServer, mqttPort)` untuk mendaftarkan alamat broker ke objek client.

**`loop()`**
- Memastikan koneksi MQTT aktif; jika terputus, memanggil `hubungkanMQTT()` untuk menyambung ulang.
- Memanggil `client.loop()` agar library memproses keep-alive dan pesan masuk/keluar.
- Menyusun data `suhu` dan `kelembaban` ke `JsonDocument`, mengubahnya menjadi `char buffer[]` dengan `serializeJson()`.
- Mempublikasikan buffer tersebut ke `mqttTopic` menggunakan `client.publish()`, lalu mencetak hasilnya ke Serial Monitor.
- `delay(5000)` menjeda 5 detik sebelum iterasi publish berikutnya.

### Penjelasan Percabangan/Conditional

| Kondisi | Fungsi |
|---|---|
| `while (WiFi.status() != WL_CONNECTED)` | Menahan program hingga WiFi tersambung, sama seperti pada Percobaan 3A. |
| `while (!client.connected())` (dalam `hubungkanMQTT()`) | Mengulang percobaan koneksi ke broker sampai berhasil, dengan jeda 2 detik antar percobaan agar tidak membanjiri broker dengan request. |
| `if (client.connect(...)) { ... } else { ... }` | Membedakan penanganan ketika koneksi ke broker berhasil (mencetak pesan sukses) dengan ketika gagal (mencetak kode error dan mencoba lagi). |
| `if (!client.connected())` (dalam `loop()`) | Mendeteksi jika koneksi MQTT terputus di tengah jalan, lalu memicu proses reconnect sebelum melanjutkan publish data. |

### Hasil Pengamatan

Board berhasil terhubung ke broker `broker.hivemq.com:1883` (ditandai pesan *"berhasil terhubung!"*), kemudian berhasil mempublikasikan data JSON `{"suhu":28.5,"kelembaban":65}` ke topic `unsoed/tk245004/kelompokAnda/sensor` secara berkala setiap 5 detik tanpa terputus.

<img width="1600" height="940" alt="Serial Monitor 3B" src="https://github.com/user-attachments/assets/f8920162-8562-40cd-9f2f-a3307b2c0b5b" />

Hasil ini sesuai dengan spesifikasi yang diharapkan: board terhubung ke broker publik, data terpublikasi secara berkala dalam format JSON, dan status koneksi/publikasi tampil jelas di Serial Monitor. Verifikasi tambahan pada MQTT Explorer yang subscribe ke topic yang sama menunjukkan data JSON muncul secara berkala dengan payload identik: `{"suhu":28.5,"kelembaban":65}`, sehingga integritas data end-to-end (publisher → broker → subscriber) terkonfirmasi dan koneksi MQTT tetap stabil selama pengujian.

### Jawaban Pertanyaan Praktikum 3B

**1. Apa fungsi dari topic pada protokol MQTT, dan mengapa topic yang digunakan perlu dibuat unik?**

Topic berfungsi sebagai "alamat" atau saluran virtual tempat publisher mengirim data dan subscriber menerima data; broker menggunakan topic untuk merutekan pesan ke subscriber yang tepat. Topic perlu dibuat unik (mis. menyertakan identitas kelompok) agar data yang dipublikasikan tidak tercampur atau bentrok dengan data kelompok/perangkat lain yang menggunakan broker publik yang sama.

**2. Jelaskan fungsi dari perintah client.loop() yang dipanggil pada setiap iterasi loop()!**

Dipanggil pada setiap iterasi `loop()` untuk menjaga koneksi MQTT tetap hidup (mengirim keep-alive/ping ke broker), memproses pesan masuk jika ada subscribe, serta menjalankan proses internal library `PubSubClient`. Tanpa pemanggilan rutin ini, koneksi dapat timeout dan terputus.

**3. Apa yang akan terjadi apabila koneksi ke broker MQTT terputus di tengah program berjalan?**

`client.connected()` akan bernilai `false`, sehingga pada iterasi `loop()` berikutnya program akan mendeteksi hal ini dan memanggil `hubungkanMQTT()` untuk mencoba menyambung ulang (reconnect) sampai berhasil. Selama belum tersambung kembali, `client.publish()` tidak akan berhasil mengirimkan data.

---

## Pertanyaan Analisis

**1. Uraikan hasil tugas pada praktikum yang telah dilakukan pada setiap percobaan!**

*Percobaan 3A (HTTP)*: NodeMCU 1.0 (ESP-12E) berhasil terhubung ke WiFi "rakjel" dan mengirim data JSON ke httpbin.org/post via HTTPS. Server merespons kode 200 dan mengembalikan (echo) data JSON yang dikirim di bagian "json". Serial Monitor menampilkan proses koneksi, data terkirim, kode response, dan isi response body. Tidak ada error. Program berulang tiap 10 detik. 

*Percobaan 3B (MQTT)*: NodeMCU berhasil terhubung ke broker.hivemq.com. Data JSON  dipublikasikan ke topic unsoed/tk245004/kelompokAnda/sensor setiap 5 detik. Data berhasil diverifikasi via MQTT Explorer yang subscribe ke topic sama. Serial Monitor menampilkan status koneksi dan setiap data yang dipublikasikan. Tidak ada error.

**2. Bandingkan besar overhead data dan pola komunikasi antara protokol HTTP dan MQTT berdasarkan hasil percobaan yang telah dilakukan!**

Pada percobaan ini, HTTP POST membawa header yang jauh lebih besar (terlihat pada echo respons: `Host`, `User-Agent`, `Content-Length`, `Accept-Encoding`, `X-Amzn-Trace-Id`, dll.) dan menggunakan pola request-response, setiap pengiriman data melakukan siklus koneksi baru (ditambah handshake TLS karena endpoint HTTPS). Sebaliknya, MQTT hanya mengirim payload JSON yang ringkas tanpa header HTTP, menggunakan pola publish-subscribe melalui satu koneksi *persistent* ke broker, sehingga overhead per pesan jauh lebih kecil dibandingkan HTTP.

**3. Untuk skenario pengiriman data sensor secara terus-menerus setiap beberapa detik dalam jangka waktu lama, protokol manakah (HTTP atau MQTT) yang lebih sesuai digunakan? Jelaskan alasannya!**

*MQTT* lebih sesuai, karena koneksinya bersifat persistent (tidak perlu membangun ulang koneksi setiap pengiriman), overhead per pesan kecil sehingga hemat daya dan bandwidth, serta mendukung banyak subscriber menerima data yang sama secara bersamaan — cocok untuk pengiriman data sensor setiap beberapa detik secara terus-menerus.

**4. Bagaimana peran format JSON dalam mendukung interoperabilitas data antara perangkat IoT dan berbagai platform/aplikasi yang berbeda?**

JSON adalah format berbasis teks yang independen terhadap bahasa pemrograman maupun platform, dan dapat di-parse dengan mudah oleh hampir semua bahasa (C++, Python, JavaScript, dll.) maupun framework. Hal ini membuat data yang dikirim dari mikrokontroler dapat dengan mudah dikonsumsi oleh berbagai aplikasi lain — dashboard web, aplikasi mobile, database — tanpa memerlukan format konversi khusus, sehingga sangat mendukung interoperabilitas antar perangkat dan platform pada sistem IoT.

---

## Kendala dan Solusi

| Kendala | Solusi |
|---|---|
| Board ESP32 DevKit yang disyaratkan modul tidak tersedia | Digunakan board alternatif NodeMCU 1.0 (ESP-12E, ESP8266) dengan pustaka `ESP8266WiFi.h` sebagai pengganti |
| Request HTTP POST gagal karena verifikasi sertifikat TLS (endpoint HTTPS) | Menggunakan `WiFiClientSecure` dengan `client.setInsecure()` untuk melewati verifikasi sertifikat (hanya untuk keperluan pengujian, bukan untuk produksi) |
| Koneksi awal ke broker MQTT publik terkadang butuh beberapa kali percobaan | Ditambahkan mekanisme retry pada `hubungkanMQTT()` dengan jeda 2 detik antar percobaan hingga berhasil |
| Pustaka ArduinoJson versi 7 tidak lagi mendukung `StaticJsonDocument` | Diganti menggunakan `JsonDocument` (API ArduinoJson v7) pada kedua sketch |
| Data MQTT awalnya tidak muncul di MQTT Explorer | Dipastikan subscribe menggunakan topic yang sama persis (case-sensitive) dengan `mqttTopic` pada kode ESP8266 |
| Port serial tidak otomatis terpilih | Port terdeteksi dan dipilih secara manual di COM4 pada Arduino IDE |

---

## KESIMPULAN 
1. Protokol komunikasi merupakan aturan yang mengatur pertukaran data antar perangkat dalam sistem IoT. 
2. HTTP menggunakan model request-response yang bersifat stateless, cocok untuk komunikasi periodik yang tidak terlalu sering. Pada percobaan ini, HTTP POST berhasil mengirim data JSON ke httpbin.org dengan response code 200. 
3. MQTT menggunakan pola publish-subscribe melalui broker, memiliki overhead kecil, dan cocok untuk komunikasi kontinu dengan koneksi persistent. Pada percobaan ini, data berhasil dipublikasikan ke broker.hivemq.com dan diverifikasi melalui MQTT Explorer. 
4. Format JSON memudahkan pertukaran data antar platform karena ringan, mudah dibaca, dan universal. Terbukti server httpbin.org dapat langsung memparsing JSON yang dikirim. 
5. NodeMCU 1.0 (ESP-12E) berbasis ESP8266 berhasil mengimplementasikan kedua protokol (HTTP POST dan MQTT publish) dengan format JSON. 
6. Untuk aplikasi IoT dengan pengiriman data sensor secara terus-menerus, MQTT lebih efisien dibandingkan HTTP karena overhead lebih kecil dan koneksi persistent.
