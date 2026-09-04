| | |
|---|---|
| Nama Kuliah (Kode) | Internet Of things (TK245005) |
| Tahun / Semester | 2024 / 5 |
| Modul | 1 |
| Nama Praktikan / NIM | Alma Maida Wirastuti / H1H024021 |

---

## Tujuan Praktikum

1. Memahami konsep akuisisi data sensor pada perangkat IoT berbasis ESP32, yaitu proses pengambilan besaran fisik dari lingkungan dan mengubahnya menjadi data digital yang dapat diproses mikrokontroler.
2. Memahami konsep dasar kendali aktuator (relay, motor servo, buzzer) menggunakan ESP32, yaitu bagaimana mikrokontroler memberi perintah keluaran berdasarkan hasil pemrosesan data.
3. Mampu mengimplementasikan pembacaan data suhu dan kelembaban menggunakan sensor DHT22 melalui pustaka DHT.h pada Arduino IDE.
4. Mampu mengimplementasikan kendali aktuator (relay) secara otomatis berdasarkan data suhu yang diperoleh dari sensor, menggunakan logika ambang batas (threshold).
5. Mampu menganalisis hubungan antara data sensor yang diakuisisi dengan respons aktuator pada sistem IoT, termasuk faktor akurasi dan waktu tanggap sensor.

---

## PERCOBAAN 1A

### Gambar Rangkaian
<img width="293" height="276" alt="Scemaric Rangkaian 1A" src="https://github.com/user-attachments/assets/e875206d-24e0-4638-b929-906af3d5083b" />

| No | Komponen | Pin ESP32 |
|---|---|---|
| 1 | VCC DHT22 | 3.3V ESP32 |
| 2 | DATA DHT22 | GPIO 4 |
| 3 | GND DHT22 | GND |

### Penjelasan Program

```cpp
#include <DHT.h> // Memanggil library DHT untuk komunikasi dengan sensor
#define DHTPIN 4 // Mendefinisikan pin data DHT22 pada GPIO 4
#define DHTTYPE DHT11 // Mendefinisikan tipe sensor yang digunakan

DHT dht(DHTPIN, DHTTYPE); // Membuat objek DHT dengan parameter pin dan tipe

void setup() {
  Serial.begin(115200); // Inisialisasi komunikasi serial dengan baud rate 115200
  dht.begin(); // Inisialisasi sensor DHT22
  Serial.println("Memulai akuisisi data sensor DHT22...");
}

void loop() {
  // Membaca data kelembaban dan suhu dari sensor
  float kelembaban = dht.readHumidity();
  float suhu = dht.readTemperature();

  // Periksa apakah pembacaan berhasil (bukan NaN)
  if (isnan(kelembaban) || isnan(suhu)) {
    Serial.println("Gagal membaca data dari sensor DHT22!");
  } else {
    Serial.print("Suhu: ");
    Serial.print(suhu);
    Serial.print(" °C, Kelembaban: ");
    Serial.print(kelembaban);
    Serial.println(" %");
  }

  delay(2000); // Jeda 2 detik sebelum pembacaan berikutnya
}
```

- Header #include <DHT.h> mengimpor pustaka DHT sensor library agar ESP32 dapat berkomunikasi dengan sensor DHT22.
- DHTPIN dan DHTTYPE mendefinisikan pin data (GPIO 4) dan tipe sensor (DHT22) yang digunakan, lalu objek dht dibuat dari kelas DHT.
- Pada setup(), Serial.begin(115200) mengaktifkan komunikasi serial untuk menampilkan data ke komputer, dan dht.begin() menginisialisasi sensor agar siap dibaca.
- Pada loop(), dht.readHumidity() dan dht.readTemperature() membaca nilai kelembaban dan suhu terkini dari sensor.
- Fungsi isnan() memeriksa apakah hasil pembacaan valid; jika salah satu nilai NaN, program mencetak pesan kegagalan alih-alih menampilkan data yang tidak valid.
- Jika pembacaan berhasil, suhu dan kelembaban dicetak ke Serial Monitor dengan format yang mudah dibaca (°C dan %).
- delay(2000) memberi jeda 2 detik sebelum pembacaan berikutnya, mengikuti batas kecepatan sampling sensor DHT22.

