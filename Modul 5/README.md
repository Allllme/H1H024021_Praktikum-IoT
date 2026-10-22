# Modul 5 – Pengolahan dan Analisis Data IoT

| | |
|---|---|
| **Nama Mata Kuliah (Kode)** | Praktikum Internet of Things (TK245005) |
| **Tahun / Semester** | 2024 / 5 |
| **Modul** | 5 |
| **Nama Asisten** | Athallah Tsany Satriaji |
| **Nama Praktikan / NIM** | Alma Maida Wirastuti / H1H024021 |

Repository ini berisi dokumentasi dan source code hasil praktikum Modul 5, yang membahas tahapan pengolahan dan analisis data pada sistem IoT: **edge processing** (moving average & statistik min/max langsung di ESP8266) dan **server processing** (pengambilan data via MQTT dengan Python, penyimpanan ke CSV, visualisasi, dan deteksi anomali dengan z-score).

---

## Tujuan Praktikum
1. Memahami tahapan pengolahan dan analisis data pada sistem IoT.
2. Memahami perbedaan edge processing dan server processing.
3. Mengimplementasikan moving average serta statistik min/max pada ESP8266.
4. Mengambil data melalui MQTT menggunakan Python dan menyimpannya ke CSV.
5. Menganalisis, memvisualisasikan data, serta mendeteksi anomali dengan z-score.

## Alat dan Bahan

| Alat/Bahan | Keterangan |
|---|---|
| Board ESP8266 (NodeMCU) | 1 buah |
| Fungsi suhu *mock* | Pengganti sensor DHT22, diimplementasikan langsung di program (tanpa sensor fisik) |
| Breadboard & kabel jumper | Secukupnya |
| Kabel USB (Micro-USB) | Untuk pemrograman & power |
| Laptop/PC | Arduino IDE + Python 3 (VS Code) |
| Jaringan WiFi | Terhubung ke internet |
| Broker MQTT publik | `broker.hivemq.com` port `1883` |

## Library / Dependencies

**Arduino IDE (ESP8266)**

