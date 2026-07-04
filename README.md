# OBD2 Dashboard (Android)

Aplikasi dashboard mobil yang membaca data dari dongle OBD2 Bluetooth (ELM327) dan menampilkan:

- Speedometer (km/h)
- RPM
- Suhu mesin (°C)
- Level bensin (%)
- Beban mesin (%)
- Tegangan aki (V)
- Indikator check engine (MIL)
- Indikator sein kiri/kanan (lihat catatan penting di bawah)

## ⚠️ Catatan penting soal SEIN

Sein (turn signal) **bukan bagian dari data standar OBD-II (SAE J1979)**.
Data sein ada di jaringan CAN body control module kendaraan, dan hanya bisa
diakses lewat PID *enhanced/manufacturer-specific* (mode 22) yang berbeda-beda
untuk tiap merek dan model mobil — dongle ELM327 generik umumnya **tidak bisa**
membacanya.

Karena itu di aplikasi ini indikator sein dibuat sebagai **toggle manual/demo**
(tap ikon panah di UI). Kalau Anda tahu PID khusus untuk mobil Anda (biasanya
harus riset per merek, atau pakai tool seperti Torque Pro yang punya database
custom PID), Anda bisa tambahkan query serupa PID lain di `ObdManager.kt`.

## Cara build

1. Download / clone folder project ini.
2. Buka dengan **Android Studio** (Hedgehog/Koala ke atas disarankan).
3. Biarkan Gradle sync (butuh koneksi internet saat sync pertama kali untuk
   download dependency).
4. Sambungkan HP Android (USB debugging aktif) atau pakai emulator yang
   support Bluetooth (emulator biasanya TIDAK support Bluetooth asli, jadi
   disarankan pakai HP fisik).
5. Klik Run ▶.

## Cara pakai di HP

1. Colok dongle OBD2 Bluetooth (ELM327) ke port OBD2 mobil (biasanya di bawah
   setir).
2. Nyalakan kontak mobil (minimal posisi ON, tidak perlu starter).
3. Di HP, buka **Pengaturan > Bluetooth**, pairing dengan dongle (biasanya
   nama "OBDII" / "OBD2" / "V-LINK", PIN default sering `1234` atau `0000`).
4. Buka aplikasi ini, tekan tombol **Hubungkan**, pilih dongle dari daftar.
5. Data akan mulai update tiap ~250ms setelah berhasil konek.

## Struktur kode

- `MainActivity.kt` — UI, permission runtime, memilih & konek device Bluetooth.
- `ObdManager.kt` — komunikasi Bluetooth SPP ke ELM327, kirim perintah AT &
  PID OBD-II, parsing respons hex jadi nilai (speed, RPM, suhu, dll).
- `GaugeView.kt` — custom View untuk speedometer & RPM analog.
- `activity_main.xml` — layout dashboard.

## Protokol ECU

Aplikasi ini secara default **memaksa protokol `ISO 14230-4 KWP (fast init, 10.4 kbaud)`**
(kode ELM327 `ATSP5`) alih-alih auto-detect, karena banyak mobil (terutama produksi
Eropa/Asia era 2000-an awal) memakai protokol ini dan proses auto-detect ELM327
kadang lambat/gagal menebaknya.

Alurnya di `ObdManager.kt`:
1. Kirim `ATSP5` untuk memaksa protokol KWP fast init.
2. Kirim tes `0100` (daftar PID yang didukung) untuk memastikan ECU merespons.
3. Kalau ECU **tidak** merespons dengan protokol ini (`UNABLE TO CONNECT` /
   `NO DATA` / respons kosong), otomatis fallback ke `ATSP0` (auto-detect) dan
   memberi tahu lewat `onError()`.

Kalau mobil Anda ternyata pakai protokol lain (mis. ISO 15765-4 CAN yang umum di
mobil 2008 ke atas), tinggal ganti parameter saat memanggil `connect()` di
`MainActivity.kt`:

```kotlin
obdManager?.connect(device, ObdProtocol.ISO_15765_4_CAN_11B_500K)
```

Semua kode protokol yang didukung ada di enum `ObdProtocol` (sesuai datasheet ELM327,
perintah `ATSP0`–`ATSPA`).

## PID OBD-II yang dipakai

| Data          | PID   | Rumus                          |
|---------------|-------|---------------------------------|
| RPM           | 010C  | ((A*256)+B)/4                   |
| Speed         | 010D  | A (km/h)                        |
| Suhu mesin    | 0105  | A - 40 (°C)                     |
| Level bensin  | 012F  | A*100/255 (%)                   |
| Beban mesin   | 0104  | A*100/255 (%)                   |
| Check engine  | 0101  | bit 7 byte A                    |
| Tegangan aki  | ATRV  | perintah khusus ELM327 (bukan PID OBD) |

## Kemungkinan pengembangan lanjutan

- Simpan histori perjalanan (trip log) ke database lokal (Room).
- Tambah alarm suara kalau suhu mesin / RPM melewati batas.
- Dukungan Bluetooth LE untuk dongle OBD2 tipe BLE (bukan hanya SPP klasik).
- Tambah scan Diagnostic Trouble Code (DTC) — PID mode 03.