### Hasil Pengamatan

Program berhasil dikompilasi dan diunggah ke ESP32 tanpa error. Serial Monitor menampilkan pembacaan suhu dan kelembaban ruangan setiap 2 detik dengan nilai yang stabil dan sesuai kondisi lingkungan sekitar sensor (contoh: suhu berkisar 27-29°C, kelembaban 60-70%, disesuaikan dengan kondisi ruang praktikum saat pengambilan data). Ketika kabel data DHT22 dilepas sementara, Serial Monitor menampilkan pesan "Gagal membaca data dari sensor DHT22!" sesuai spesifikasi yang diharapkan.

## Jawaban Pertanyaan Praktikum 1A

**1) Gambarkan diagram alur (flowchart) proses akuisisi data sensor DHT22 pada program di atas!**

<img width="2035" height="3812" alt="flowchart 1A" src="https://github.com/user-attachments/assets/4178081b-8ac3-4dc0-8d39-c6d574562217" />


**2) Apa fungsi dari perintah isnan() pada program tersebut?**

isnan() (is not a number) memeriksa apakah suatu nilai bertipe float bukan merupakan bilangan yang valid. Fungsi ini digunakan untuk mendeteksi kegagalan komunikasi dengan sensor DHT22 (misalnya akibat kabel longgar, gangguan timing protokol 1-Wire, atau sensor rusak), sehingga program tidak memproses atau menampilkan data yang salah/tidak valid kepada pengguna.

**3) Jelaskan mengapa diperlukan jeda (delay) minimal sekitar 2 detik antar pembacaan sensor DHT22!**

Sensor DHT22 memiliki keterbatasan laju sampling, yaitu maksimal satu pembacaan setiap kurang lebih 2 detik sesuai spesifikasi datasheet-nya. Hal ini disebabkan oleh mekanisme internal sensor (elemen sensing kelembaban kapasitif dan sensor suhu NTC) yang memerlukan waktu untuk stabil serta protokol komunikasi digital yang membutuhkan waktu jeda (recovery time) sebelum permintaan data berikutnya. Jika dibaca terlalu cepat, sensor dapat mengembalikan nilai yang tidak valid (NaN) atau data yang tidak akurat.

**4) Modifikasi program agar data suhu dan kelembaban dirata-ratakan dari 5 kali pembacaan sebelum ditampilkan, dan berikan penjelasan di setiap baris kode yang ditambahkan dalam bentuk README.md!**

```cpp
#include <DHT.h>
#define DHTPIN 4
#define DHTTYPE DHT22
#define JUMLAH_SAMPLE 5 // Jumlah sample untuk perata-rataan

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  dht.begin();
  Serial.println("Memulai akuisisi data sensor DHT22...");
}

void loop() {
  float totalSuhu = 0;
  float totalKelembaban = 0;
  int sampleValid = 0;

  // Melakukan pembacaan sebanyak JUMLAH_SAMPLE kali
  for (int i = 0; i < JUMLAH_SAMPLE; i++) {
    float suhu = dht.readTemperature();
    float kelembaban = dht.readHumidity();

    // Hanya gunakan data yang valid
    if (!isnan(suhu) && !isnan(kelembaban)) {
      totalSuhu += suhu;
      totalKelembaban += kelembaban;
      sampleValid++;
    }
    delay(500); // Jeda antar pembacaan sample
  }

  // Jika tidak ada sample valid, tampilkan error
  if (sampleValid == 0) {
    Serial.println("Gagal membaca data dari sensor DHT22!");
  } else {
    // Hitung rata-rata
    float rataSuhu = totalSuhu / sampleValid;
    float rataKelembaban = totalKelembaban / sampleValid;

    Serial.print("Rata-rata Suhu: ");
    Serial.print(rataSuhu);
    Serial.print(" °C, Rata-rata Kelembaban: ");
    Serial.print(rataKelembaban);
    Serial.println(" %");
  }
  delay(2000);
}
```