| Library | Fungsi |
|---|---|
| `ESP8266WiFi.h` | Bawaan board package ESP8266 – koneksi WiFi |
| `PubSubClient` (Nick O'Leary) | Komunikasi MQTT (connect, publish, subscribe) |
| `ArduinoJson` (Benoit Blanchon) | Membuat data berformat JSON |
| `math.h` | Fungsi matematika (`sin()`, `sqrt()`) |

**Python 3**

| Library | Fungsi |
|---|---|
| `paho-mqtt` | Client MQTT untuk subscribe data dari broker |
| `pandas` | Pengolahan data tabel & ekspor CSV |
| `matplotlib` | Visualisasi grafik suhu |
| `json`, `time` | Bawaan Python – parsing JSON & pencatatan waktu |

Instalasi library Python:
```bash
pip install paho-mqtt pandas matplotlib
```

## Gambar Rangkaian

<img width="1200" height="1600" alt="Rangkaian Percobaan 5" src="https://github.com/user-attachments/assets/3401e62d-6948-4ed9-9b8b-2f1d32620fa1" />

Rangkaian hanya berupa board ESP8266 di atas breadboard yang dihubungkan ke laptop menggunakan kabel USB (power + serial). Tidak ada sensor fisik karena data suhu dihasilkan oleh fungsi *mock*.

---

## Percobaan 5A – Edge Processing (Moving Average)

Pada percobaan ini ESP8266 menghasilkan data suhu simulasi, lalu **memfilternya dengan moving average (N = 5)** dan mencatat nilai **minimum/maksimum** langsung di perangkat (edge processing).

### Kode Program 5A

```cpp
#include <math.h> // Library untuk fungsi sin()

unsigned long jumlahBaca = 0; // Counter pembacaan

// Fungsi pengganti sensor DHT22
float bacaSuhuMock() {
  jumlahBaca++;
  float suhuDasar = 28.0 + 2.0 * sin(jumlahBaca * 0.08); // Pola suhu bertahap
  float noise = random(-5, 6) * 0.1;                     // Noise kecil
  float suhu = suhuDasar + noise;
  if (jumlahBaca % 25 == 0) suhu += 8.0;                 // Anomali tiap 25 data
  return suhu;
}

const int N = 5;        // Ukuran jendela moving average
float buffer[N];        // Buffer penyimpan data
int indeks = 0;         // Posisi tulis buffer
int jumlah = 0;         // Jumlah data terisi
float suhuMin = 1000.0;
float suhuMax = -1000.0;

float hitungMovingAverage(float nilaiBaru) {
  buffer[indeks] = nilaiBaru;     // Simpan data baru
  indeks = (indeks + 1) % N;      // Geser indeks melingkar
  if (jumlah < N) jumlah++;
  float total = 0;
  for (int i = 0; i < jumlah; i++) total += buffer[i];
  return total / jumlah;
}

void setup() {
  Serial.begin(115200);
  randomSeed(analogRead(A0)); // Seed acak
}

void loop() {
  float suhu = bacaSuhuMock();
  float suhuFilter = hitungMovingAverage(suhu);
  if (suhu < suhuMin) suhuMin = suhu;
  if (suhu > suhuMax) suhuMax = suhu;

  Serial.print("Mentah: ");  Serial.print(suhu, 2);
  Serial.print(" | Filter: "); Serial.print(suhuFilter, 2);
  Serial.print(" | Min: ");    Serial.print(suhuMin, 2);
  Serial.print(" | Max: ");    Serial.println(suhuMax, 2);
  delay(2000);
}
```

### Penjelasan Code
- **`#include <math.h>`** – diperlukan untuk fungsi `sin()` yang membentuk pola suhu bertahap.
- **Variabel global** – `jumlahBaca` (counter pembacaan), `buffer[N]` (penyimpan N data terakhir), `indeks` (posisi tulis), `jumlah` (banyaknya data terisi), `suhuMin`/`suhuMax` (nilai awal ekstrem agar pembacaan pertama pasti menggantikannya).
- **`const int N = 5`** – ukuran jendela moving average.

### Penjelasan Setiap Fungsi
- **`bacaSuhuMock()`** – menggantikan sensor DHT22. Suhu dasar `28 + 2·sin(jumlahBaca·0.08)` membentuk gelombang halus, ditambah *noise* acak ±0,5 °C (`random(-5, 6) * 0.1`), dan **lonjakan anomali +8 °C setiap pembacaan kelipatan 25**.
- **`hitungMovingAverage(nilaiBaru)`** – menyimpan data baru ke `buffer` secara melingkar (*circular buffer*), menjumlahkan seluruh isi buffer yang terisi, lalu mengembalikan rata-ratanya.
- **`setup()`** – mengaktifkan Serial (115200 baud) dan `randomSeed(analogRead(A0))` agar urutan noise acak berbeda tiap reset.
- **`loop()`** – membaca suhu mock, menghitung moving average, memperbarui min/max, mencetak ke Serial Monitor, lalu `delay(2000)`.

### Penjelasan Percabangan/Conditional

| Kondisi | Fungsi |
|---|---|
| `if (jumlahBaca % 25 == 0) suhu += 8.0;` | Menyisipkan anomali (lonjakan +8 °C) setiap 25 pembacaan sebagai data uji deteksi anomali. |
| `if (jumlah < N) jumlah++;` | Menghitung data terisi hingga maksimal N, sehingga rata-rata pada awal program (buffer belum penuh) dibagi dengan jumlah data yang benar, bukan N. |
| `if (suhu < suhuMin) suhuMin = suhu;` | Memperbarui suhu minimum jika ada pembacaan yang lebih rendah. |
| `if (suhu > suhuMax) suhuMax = suhu;` | Memperbarui suhu maksimum jika ada pembacaan yang lebih tinggi. |
| `for (int i = 0; i < jumlah; i++)` | Mengulang penjumlahan hanya pada elemen buffer yang sudah terisi. |

### Hasil Pengamatan

| No | Waktu (s) | Suhu Mentah (°C) | Moving Average (°C) | Min (°C) | Max (°C) |
|---|---|---|---|---|---|
| 1 | 2 | 28,14 | 28,30 | 27,86 | 38,32 |
| 2 | 4 | 27,48 | 28,08 | 27,48 | 38,32 |
| 3 | 6 | 27,32 | 27,80 | 27,32 | 38,32 |
| 4 | 8 | 27,37 | 27,70 | 27,32 | 38,32 |
| 5 | 10 | 27,21 | 27,51 | 27,21 | 38,32 |

![Serial Monitor 5A](images/serial-5a.png)

**Analisis:** *Suhu mentah* adalah keluaran fungsi mock sebelum difilter. *Moving average* dihitung dari maksimal lima pembacaan terakhir sehingga nilainya lebih halus dan tidak langsung mengikuti fluktuasi noise. *Min/Max* adalah nilai terendah dan tertinggi sejak program berjalan; suhu maksimum sempat mencapai **38,32 °C** akibat lonjakan anomali simulasi.

### Jawaban Pertanyaan Praktikum 5A

**1. Gambarkan diagram alur (flowchart) proses perhitungan moving average pada fungsi `hitungMovingAverage()`!**

<img width="910" height="3790" alt="deepseek_mermaid_20261021_879da4" src="https://github.com/user-attachments/assets/a611f88c-6c8b-48ff-b247-7432f8706432" />

Alur ringkas: Mulai → terima `nilaiBaru` → `buffer[indeks] = nilaiBaru` → `indeks = (indeks + 1) % N` → apakah `jumlah < N`? (Ya: `jumlah++`) → `total = 0` → loop `for i = 0..jumlah-1: total += buffer[i]` → `rata-rata = total / jumlah` → kembalikan nilai → Selesai.

**2. Jelaskan fungsi dari operasi `(indeks + 1) % N` pada program tersebut!**

Menjaga `indeks` tetap berada pada rentang 0 sampai N−1 (*circular buffer*). Ketika indeks mencapai N−1, operasi modulus mengembalikannya ke 0 sehingga penulisan buffer berputar dan menimpa data terlama, tanpa overflow/keluar batas array.

**3. Apa dampaknya terhadap data hasil filter apabila nilai N diperbesar menjadi 20 atau diperkecil menjadi 2?**

- **N = 20:** data lebih halus dan noise lebih teredam, tetapi respons terhadap perubahan sebenarnya menjadi lambat.
- **N = 2:** respons cepat, tetapi data lebih berfluktuasi karena noise kurang teredam.

**4. Modifikasi program agar ESP32 juga menghitung simpangan baku dari data pada buffer, dan berikan penjelasan di setiap baris kode yang ditambahkan.**

```cpp
#include <math.h>

const int N = 5;
float buffer[N];
int indeks = 0;
int jumlah = 0;

// ===== FUNGSI BARU =====
float hitungSimpanganBaku() {
  if (jumlah < 2) return 0.0;        // Minimal 2 data agar sebaran bisa dihitung
  float total = 0.0;                 // Penampung jumlah seluruh data buffer
  for (int i = 0; i < jumlah; i++) total += buffer[i];  // Jumlahkan data terisi
  float rataRata = total / jumlah;   // Rata-rata isi buffer

  float totalKuadrat = 0.0;          // Penampung jumlah kuadrat selisih
  for (int i = 0; i < jumlah; i++) {
    float selisih = buffer[i] - rataRata;   // Jarak data terhadap rata-rata
    totalKuadrat += selisih * selisih;      // Kuadratkan agar selalu positif
  }
  float varians = totalKuadrat / jumlah;    // Varians populasi
  return sqrt(varians);                     // Akar varians = simpangan baku
}

float hitungMovingAverage(float nilaiBaru) {
  buffer[indeks] = nilaiBaru;
  indeks = (indeks + 1) % N;
  if (jumlah < N) jumlah++;

  float total = 0.0;
  for (int i = 0; i < jumlah; i++) total += buffer[i];
  float rataRata = total / jumlah;

  float simpanganBaku = hitungSimpanganBaku();   // BARU: hitung std dev buffer

  Serial.print("Moving Average: "); Serial.print(rataRata, 2);          // BARU
  Serial.print(" | Simpangan Baku: "); Serial.println(simpanganBaku, 2); // BARU
  return rataRata;
}
```

**Penjelasan baris yang ditambahkan:**
- `if (jumlah < 2) return 0.0;` – simpangan baku tidak bermakna untuk 1 data, jadi dikembalikan 0.
- Loop pertama menghitung `rataRata` isi buffer, karena simpangan baku membutuhkan rata-rata sebagai acuan.
- Loop kedua menghitung `selisih = buffer[i] - rataRata` lalu mengkuadratkannya (`selisih * selisih`) dan menjumlahkannya ke `totalKuadrat`.
- `varians = totalKuadrat / jumlah` – membagi dengan jumlah data (varians populasi).
- `return sqrt(varians);` – akar varians adalah simpangan baku.
- Di `hitungMovingAverage()`, pemanggilan `hitungSimpanganBaku()` dan dua baris `Serial.print` menampilkan rata-rata dan simpangan baku ke Serial Monitor setiap pembacaan baru.

---

## Percobaan 5B – Akuisisi MQTT, Analisis, dan Deteksi Anomali

Pada percobaan ini ESP8266 **mempublikasikan** suhu mock dalam format JSON ke broker MQTT. Program Python di VS Code **men-subscribe** topic yang sama, menyimpan data ke CSV, menghitung statistik, mendeteksi anomali dengan **z-score**, dan membuat grafik (server processing).

**Alur sistem:** `ESP8266 (publisher)` → `broker.hivemq.com:1883` → `Python (subscriber)` → `CSV + analisis + grafik`

### Kode Program 5B – Arduino IDE (ESP8266)

> ⚠️ Ganti `SSID_ANDA` dan `PASSWORD_ANDA` dengan kredensial WiFi Anda. Jangan meng-commit password asli ke repository publik.

```cpp
#include <ESP8266WiFi.h>    // Library koneksi Wi-Fi ESP8266
#include <PubSubClient.h>   // Library komunikasi MQTT
#include <ArduinoJson.h>    // Library membuat data JSON
#include <math.h>           // Library matematika untuk sin()

// Konfigurasi Wi-Fi
const char* ssid     = "SSID_ANDA";
const char* password = "PASSWORD_ANDA";

// Konfigurasi broker dan topic MQTT
const char* mqttServer = "broker.hivemq.com";
const int   mqttPort   = 1883;                              // Port MQTT tanpa TLS
const char* topicData  = "unsoed/tk245004/kelompok6/data";  // Topic pengiriman data

WiFiClient espClient;
PubSubClient client(espClient);

unsigned long jumlahBaca = 0;

// Fungsi simulasi suhu sebagai pengganti sensor DHT22
float bacaSuhuMock() {
  jumlahBaca++;
  float suhuDasar = 28.0 + 2.0 * sin(jumlahBaca * 0.08);
  float noise = random(-5, 6) * 0.1;
  float suhu = suhuDasar + noise;
  if (jumlahBaca % 25 == 0) {
    suhu += 8.0;                     // Lonjakan suhu 8 derajat tiap 25 pembacaan
  }
  return suhu;
}

// Menghubungkan ESP8266 ke Wi-Fi
void setupWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  Serial.print("Menghubungkan Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWi-Fi berhasil terhubung!");
  Serial.print("Alamat IP: ");
  Serial.println(WiFi.localIP());
}

// Menghubungkan kembali ke broker MQTT jika terputus
void reconnectMQTT() {
  while (!client.connected()) {
    String clientId = "ESP8266-Mock-";
    clientId += String(ESP.getChipId(), HEX);   // ID klien unik dari chip ID
    Serial.println("Mencoba koneksi MQTT...");
    if (client.connect(clientId.c_str())) {
      Serial.println("MQTT berhasil terhubung!");
    } else {
      Serial.print("MQTT gagal, kode: ");
      Serial.println(client.state());
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  randomSeed(micros());
  setupWiFi();
  client.setServer(mqttServer, mqttPort);
  reconnectMQTT();
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) setupWiFi();   // Reconnect Wi-Fi
  if (!client.connected()) reconnectMQTT();         // Reconnect MQTT
  client.loop();                                    // Jaga komunikasi MQTT

  float suhu = bacaSuhuMock();

  StaticJsonDocument<128> doc;
  doc["suhu"] = suhu;
  char payload[128];
  serializeJson(doc, payload);

  bool berhasil = client.publish(topicData, payload);

  Serial.print("Pembacaan ke-");   Serial.print(jumlahBaca);
  Serial.print(" | Suhu mock: ");  Serial.print(suhu, 2);
  Serial.print(" °C | Topic: ");   Serial.print(topicData);
  Serial.print(" | Publish: ");    Serial.println(berhasil ? "Berhasil" : "Gagal");

  delay(5000);   // Kirim pembacaan berikutnya setelah 5 detik
}
```

### Kode Program 5B – Python (VS Code)

```python
import json                          # Mengolah data berformat JSON
import time                          # Mengambil waktu penerimaan data
import paho.mqtt.client as mqtt      # Library komunikasi MQTT
import pandas as pd                  # Mengolah data tabel
import matplotlib.pyplot as plt      # Membuat grafik

# Konfigurasi MQTT (harus sama dengan Arduino IDE)
BROKER = "broker.hivemq.com"
PORT = 1883
TOPIC = "unsoed/tk245004/kelompok6/data"
JUMLAH_SAMPEL = 30                   # Jumlah data yang akan diterima

data = []                            # Menampung waktu dan suhu

def on_connect(client, userdata, flags, reason_code, properties):
    print("Terhubung ke broker, kode:", reason_code)
    if reason_code == 0:             # 0 = koneksi berhasil
        client.subscribe(TOPIC)
        print("Subscribe topic:", TOPIC)
        print("Menunggu data suhu dari ESP8266...")

def on_message(client, userdata, msg):
    try:
        payload = json.loads(msg.payload.decode("utf-8"))   # JSON -> dict
        suhu = float(payload["suhu"])
    except (ValueError, KeyError, TypeError, json.JSONDecodeError):
        print("Data tidak valid, diabaikan.")
        return
    waktu = time.strftime("%Y-%m-%d %H:%M:%S")
    data.append({"waktu": waktu, "suhu": suhu})
    print(f"[{len(data)}/{JUMLAH_SAMPEL}] Waktu: {waktu} | Suhu: {suhu:.2f} °C")
    if len(data) >= JUMLAH_SAMPEL:
        client.disconnect()

# ===== Tahap 1: Akuisisi data melalui MQTT =====
client = mqtt.Client(callback_api_version=mqtt.CallbackAPIVersion.VERSION2)
client.on_connect = on_connect
client.on_message = on_message

print("Menghubungkan ke broker MQTT...")
try:
    client.connect(BROKER, PORT, keepalive=60)
    client.loop_forever()
except KeyboardInterrupt:
    print("\nPengambilan data dihentikan oleh pengguna.")
except Exception as error:
    print("Terjadi kesalahan koneksi MQTT:", error)

# ===== Tahap 2: Simpan & analisis data =====
if len(data) == 0:
    print("Tidak ada data yang diterima.")
    print("Periksa koneksi ESP8266, broker, dan topic MQTT.")
else:
    df = pd.DataFrame(data)
    rata2 = df["suhu"].mean()
    std = df["suhu"].std()

    if pd.isna(std) or std == 0:
        df["zscore"] = 0.0
    else:
        df["zscore"] = (df["suhu"] - rata2) / std

    df["anomali"] = df["zscore"].abs() > 2      # Anomali jika |z| > 2
    df.to_csv("data_suhu.csv", index=False)

    print("\n========= HASIL ANALISIS DATA =========")
    print(f"Jumlah data     : {len(df)}")
    print(f"Rata-rata       : {rata2:.2f} °C")
    print(f"Standar deviasi : {std:.2f}")
    print(f"Minimum         : {df['suhu'].min():.2f} °C")
    print(f"Maksimum        : {df['suhu'].max():.2f} °C")
    print(f"Jumlah anomali  : {int(df['anomali'].sum())}")
    print("Data tersimpan di data_suhu.csv")

    # ===== Tahap 3: Visualisasi =====
    plt.figure(figsize=(10, 4))
    plt.plot(df.index + 1, df["suhu"], marker="o", label="Suhu")
    anom = df[df["anomali"]]
    plt.scatter(anom.index + 1, anom["suhu"], color="red", s=100, label="Anomali", zorder=3)
    plt.axhline(rata2, color="gray", linestyle="-", label="Rata-rata")
    plt.xlabel("Urutan data")
    plt.ylabel("Suhu (°C)")
    plt.title("Data Suhu dan Deteksi Anomali")
    plt.legend()
    plt.tight_layout()
    plt.savefig("grafik_suhu.png")
    print("Grafik tersimpan di grafik_suhu.png")
    plt.show()
```

### Penjelasan Code

**Sisi ESP8266 (publisher)**
- Konfigurasi WiFi, broker (`broker.hivemq.com:1883`), dan topic `unsoed/tk245004/kelompok6/data`.
- `PubSubClient` dibangun di atas `WiFiClient` sebagai jalur TCP menuju broker.
- Data suhu dari fungsi mock dibungkus JSON `{"suhu": <nilai>}` lalu dipublikasikan setiap 5 detik.

**Sisi Python (subscriber & analisis)** – terbagi menjadi empat tahap:
1. **Akuisisi** – callback `on_connect` dan `on_message` menerima data dari broker.
2. **Penyimpanan** – data ditampung di `list`, diubah menjadi `DataFrame`, lalu diekspor ke `data_suhu.csv`.
3. **Analisis** – statistik deskriptif (rata-rata, standar deviasi, min, maks) dan deteksi anomali dengan z-score.
4. **Visualisasi** – grafik garis suhu, titik merah untuk anomali, dan garis rata-rata, disimpan sebagai `grafik_suhu.png`.

### Penjelasan Setiap Fungsi

| Fungsi | Penjelasan |
|---|---|
| `bacaSuhuMock()` | Menghasilkan suhu simulasi (sinus + noise + anomali tiap 25 data). |
| `setupWiFi()` | Mengatur mode station dan menghubungkan ESP8266 ke WiFi, menunggu hingga `WL_CONNECTED`. |
| `reconnectMQTT()` | Membuat `clientId` unik dari chip ID, mencoba `client.connect()` hingga berhasil; bila gagal mencetak `client.state()` dan menunggu 5 detik. |
| `setup()` | Inisialisasi Serial, seed acak, koneksi WiFi, set server MQTT, lalu konek ke broker. |
| `loop()` | Menjaga koneksi WiFi & MQTT, memanggil `client.loop()`, membuat JSON, publish, dan mencetak status. |
| `on_connect()` (Python) | Dipanggil saat terhubung ke broker; jika sukses, subscribe ke `TOPIC`. |
| `on_message()` (Python) | Dipanggil tiap pesan masuk; parsing JSON, ambil `suhu`, catat waktu, simpan ke list, dan putuskan koneksi setelah `JUMLAH_SAMPEL` tercapai. |

### Penjelasan Percabangan/Conditional

| Kondisi | Fungsi |
|---|---|
| `if (WiFi.status() != WL_CONNECTED)` | Menyambung ulang WiFi bila terputus. |
| `if (!client.connected())` | Menyambung ulang MQTT bila terputus. |
| `while (!client.connected())` / `if (client.connect(...))` | Mengulang percobaan koneksi ke broker dengan jeda 5 detik; membedakan pesan sukses dan gagal (disertai kode error). |
| `berhasil ? "Berhasil" : "Gagal"` | Menampilkan status hasil publish. |
| `if reason_code == 0` | Subscribe hanya jika koneksi ke broker berhasil. |
| `try ... except (...)` | Mengabaikan data yang formatnya tidak valid tanpa menghentikan program. |
| `if len(data) >= JUMLAH_SAMPEL` | Memutus koneksi setelah jumlah sampel terpenuhi. |
| `if len(data) == 0 ... else` | Menghentikan analisis bila tidak ada data yang diterima. |
| `if pd.isna(std) or std == 0` | Menghindari pembagian dengan nol saat menghitung z-score. |
| `df["zscore"].abs() > 2` | Menandai data sebagai anomali bila \|z\| > 2. |

### Hasil Pengamatan

Cuplikan 20 data pertama yang diterima Python:

| No. | Waktu Penerimaan | Suhu (°C) | Z-score | Status Anomali | Status CSV |
|---|---|---|---|---|---|
| 1 | 09:06:33 | 27.47 | -1.06 | Tidak | Tersimpan |
| 2 | 09:06:38 | 28.95 | -0.23 | Tidak | Tersimpan |
| 3 | 09:06:39 | 28.13 | -0.69 | Tidak | Tersimpan |
| 4 | 09:06:43 | 28.40 | -0.54 | Tidak | Tersimpan |
| 5 | 09:06:44 | 28.39 | -0.54 | Tidak | Tersimpan |
| 6 | 09:06:48 | 28.66 | -0.39 | Tidak | Tersimpan |
| 7 | 09:06:48 | 28.65 | -0.40 | Tidak | Tersimpan |
| 8 | 09:06:53 | 28.50 | -0.48 | Tidak | Tersimpan |
| 9 | 09:06:54 | 28.11 | -0.70 | Tidak | Tersimpan |
| 10 | 09:06:58 | 29.34 | -0.01 | Tidak | Tersimpan |
| 11 | 09:06:59 | 27.96 | -0.79 | Tidak | Tersimpan |
| 12 | 09:07:03 | 29.07 | -0.16 | Tidak | Tersimpan |
| 13 | 09:07:04 | 28.72 | -0.36 | Tidak | Tersimpan |
| 14 | 09:07:08 | 28.80 | -0.31 | Tidak | Tersimpan |
| 15 | 09:07:09 | 28.97 | -0.22 | Tidak | Tersimpan |
| 16 | 09:07:13 | 29.02 | -0.19 | Tidak | Tersimpan |
| 17 | 09:07:13 | 29.11 | -0.14 | Tidak | Tersimpan |
| **18** | **09:07:18** | **38.03** | **4.87** | **Ya** | Tersimpan |
| 19 | 09:07:19 | 29.05 | -0.17 | Tidak | Tersimpan |
| 20 | 09:07:23 | 29.52 | 0.09 | Tidak | Tersimpan |

**Ringkasan hasil analisis data:**

| Parameter | Hasil |
|---|---|
| Jumlah data | 30 |
| Rata-rata suhu | 29,36 °C |
| Standar deviasi | 1,78 |
| Suhu minimum | 27,47 °C |
| Suhu maksimum | 38,03 °C |
| Jumlah anomali | 1 |

![Serial Monitor 5B](images/serial-5b.png)
![Output Python](images/output-python.png)
![Grafik Suhu](images/grafik_suhu.png)

**Analisis:** Status anomali ditentukan dengan aturan \|z\| > 2. Pada data ke-18 terjadi lonjakan suhu ekstrem menjadi 38,03 °C dengan z-score **4,87**, sehingga terdeteksi sebagai anomali (lonjakan +8 °C dari fungsi mock). Seluruh data berhasil tersimpan ke `data_suhu.csv`. Seluruh data pengamatan ini berasal dari fungsi mock sebagai pengganti sensor fisik DHT22.

### Jawaban Pertanyaan Praktikum 5B

**1. Jelaskan fungsi dari blok try-except pada fungsi `on_message()` dan apa yang terjadi apabila blok tersebut dihilangkan!**

Blok ini menangani kesalahan saat mengubah JSON menjadi data Python atau saat mengambil atribut `suhu`. Jika data tidak valid, program menampilkan pesan lalu mengabaikan data tanpa berhenti. Tanpa try-except, satu data rusak dapat memunculkan exception yang menghentikan callback dan mengganggu proses penerimaan data.

**2. Jelaskan arti nilai z-score dan mengapa batas \|z\| > 2 digunakan sebagai penentu anomali!**

Z-score menyatakan jarak suatu nilai dari rata-rata dalam satuan standar deviasi. Nilai positif berarti di atas rata-rata, negatif berarti di bawah rata-rata. Batas \|z\| > 2 dipakai karena pada distribusi normal sekitar 95% data berada dalam rentang tersebut, sehingga nilai di luar rentang dianggap jarang dan berpotensi anomali.

**3. Apa kelemahan metode z-score apabila data memiliki tren naik atau turun yang tajam?**

Rata-rata dan standar deviasi ikut berubah mengikuti tren. Perubahan normal akibat tren bisa terdeteksi sebagai anomali, dan anomali nyata bisa tidak terdeteksi. Metode ini perlu *detrending* atau moving average sebelum diterapkan.

**4. Modifikasi program agar menggunakan moving average (jendela 5 data) pada data hasil CSV sebelum dihitung z-score-nya, kemudian bandingkan jumlah anomali yang terdeteksi, dan berikan penjelasan di setiap baris kode yang ditambahkan.**

```python
# Moving average jendela 5
df["suhu_ma"] = df["suhu"].rolling(window=5, min_periods=1).mean()

# Z-score data mentah
rata2_m = df["suhu"].mean()
std_m = df["suhu"].std()
df["z_mentah"] = 0.0 if (pd.isna(std_m) or std_m == 0) else (df["suhu"] - rata2_m) / std_m
df["anomali_mentah"] = df["z_mentah"].abs() > 2

# Z-score data hasil filter
rata2_ma = df["suhu_ma"].mean()
std_ma = df["suhu_ma"].std()
df["z_ma"] = 0.0 if (pd.isna(std_ma) or std_ma == 0) else (df["suhu_ma"] - rata2_ma) / std_ma
df["anomali_ma"] = df["z_ma"].abs() > 2

print("Anomali mentah:", int(df["anomali_mentah"].sum()))
print("Anomali setelah MA:", int(df["anomali_ma"].sum()))
df.to_csv("data_suhu.csv", index=False)
```

**Penjelasan baris yang ditambahkan:**
- `df["suhu_ma"] = df["suhu"].rolling(window=5, min_periods=1).mean()` – membuat kolom moving average jendela 5; `min_periods=1` membuat 4 data awal tetap dihitung dari data yang tersedia.
- `rata2_m`, `std_m` – rata-rata dan standar deviasi data **mentah**.
- `df["z_mentah"] = ...` – z-score data mentah; bernilai 0 jika std tidak valid/nol untuk menghindari pembagian dengan nol.
- `df["anomali_mentah"] = ... > 2` – penanda anomali pada data mentah.
- `rata2_ma`, `std_ma`, `df["z_ma"]`, `df["anomali_ma"]` – langkah yang sama, tetapi pada data **hasil moving average**.
- Dua `print` menampilkan jumlah anomali kedua metode agar dapat dibandingkan.
- `df.to_csv(...)` – menyimpan semua kolom baru ke CSV.

**Perbandingan jumlah anomali:**

| Metode | Jumlah Anomali |
|---|---|
| Data mentah | _(isi dari output `Anomali mentah:`)_ |
| Setelah moving average | _(isi dari output `Anomali setelah MA:`)_ |

> Secara teori, moving average meredam lonjakan sesaat (nilai ekstrem dirata-ratakan dengan 4 data di sekitarnya), sehingga z-score lonjakan menjadi lebih kecil dan jumlah anomali yang terdeteksi cenderung **berkurang** dibanding data mentah. Isi tabel di atas dengan hasil nyata dari program Anda.

---

## Pertanyaan Analisis

**1. Uraikan hasil tugas pada praktikum yang telah dilakukan pada setiap percobaan!**

- **Percobaan 5A:** ESP8266 berhasil menghitung moving average 5 data dan mencatat nilai min/max suhu mock, ditampilkan di Serial Monitor.
- **Percobaan 5B:** Data suhu berhasil dikirim via MQTT, diterima Python, disimpan ke CSV, dianalisis statistiknya, dideteksi anomalinya, dan divisualisasikan dalam grafik.

**2. Bandingkan pengolahan data di sisi perangkat (Percobaan 5A) dengan pengolahan data di sisi komputer (Percobaan 5B) dari segi kecepatan, kompleksitas analisis, dan kebutuhan sumber daya!**

Edge processing lebih cepat karena data diproses langsung di perangkat dan mengurangi data yang dikirim, tetapi kemampuan komputasinya terbatas. Server processing memiliki sumber daya besar untuk analisis kompleks dan visualisasi, tetapi bergantung pada koneksi dan menambah latensi.

**3. Bagaimana pengaruh pemilihan ukuran jendela moving average terhadap kualitas data dan kecepatan respons sistem kendali apabila data tersebut digunakan untuk mengendalikan aktuator?**

Jendela besar menghasilkan data lebih halus dan stabil, tetapi respons aktuator menjadi lambat terhadap perubahan. Jendela kecil membuat respons cepat, tetapi data lebih berisik sehingga aktuator bisa bekerja tidak stabil.

**4. Bagaimana hasil analisis dan deteksi anomali dapat dimanfaatkan untuk meningkatkan keandalan sistem IoT, misalnya pada sistem smart farming atau smart home?**

Pada smart farming, anomali suhu/kelembapan dapat menjadi peringatan dini kondisi lingkungan tidak normal. Pada smart home, anomali suhu dapat memicu penyesuaian aktuator atau notifikasi ke pengguna, sehingga keandalan sistem meningkat.

---

## Kendala dan Solusi

| Kendala | Solusi |
|---|---|
| Memastikan koneksi ESP8266, broker MQTT, dan Python di VS Code berjalan baik | Memeriksa ulang SSID/password WiFi, alamat broker, dan port, serta memastikan ESP8266 sudah publish sebelum Python dijalankan. |
| Data suhu yang diterima Python tidak sama dengan yang tampil di Serial Monitor Arduino IDE | Dipastikan topic MQTT identik (case-sensitive) di kedua sisi. Perbedaan dapat terjadi karena Python mulai subscribe di tengah proses publish (data sebelumnya tidak diterima) atau karena broker publik dipakai bersama sehingga topic perlu unik. |

## Kesimpulan

- Pengolahan data IoT dapat dilakukan di sisi perangkat (*edge*) maupun komputer/server, masing-masing dengan kelebihan dan keterbatasan.
- Moving average pada ESP8266 efektif meredam noise pada data suhu, dan ukuran jendela menentukan keseimbangan antara kehalusan data dan kecepatan respons.
- MQTT memudahkan pengiriman data dari mikrokontroler ke Python, yang kemudian dapat disimpan ke CSV, dianalisis, dan divisualisasikan.
- Metode z-score dengan batas \|z\| > 2 berhasil mendeteksi lonjakan suhu simulasi (38,03 °C, z = 4,87) sebagai anomali.
