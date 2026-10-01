🇮🇩 Bahasa Indonesia | [🇬🇧 English](README.en.md)

# ESP32 Secure OTA Firmware Assessment

[![ESP-IDF](https://img.shields.io/badge/ESP--IDF-v5.4-blue)](https://docs.espressif.com/projects/esp-idf/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

Sistem update firmware Over-The-Air (OTA) kelas produksi untuk ESP32 dengan mekanisme fail-safe, rollback otomatis, dan recovery mode.

## Fitur

- ✅ **Dual-Partition OTA**: layout factory + ota_0 + ota_1
- ✅ **Validasi 10 Detik**: Pengecekan stabilitas firmware otomatis
- ✅ **Rollback Otomatis**: Boot ke partisi sebelumnya jika crash
- ✅ **Recovery Mode**: Portal WiFi AP yang dipicu lewat GPIO
- ✅ **Konfigurasi WiFi**: Manajemen kredensial berbasis web
- ✅ **Indikator LED**: Umpan balik visual untuk status sistem
- ✅ **Verifikasi SHA256**: Validasi integritas firmware
- ✅ **Trigger OTA Manual**: Update firmware berbasis HTTP

## Kebutuhan Hardware

| Komponen | Spesifikasi |
|-----------|---------------|
| MCU | ESP32 (ESP32, ESP32-S3, ESP32-C3) |
| Flash | Minimal 4MB |
| LED | Bawaan atau eksternal (GPIO2) |
| Tombol | Pemicu recovery (GPIO4) |

## Konfigurasi Pin

| Fungsi | GPIO | Deskripsi |
|----------|------|-------------|
| LED | 2 | Indikator status (bawaan di kebanyakan DevKit) |
| Tombol Recovery | 4 | Tahan LOW saat reset untuk masuk recovery mode |

### Indikator LED

| Pola | Interval | Arti |
|---------|----------|---------|
| Kedip lambat | 1s ON / 1s OFF | Operasi normal |
| Kedip cepat | 200ms ON / 200ms OFF | Update OTA sedang berjalan |
| Kedip ganda | 2x kedip 100ms, jeda 800ms | Recovery mode aktif |

## Mulai Cepat

### Prasyarat
```bash
# Install ESP-IDF v5.4+
git clone -b v5.4 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
./install.sh
source export.sh
```

### Build & Flash
```bash
# Clone repository
git clone https://github.com/mjmokhtar/fota-esp32-idf
cd fota-esp32-idf

# Konfigurasi (opsional)
idf.py menuconfig

# Build
idf.py build

# Flash & Monitor
python -m esptool --chip esp32 -b 460800 --before default_reset --after hard_reset write_flash --flash_mode dio --flash_size 4MB --flash_freq 40m 0x1000 build/bootloader/bootloader.bin 0x8000 build/partition_table/partition-table.bin 0x10000 build/secure-ota-esp32.bin

python -m serial.tools.miniterm "COM4" 115200
```

### Boot Pertama

Setelah flashing, perangkat akan:
1. Boot ke partisi `factory`
2. Menginisialisasi WiFi (kredensial default ada di kode)
3. Menjalankan OTA HTTP server di port 80
4. LED berkedip lambat (mode normal)

## Panduan Penggunaan

### Operasi Mode Normal

1. Perangkat boot dan terhubung ke WiFi
2. Cari IP perangkat di serial monitor:
```
   I (xxx) WIFI_MGR: Got IP: 192.168.8.100
```
3. Akses portal OTA: `http://192.168.8.100`
4. Masukkan URL firmware lalu klik "Start Update"

### Recovery Mode

**Aktivasi:**
1. Hubungkan kabel jumper: GPIO4 → GND
2. Tekan tombol RESET
3. Tunggu pola LED kedip ganda
4. Lepas kabel jumper

**Penggunaan:**
1. Hubungkan ke WiFi AP: `ESP32-Recovery` / `recovery123`
2. Buka browser: `http://192.168.4.1`
3. Konfigurasi kredensial WiFi atau picu update OTA
4. Reboot perangkat

### Prosedur Update OTA

#### Langkah 1: Siapkan Firmware
```bash
# Build versi baru
idf.py build

# (Opsional) Tambahkan metadata untuk pelacakan
python tools/prepare-firmware.py   build/secure-ota-esp32.bin release/firmware_v2.0.0.bin 2.0.0
```

#### Langkah 2: Host Firmware
```bash
cd build/
python -m http.server 8000
```

#### Langkah 3: Picu Update

Lewat portal web, masukkan URL:
```
http://<IP_PC_KAMU>:8000/secure-ota-esp32.bin
```

#### Langkah 4: Pantau Update

Output serial:
```
I (xxx) OTA_MGR: Starting OTA update
I (xxx) OTA_MGR: Progress: 10% ... 100%
I (xxx) OTA_MGR: OTA successful! Rebooting...
```

Setelah reboot:
```
I (xxx) MAIN: New firmware detected, validating...
[10 second wait - LED fast blink]
I (xxx) MAIN: Firmware validated successfully!
I (xxx) MAIN: Running from partition: ota_0
```

## Arsitektur

### Layout Partisi
```
┌─────────────────────┐ 0x0000
│   Bootloader        │ 32KB
├─────────────────────┤ 0x8000
│   Partition Table   │ 4KB
├─────────────────────┤ 0x9000
│   NVS               │ 24KB
├─────────────────────┤ 0xF000
│   PHY Init          │ 4KB
├─────────────────────┤ 0x10000
│   Factory (App)     │ 1MB
├─────────────────────┤ 0x110000
│   OTA_0             │ 1MB
├─────────────────────┤ 0x210000
│   OTA_1             │ 1MB
└─────────────────────┘
```

### State Machine
```
┌─────────┐
│  BOOT   │
└────┬────┘
     │
     v
┌─────────────────┐      ┌──────────────┐
│ GPIO4 == LOW?   ├─YES─→│ RECOVERY     │
└────┬────────────┘      │ MODE         │
     NO                   └──────────────┘
     │
     v
┌─────────────────┐
│ Check Partition │
│ State           │
└────┬────────────┘
     │
     ├─ PENDING_VERIFY
     │  ↓
     │  ┌──────────────┐
     │  │ 10s Wait     │
     │  └──┬───────────┘
     │     │
     │     ├─ Success → Mark Valid
     │     └─ Crash → Rollback
     │
     └─ VALID
        ↓
    ┌──────────────┐
    │ NORMAL MODE  │
    └──────────────┘
```

Untuk arsitektur lebih rinci, lihat [ARCHITECTURE.md](docs/ARCHITECTURE.md)


## Pemecahan Masalah

### Masalah: Perangkat terjebak di download mode
**Solusi:** Pastikan GPIO0 (tombol BOOT) tidak ditekan saat boot normal

### Masalah: Koneksi WiFi gagal
**Solusi:** Gunakan recovery mode untuk mengonfigurasi ulang kredensial

### Masalah: OTA gagal dengan "invalid magic byte"
**Solusi:** Pastikan memakai biner mentah (`secure-ota-esp32.bin`), bukan versi yang sudah lewat prepare

### Masalah: Recovery mode tidak terpicu
**Solusi:** Pastikan GPIO4 dalam kondisi LOW sebelum menekan RESET, tahan sampai LED kedip ganda

## Struktur Project
```
firmware-assessment-esp32/
├── main/
│   ├── main.c              # Logika aplikasi utama
│   ├── led_indicator.c/h   # Kontrol LED
│   ├── wifi_manager.c/h    # WiFi & NVS
│   ├── ota_manager.c/h     # Implementasi OTA
│   ├── recovery_mode.c/h   # Portal recovery
│   └── CMakeLists.txt
├── tools/
│   └── prepare-firmware.py # Tool metadata firmware
├── docs/
│   ├── ARCHITECTURE.md     # Keputusan desain
│   └── prompt.md           # Log bantuan AI
├── partitions.csv          # Tabel partisi
├── CMakeLists.txt
└── README.md
```

## Pengembangan

### Gaya Kode

- Ikuti konvensi penulisan kode ESP-IDF
- Gunakan makro ESP_LOG untuk logging
- Tambahkan penanganan error untuk semua operasi
- Dokumentasikan kode yang tidak jelas maksudnya

## Penulis

**Muhammad Jumi'at Mokhtar** - Firmware Assessment Submission

## Ucapan Terima Kasih

- ESP-IDF oleh Espressif Systems
- Desain assessment oleh MJ Mokhtar