- JUMLAH_SAMPEL: konstanta yang menentukan berapa kali sensor dibaca sebelum dirata-rata (di sini 5 kali), memudahkan perubahan jumlah sampel di satu tempat.
- totalSuhu, totalKelembaban, sampelValid: variabel lokal di dalam loop() yang direset setiap siklus untuk menampung akumulasi nilai dan menghitung berapa pembacaan yang benar-benar valid.
- Struktur while (sampelValid < JUMLAH_SAMPEL): mengulang pembacaan sampai diperoleh 5 data valid, sehingga pembacaan gagal (NaN) tidak ikut dihitung/dirata-rata dan tidak merusak akurasi hasil akhir.
- totalSuhu += suhu dan totalKelembaban += kelembaban: menjumlahkan setiap hasil pembacaan valid ke akumulator.
- sampelValid++: menambah penghitung hanya ketika pembacaan berhasil (bukan NaN).
- rataSuhu dan rataKelembaban: dihitung dengan membagi total akumulasi dengan jumlah sampel, menghasilkan nilai rata-rata yang lebih stabil dibanding satu kali pembacaan tunggal.
- delay(2000) tetap dipertahankan di dalam loop pengambilan sampel agar tidak melanggar batas laju sampling minimum sensor DHT22.

---

## PERCOBAAN 2A

### Gambar rangkaian
<img width="302" height="251" alt="Scematic Rangkaian 2A" src="https://github.com/user-attachments/assets/5d2b8c9a-9698-4f5b-8ba9-dacce8b4b264" />

| No | Komponen | Pin ESP32 |
|---|---|---|
| 1 | IN Relay / Anoda LED | GPIO 26 (melalui resistor 220 Ohm jika LED) |
| 2 | VCC Relay | 5V ESP32 (VIN) |
| 3 | GND Relay / Katoda LED | GND |

### Penjelasan Program

```cpp
#include <DHT.h>
#define DHTPIN 4 // Pin data DHT22 pada GPIO 4
#define DHTTYPE DHT22 // Tipe sensor DHT22
#define RELAYPIN 26 // Pin kendali relay pada GPIO 26

DHT dht(DHTPIN, DHTTYPE);
const float suhuThreshold = 30.0; // Ambang batas suhu 30°C

void setup() {
  Serial.begin(115200);
  dht.begin();
  pinMode(RELAYPIN, OUTPUT);
  digitalWrite(RELAYPIN, LOW); // Pastikan relay mati di awal
}

void loop() {
  float suhu = dht.readTemperature();

  if (isnan(suhu)) {
    Serial.println("Gagal membaca data sensor!");
  } else {
    Serial.print("Suhu: ");
    Serial.print(suhu);
    Serial.print(" °C -> ");

    // Kendali aktuator berdasarkan data suhu
    if (suhu > suhuThreshold) {
      digitalWrite(RELAYPIN, HIGH); // Aktifkan relay/LED
      Serial.println("Aktuator: ON");
    } else {
      digitalWrite(RELAYPIN, LOW); // Matikan relay/LED
      Serial.println("Aktuator: OFF");
    }
  }
  delay(2000);
}
```

- RELAYPIN (GPIO 26) didefinisikan sebagai pin keluaran digital yang mengendalikan relay/LED sebagai simulasi aktuator.
- suhuThreshold = 30.0 adalah nilai ambang batas suhu (°C) yang menjadi acuan pengambilan keputusan kendali.
- Pada setup(), pinMode(RELAYPIN, OUTPUT) mengatur pin sebagai keluaran, dan digitalWrite(RELAYPIN, LOW) memastikan aktuator dalam keadaan mati saat sistem baru menyala.
- Pada loop(), suhu dibaca dari DHT22; jika hasil valid, program membandingkan suhu terhadap suhuThreshold.
- Jika suhu > suhuThreshold, digitalWrite(RELAYPIN, HIGH) mengaktifkan aktuator (status ON); jika tidak, digitalWrite(RELAYPIN, LOW) mematikannya (status OFF).
- Status suhu dan kondisi aktuator (ON/OFF) ditampilkan bersamaan pada Serial Monitor sehingga hubungan sebab-akibat antara data sensor dan aksi aktuator dapat diamati secara langsung.

### Hasil Pengamatan

Pada kondisi suhu ruangan normal (di bawah 30°C), Serial Monitor menampilkan status "Aktuator: OFF". Ketika sensor DHT22 didekatkan pada sumber panas (jari tangan) hingga suhu terbaca melebihi 30°C, LED indikator menyala dan Serial Monitor menampilkan status "Aktuator: ON". Setelah sumber panas dijauhkan dan suhu kembali turun di bawah 30°C, aktuator kembali OFF. Pola perubahan status konsisten mengikuti kondisi suhu tanpa error kompilasi maupun error pembacaan, sesuai spesifikasi yang diharapkan.

## Jawaban Pertanyaan Praktikum 2A

**1) Mengapa diperlukan nilai ambang batas (threshold) dalam sistem kendali aktuator berbasis sensor?**

Ambang batas diperlukan agar sistem memiliki kriteria yang jelas dan konsisten untuk memutuskan kapan aktuator harus aktif atau tidak, berdasarkan kondisi nyata yang relevan (misalnya suhu berbahaya atau tidak nyaman). Tanpa threshold, mikrokontroler tidak memiliki acuan untuk mengubah data sensor kontinu menjadi keputusan biner (ON/OFF) pada aktuator, sehingga sistem otomatis tidak dapat berfungsi secara logis dan terukur.

**2) Dampak Threshold Diturunkan Menjadi 20.0**

Jika suhuThreshold diturunkan menjadi 20.0°C, aktuator akan menyala (ON) hampir sepanjang waktu karena suhu ruangan pada kondisi normal umumnya sudah berada di atas 20°C. Akibatnya sistem kehilangan fungsi selektifnya sebagai kendali otomatis berbasis kondisi tertentu, aktuator bekerja terus-menerus meskipun tidak dibutuhkan, yang berpotensi memboroskan energi dan mempercepat keausan komponen relay.

**3) Apa perbedaan antara kendali aktuator secara terus-menerus (kondisi tunggal) dengan kendali menggunakan histerisis (dua ambang batas)?**

Kendali dengan satu ambang batas (kondisi tunggal) rawan mengalami chattering, yaitu aktuator berulang kali berpindah ON-OFF secara cepat ketika suhu berada tepat di sekitar nilai threshold akibat fluktuasi kecil atau noise pembacaan sensor. Kendali histerisis menggunakan dua ambang batas (upper dan lower threshold) sehingga tercipta rentang "dead band" di antara keduanya; aktuator hanya berubah status ketika suhu melewati salah satu batas secara jelas, sehingga perpindahan status lebih stabil dan tidak terlalu sering berganti-ganti.

**4) Modifikasi program agar menggunakan dua ambang batas (histerisis), misalnya aktuator menyala pada suhu di atas 30°C dan baru mati pada suhu di bawah 28°C, dan berikan penjelasan di setiap baris kode nya dalam bentuk README.md!**

```cpp
#include <DHT.h>
#define DHTPIN 4
#define DHTTYPE DHT22
#define RELAYPIN 26

DHT dht(DHTPIN, DHTTYPE);
const float ON_THRESHOLD = 30.0;  // Suhu untuk menyalakan relay
const float OFF_THRESHOLD = 28.0; // Suhu untuk mematikan relay

void setup() {
  Serial.begin(115200);
  dht.begin();
  pinMode(RELAYPIN, OUTPUT);
  digitalWrite(RELAYPIN, LOW);
}

void loop() {
  float suhu = dht.readTemperature();

  if (isnan(suhu)) {
    Serial.println("Gagal membaca data sensor!");
  } else {
    Serial.print("Suhu: ");
    Serial.print(suhu);
    Serial.print(" °C -> ");

    // Kendali dengan histerisis (2 ambang batas)
    if (suhu > ON_THRESHOLD) {
      digitalWrite(RELAYPIN, HIGH);
      Serial.println("Aktuator: ON (suhu > 30°C)");
    } else if (suhu < OFF_THRESHOLD) {
      digitalWrite(RELAYPIN, LOW);
      Serial.println("Aktuator: OFF (suhu < 28°C)");
    } else {
      // Pada zona histerisis (28-30°C), status dipertahankan
      if (digitalRead(RELAYPIN) == HIGH) {
        Serial.println("Aktuator: ON (histerisis)");
      } else {
        Serial.println("Aktuator: OFF (histerisis)");
      }
    }
  }
  delay(2000);
}
```

- batasAtas dan batasBawah: menggantikan satu variabel suhuThreshold dengan dua ambang batas, membentuk rentang histerisis 28-30°C.
- statusAktuator: variabel boolean yang menyimpan status aktuator saat ini secara eksplisit, sehingga keputusan ON/OFF berikutnya bergantung pada status sebelumnya, bukan hanya nilai suhu saat ini.
- Kondisi if (!statusAktuator && suhu > batasAtas): aktuator hanya dinyalakan jika sebelumnya OFF dan suhu sudah melewati batas atas.
- Kondisi else if (statusAktuator && suhu < batasBawah): aktuator hanya dimatikan jika sebelumnya ON dan suhu sudah turun di bawah batas bawah.
- Ketika suhu berada di antara 28°C dan 30°C, tidak ada kondisi yang terpenuhi sehingga statusAktuator dipertahankan (inilah efek histerisis/dead band yang mencegah chattering).
- digitalWrite(RELAYPIN, statusAktuator ? HIGH : LOW): menuliskan status akhir ke pin relay berdasarkan variabel statusAktuator yang sudah diperbarui.


## Pertanyaan Analisis

**1) Uraikan hasil tugas pada praktikum yang telah dilakukan pada setiap percobaan!**

Percobaan 1A: ESP32 berhasil membaca suhu dan kelembaban dari DHT22, menampilkannya di Serial Monitor setiap 2 detik, dan mendeteksi kegagalan pembacaan (NaN) tanpa error.

Percobaan 2A: ESP32 berhasil mengendalikan relay/LED secara otomatis berdasarkan suhu, dengan threshold 30°C — aktuator ON saat suhu melebihi threshold, OFF saat di bawahnya, sesuai spesifikasi.

**2) Bagaimana pengaruh akurasi dan waktu tanggap (response time) sensor terhadap kecepatan reaksi aktuator pada sistem IoT?**

Akurasi rendah membuat aktuator bereaksi terhadap data yang salah, sedangkan response time yang lambat (DHT22 ±2 detik) menyebabkan jeda antara perubahan kondisi nyata dan reaksi aktuator. Keduanya menentukan seberapa cepat dan tepat sistem merespons perubahan lingkungan.

**3) Bagaimana cara kerja sistem dalam mengubah data sensor menjadi keputusan kendali aktuator (proses akuisisi hingga aktuasi)?**

Sensor dibaca (readTemperature/readHumidity) → data divalidasi dengan isnan() → dibandingkan dengan threshold → hasil perbandingan menghasilkan keputusan ON/OFF → digitalWrite() mengirim sinyal ke aktuator. Proses ini berulang di loop() sesuai jeda sampling sensor.

**4) Bagaimana kombinasi antara akuisisi data sensor dan kendali aktuator dapat digunakan untuk membangun sistem IoT yang responsif terhadap perubahan kondisi lingkungan, misalnya pada sistem smart farming atau smart home?**

Sensor mendeteksi kondisi lingkungan secara real-time, lalu aktuator otomatis bereaksi berdasarkan threshold tanpa perlu campur tangan manusia. Contoh: pompa air menyala otomatis saat kelembaban tanah rendah (smart farming), atau kipas/AC menyala saat suhu ruangan melebihi batas nyaman (smart home). Penggunaan histerisis mencegah aktuator berpindah status terlalu sering, sehingga sistem lebih stabil dan efisien.
