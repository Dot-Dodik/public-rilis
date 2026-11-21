# cmkagent-rust — Universal Rust Monitoring Agent (Linux, Windows & Android APK)

**cmkagent-rust** adalah agen monitoring multi-platform ultra-ringan berbasis **Rust** yang menggabungkan Checkmk TCP Pull (`0.0.0.0:6556`), HTTP(S) Push Backup, serta seluruh *local checks* kustom dalam **satu binary tunggal** (tanpa perlu instalasi skrip `.sh` / `.ps1` terpisah dan tanpa memunculkan *unmonitored services* saat di-pull).

Dibangun dengan menerapkan prinsip **Final Skill** (*Think Before Coding*, *Simplicity First*, *Surgical Changes*, *Goal-Driven Execution*) dan **15 Lapisan Production-Ready Engineering**.

---

## 1. Struktur Cargo Workspace

```text
cmkagent-rust/
├── Cargo.toml                        # Workspace root (opt-level="z", lto=true, strip=true, panic="abort")
├── README.md                         # Panduan utama instalasi & operasional (Linux, Windows, Android)
├── about.md                          # Dokumentasi khusus arsitektur & bahasa pemrograman APK Android (Java + Rust JNI)
├── crates/
│   ├── cmkagent-core/                # Engine protokol Checkmk, TCP/6556 server, Ping 4x/1m (ping_loss_1m + 1h) & 30d log
│   ├── cmkagent-desktop/             # Binary native untuk Linux & Windows Service (+ Static CRT Windows)
│   │   └── src/
│   │       ├── tls/                  # Native Checkmk 2.5 mTLS engine (rustls + ring, CSR PKCS#10, auto-renew, tls.log)
│   │       ├── updater.rs            # Auto-updater binary 4 jam (SHA-256 + magic header MZ/ELF + random jitter)
│   │       ├── linux/                # Collector native Linux (/proc, /sys, smartctl, HDSentinel, dmidecode)
│   │       └── windows/              # Collector native Windows (Win32 API, HWiNFO VSB, CrystalDiskInfo, WMI)
│   └── cmkagent-android-jni/         # Library JNI (`libcmkagent_jni.so`) untuk Android APK
└── android/                          # Aplikasi Android (Java 17 + Rust JNI, SDK 26–35)
```

---

## 2. Daftar Service Eksak (Tanpa Perlu Memilah / Zero Unmonitored Clutter)

Ketika server Checkmk atau **BMS (Borwita Monitoring Service)** menarik data dari agen ini, agen **hanya** mengeluarkan section dan local check berikut sehingga `Check_MK Discovery` langsung bersih (`Services: all up to date`):

### A. Linux Agent (`cmkagent` — 18 Services)
| No | Nama Service | Jadwal / Cache | Keterangan & Format Output |
| :--- | :--- | :--- | :--- |
| 1 | `Check_MK` | Real-time (60s) | Status eksekusi agent & waktu respons |
| 2 | `Check_MK Agent` | Real-time (60s) | Versi agen & informasi OS |
| 3 | `Check_MK Discovery` | Otomatis | Seluruh service langsung terpantau (0 *unmonitored*) |
| 4 | `Check_MK HW/SW Inventory` | Cache 4 Jam | Inventaris CPU, RAM, Disk, Distro, & Paket (`dpkg`/`rpm`) |
| 5 | `CPU load` | Real-time (60s) | Beban rata-rata 1, 5, 15 menit per core (`<<<cpu>>>`) |
| 6 | `CPU utilization` | Real-time (60s) | Persentase utilisasi CPU (`<<<kernel>>>` terfilter hanya baris `cpu*`) |
| 7 | `Uptime` | Real-time (60s) | Lama sistem menyala (`<<<uptime>>>`) |
| 8 | `FAN_Health` | Real-time (60s) | RPM kipas (`hwmon` / `thinkpad_acpi` / `sensors`) + korelasi suhu CPU & auto-fallback `EC/Auto` |
| 9 | `Health_Battery` | Harian 16:00 | Kapasitas desain vs aktual (`w/h`), persentase kesehatan, & deteksi baterai drop/mati (`0%` di AC) |
| 10 | `Health_RAM` | Sabtu 11:00 | Jumlah slot terpakai/kosong (`dmidecode -t 17`), modul aktif, & hasil uji `memtester` (20% Free RAM) |
| 11 | `Health_Storage (<Model>)` | Harian 16:00 | Hybrid `HDSentinel` + `smartctl` (NVMe / SSD Sata / HDD Mekanik, TBW, Suhu, Bad Sector, POH) |
| 12 | `Info_CPU` | Real-time (60s) | Model CPU, Clock Speed (`Ghz`), Core/Thread fisik/logis, Load `%`, & Suhu `°C` (Perf-O-Meter suhu) |
| 13 | `Info_Network_<Iface>` | Real-time (60s) | IPv4, Total Download/Upload, serta laju `RX Rate` & `TX Rate` per detik |
| 14 | `Info_Office` | Harian 16:00 | Deteksi otomatis LibreOffice, WPS Office, & OnlyOffice (`dpkg`/`rpm`/Flatpak/Snap) |
| 15 | `Info_OS` | Harian 16:00 | Nama distro (`PRETTY_NAME`), versi kernel, & arsitektur |
| 16 | `Info_RAM_Usage` | Real-time (60s) | Persentase RAM terpakai, Used GB, Free GB, & Total GB (`WARN` 85%, `CRIT` 95%) |
| 17 | `Info_Remote_Support` | Harian 16:00 | ID AnyDesk & ID RustDesk (CLI, Flatpak, `/etc/rustdesk_id.txt`, `RustDesk.toml`) |
| 18 | `Ping_192.168.122.10`, `Ping_8.8.8.8`, `Ping_Gateway` | 4x/menit (1m & 1h) | Grafik real-time `Ping loss 1m` (`ping_loss_1m` 0–100%) + ringkasan 1 jam (`pl`, `rta`) + histori putus 24 jam (*terbaru dulu*) & log 30 hari |
| 19 | `Storage_Usage_<Mount>` | Harian 16:00 | Pemakaian partisi fisik (`Storage_Usage_root`, `Storage_Usage_mnt_...`) tanpa partisi virtual/snap |

---

### B. Windows Agent (`cmkagent.exe` — 20+ Services)
| No | Nama Service | Jadwal / Cache | Keterangan & Format Output |
| :--- | :--- | :--- | :--- |
| 1 | `Check_MK`, `Check_MK Agent`, `Check_MK Discovery`, `Check_MK HW/SW Inventory` | Otomatis | Built-in Checkmk status & inventaris hardware/software Windows |
| 2 | `CPU utilization` | Real-time (60s) | Utilisasi CPU Windows (`<<<winperf_processor>>>`) |
| 3 | `Memory` | Real-time (60s) | Penggunaan RAM & Pagefile (`<<<mem>>>`) |
| 4 | `Uptime` | Real-time (60s) | Lama sistem menyala (`<<<uptime>>>` via `GetTickCount64`) |
| 5 | `Windows License` | Harian 16:00 | Status aktivasi lisensi Windows berbahasa Inggris (`<<<win_license>>>`) |
| 6 | `CPU_Info` | Real-time (60s) | Spesifikasi CPU, Clock Speed, Core/Thread, Load `%`, & Suhu `Celcius` (HWiNFO `HKEY_USERS` / LHM / WMI) |
| 7 | `FAN_Health` | Real-time (60s) | Kecepatan RPM kipas (HWiNFO `HKEY_USERS` VSB 0..99 / LHM / WMI) + fallback `EC/Auto` |
| 8 | `Health_Battery` | Harian 16:00 | `powercfg /batteryreport /xml` + WMI (`w/h`), persentase kesehatan, & deteksi baterai drop/mati |
| 9 | `Health_RAM` | Sabtu 11:00 | Slot RAM (`Win32_PhysicalMemory`), ukuran sampel 20% Free RAM, & status `memtester_health.log` |
| 10 | `Info_Network_<NIC>` | Real-time (60s) | IPv4, Total Download/Upload, `RX Rate` & `TX Rate` untuk adapter fisik yang `Up` |
| 11 | `Info_Office` | Harian 16:00 | Multi-lisensi MS Office (`OSPP.VBS` + versi ClickToRun + 5 karakter Product Key) + LibreOffice + OnlyOffice |
| 12 | `Info_OS` | Harian 16:00 | Edisi Windows 10/11, pemetaan tahun (`25H2` -> `Windows 11 2025 (25H2)`), Build, Status Lisensi & Key |
| 13 | `Ping_192.168.122.10`, `Ping_8.8.8.8`, `Ping_Gateway` | 4x/menit (1m & 1h) | Grafik real-time `Ping loss 1m` (`ping_loss_1m` 0–100%) + ringkasan 1 jam (`pl`, `rta`) + histori putus 24 jam (*terbaru dulu*) & log 30 hari |
| 14 | `RAM_Usage` | Real-time (60s) | Persentase RAM terpakai, Used Space GB, Free GB, & Total GB |
| 15 | `Remote_Access_ID` | Harian 16:00 | `OK: AnyDesk: <ID> \| RustDesk: <ID> \| AnyViewer: <ID>` |
| 16 | `Storage_Health_<Model>` | Harian 16:00 | Prioritas CrystalDiskInfo `diskinfo.txt` (menembus Intel RST/VMD RAID) -> `smartctl` -> `StorageReliabilityCounter` |
| 17 | `Storage_Usage_<Drive>` | Harian 16:00 | Pemakaian kapasitas per drive tetap (`Storage_Usage_C`, `Storage_Usage_D`, dst.) |

---

### C. Android APK Agent (`cmkagent-android-release.apk` — 14 Services + Interactive UI)
> Dokumentasi lengkap arsitektur internal & bahasa pemrograman khusus Android (Java 17 + Rust JNI) tersedia di **[about.md](about.md)**.

- **Unduh Langsung APK Terbaru (`auto-latest`):**
  `https://github.com/Dot-Dodik/public-rilis/releases/download/auto-latest/cmkagent-android-release.apk`
- **Service yang dikirim ke Server:**
  `Check_MK`, `Check_MK Agent`, `Check_MK Discovery`, `Check_MK HW/SW Inventory`, `Agent_Status`, `Android_Info`, `Health_Battery` (1-baris lengkap dengan mAh, siklus, suhu, voltase, arus), `Ping_8.8.8.8`, `Ping_192.168.122.10`, `Ping_Gateway` (dilengkapi grafik real-time `Ping loss 1m` 0–100% di samping grafik `Packet loss` 1 jam & `Round trip average`, daftar rentang jam putus 24 jam terbaru dulu, serta penyimpanan otomatis IP gateway terakhir saat Wi-Fi drop), `RAM_Usage`, `Storage_Usage`, `Transport_Status`, dan `WiFi_Status` (termasuk deteksi perpindahan Roaming AP BSSID 1 jam terakhir).
- **Fitur Tampilan Aplikasi Android:**
  - **Tombol `▶ START AGENT` & `■ STOP AGENT`:** Menyalakan atau menghentikan *foreground service* dan listener TCP/6556 kapan saja langsung dari aplikasi.
  - **Tombol `EXPAND ALL` & `COLLAPSE ALL`:** Membuka atau menutup seluruh kartu menu sekaligus dengan satu sentuhan.
  - **Expand & Collapse per Menu:** Setiap kartu (`AGENT`, `HYBRID TRANSPORT`, `BATTERY`, `SYSTEM`, `NETWORK, AP ROAMING & PING`, `APPLICATIONS`, `DEVICE`, `SETTINGS`) dapat diketuk pada bagian judulnya untuk menampilkan atau menyembunyikan rincian teknis, sementara ringkasan 1-baris tetap terlihat saat kartu ditutup (*collapsed*).

---

## 3. Daftar Perbaikan dari Skrip Lama ke Native Rust

1. **Perbaikan Satuan Metrik Ping (`rta` dalam detik):**
   Mengonversi nilai `rta`, `rtmin`, dan `rtmax` dari milidetik ke detik (`ms / 1000.0`, contoh: `rta=0.0289;0.100;0.250;0`) agar *Perf-O-Meter* bawaan Checkmk menampilkan angka milidetik yang benar (tidak dikali 1.000 menjadi detik).
2. **Penghapusan 17 Unmonitored Services di Linux:**
   Menghilangkan section berlebih (`<<<df>>>`, `<<<mounts>>>`, `<<<lnx_if>>>`, `<<<lnx_thermal>>>`, `<<<tcp_conn_stats>>>`, `<<<systemd_units>>>`) dan memfilter `<<<kernel>>>` agar hanya mengirim baris `cpu*` (mencegah munculnya `kernel_performance` tak terpantau).
3. **Perbaikan Bug `$sample_size_raw` pada `ram_health.sh`:**
   Pada skrip bash lama, variabel `$sample_size_raw` belum didefinisikan sebelum dipanggil sehingga `Tested Size` menjadi kosong. Di Rust, ukuran sampel `20% Free RAM` dihitung dan diparse secara langsung.
4. **Perbaikan Konflik `remote_apps.ps1` vs `remote_access_id.ps1` di Windows:**
   Memisahkan output menjadi dua service sesuai standar dashboard Checkmk: `Remote_Access_ID` (AnyDesk/RustDesk/AnyViewer ID) dan `Remote_Apps` (pendeteksi aplikasi remote tidak sah).
5. **Dukungan Intel RST / VMD RAID via CrystalDiskInfo (`diskinfo.txt`):**
   Membaca file `C:\ProgramData\checkmk\agent\log_custom\diskinfo.txt` (mendukung encoding UTF-16LE maupun UTF-8) sebelum fallback ke `smartctl` dan WMI, serta menghapus `Write-Host "<<<local>>>"` ganda dan pemotongan nama model 45 karakter yang ada di skrip lama.
6. **Pembacaan Sensor HWiNFO Lintas Session (`HKEY_USERS`):**
   Karena Windows Service berjalan sebagai `LocalSystem` (Session 0), agen Rust memindai seluruh hive `HKEY_USERS\<SID>\Software\HWiNFO64\VSB` milik user yang sedang login untuk memperoleh suhu CPU dan RPM kipas secara akurat.

---

## 4. Panduan Instalasi & Perintah CLI

### A. Linux (Debian / Ubuntu / Mint / RHEL / Rocky / AlmaLinux)
Semua metode instalasi di bawah melakukan **Clean Replace otomatis** (menghentikan/menghapus paket `check-mk-agent` bawaan, service/socket lama, cron job, serta skrip `.sh` lama) dan langsung mengaktifkan `cmkagent.service` + Auto-Update 4 Jam dari GitHub + logo **BMS**.

#### 1. Install Langsung 1-Baris dari Terminal (Tersambung ke GitHub)

Mode interaktif (menanyakan konfirmasi instalasi `smartmontools`, `lm-sensors`, `HDSentinel`, `memtester`, `dmidecode`):
```bash
curl -fsSL https://github.com/Dot-Dodik/public-rilis/releases/download/auto-latest/install-linux.sh | sudo bash
```
Mode otomatis penuh tanpa tanya (*unattended*):
```bash
curl -fsSL https://github.com/Dot-Dodik/public-rilis/releases/download/auto-latest/install-linux.sh | sudo bash -s -- --auto-tools
```

#### 2. Install via Paket `.deb` (Debian / Ubuntu / Linux Mint)

Unduh `.deb` terbaru dari GitHub Releases (`auto-latest`):
```bash
curl -fsSL -O https://github.com/Dot-Dodik/public-rilis/releases/download/auto-latest/cmkagent-linux-amd64.deb
```

Pasang paket `.deb`:
```bash
sudo apt-get update && sudo apt-get install -f -y ./cmkagent-linux-amd64.deb
```

#### 3. Install via Paket `.rpm` (RHEL / Rocky / AlmaLinux / Fedora)

Unduh `.rpm` terbaru dari GitHub Releases (`auto-latest`):
```bash
curl -fsSL -O https://github.com/Dot-Dodik/public-rilis/releases/download/auto-latest/cmkagent-linux-x86_64.rpm
```

Pasang paket `.rpm` menggunakan `dnf`:
```bash
sudo dnf install -y ./cmkagent-linux-x86_64.rpm
```

Atau menggunakan `rpm`:
```bash
sudo rpm -Uvh ./cmkagent-linux-x86_64.rpm
```

#### 4. Install dari Binary Tunggal (`install` subcommand)

```bash
chmod +x ./cmkagent-linux-x86_64 && sudo ./cmkagent-linux-x86_64 install --auto-tools
```

#### 5. Cek Status, Uji Output, & Update Manual (Linux)

Cek status service & pemakaian RAM:
```bash
systemctl status cmkagent
```

Uji payload output agen:
```bash
sudo cmkagent dump --config /etc/cmkagent/cmkagent.toml
```

Paksa cek & terapkan update binary terbaru dari GitHub sekarang:
```bash
sudo cmkagent update-check --config /etc/cmkagent/cmkagent.toml && sudo systemctl restart cmkagent
```

---

### B. Windows (Windows 10 / 11 / Server)
Seluruh installer Windows menjalankan **Clean Replace otomatis** terhadap service Checkmk lama (`CheckMkWatchdog`, `CheckMKService`, `CheckmkAgent`), task `Checkmk_Auto_Update` lama, serta langsung memasang **BMS Monitoring Agent** (`Agen BMS : 1.0.2`, lengkap dengan logo **BMS** di Task Manager & `smartmontools` / `CrystalDiskInfo` / `HWiNFO64`).

#### 1. Install Langsung 1-Baris dari PowerShell Administrator (Bisa dari `C:\WINDOWS\system32`)

Mode otomatis penuh tanpa tanya (*unattended*):
```powershell
Invoke-WebRequest -Uri "https://github.com/Dot-Dodik/public-rilis/releases/download/auto-latest/cmkagent-windows-x86_64.exe" -OutFile "$env:TEMP\cmkagent-setup.exe"; & "$env:TEMP\cmkagent-setup.exe" install --auto-tools
```

Mode interaktif (dengan konfirmasi instalasi hardware tools):
```powershell
Invoke-WebRequest -Uri "https://github.com/Dot-Dodik/public-rilis/releases/download/auto-latest/cmkagent-windows-x86_64.exe" -OutFile "$env:TEMP\cmkagent-setup.exe"; & "$env:TEMP\cmkagent-setup.exe" install
```

#### 2. Install dari File `.exe` atau `.msi` yang Sudah Diunduh

Cukup **klik 2x** file `cmkagent-windows-x86_64.exe` atau `cmkagent-windows-x86_64.msi`, atau jalankan lewat PowerShell Administrator dari folder `Downloads`:
```powershell
cd "$env:USERPROFILE\Downloads"; .\cmkagent-windows-x86_64.exe install --auto-tools
```

#### 3. Install Menggunakan `install-windows.ps1` (Jika Clone Repo Lengkap)

Jalankan perintah ini **dari dalam folder root repo `cmkagent-rust`**:
```powershell
powershell -ExecutionPolicy Bypass -File .\installers\install-windows.ps1 -BinaryPath .\cmkagent-windows-x86_64.exe -AutoInstallTools
```

#### 4. Cek Status, Uji Output, & Update Manual (Windows PowerShell Admin)

Cek status Windows Service:
```powershell
Get-Service cmkagent
```

Uji payload output agen:
```powershell
& "C:\Program Files\cmkagent\cmkagent.exe" dump --config "C:\ProgramData\cmkagent\cmkagent.toml"
```

Paksa cek & terapkan update binary terbaru dari GitHub sekarang:
```powershell
& "C:\Program Files\cmkagent\cmkagent.exe" update-check --config "C:\ProgramData\cmkagent\cmkagent.toml"; Restart-Service cmkagent
```

---

### C. Native Enkripsi TLS Checkmk 2.5 (Windows 10/11 & Linux)
`cmkagent-rust` mendukung **Mutual TLS (mTLS) native Checkmk 2.5** (wire header `16` + TLS 1.3/1.2 `rustls` + kompresi `zlib`) dalam satu binary tanpa membutuhkan `cmk-agent-ctl` atau OpenSSL OS:

- **Auto-Import Host Lama:** Jika perangkat sebelumnya sudah pernah register TLS dengan agen Checkmk bawaan (`registered_connections.json`), `cmkagent install` otomatis menyalin & memakai sertifikat lamanya sehingga langsung hijau `OK` di Checkmk Server tanpa perlu *Remove TLS registration*.
- **Mode Default (`[tls] mode = "auto"`):**
  - Begitu host terdaftar/terimpor TLS, port `6556` **hanya** menerima handshake mTLS terenkripsi dari Server Checkmk (koneksi plaintext ditolak).
  - Untuk host baru yang belum terdaftar, bisa diaktifkan langsung saat install atau kapan saja lewat `cmkagent register`.
- **Auto-Renewal & Log Diagnostik (`tls.log`):** Worker latar belakang memperbarui sertifikat otomatis sebelum kedaluwarsa (`< 45 hari`) maupun untuk kompatibilitas Checkmk 3.0 (*Authority Key Identifier*). Seluruh aktivitas dan diagnostik handshake tercatat di `<data_dir>/tls.log` dan ditampilkan pada bagian `[Log TLS Terakhir]` saat menjalankan `cmkagent tls-status`.
- **Catatan Penting Port & Parameter Registrasi:**
  - Gunakan port **Agent Receiver** Checkmk (default `:8000`, misal `'192.168.55.112:8000'`), **bukan** port Web UI (`8081`/`80`). Jika port tidak ditulis, agen akan mencoba mendeteksinya secara otomatis.
  - `--hostname` adalah **nama host persis seperti yang didaftarkan di Checkmk** (mis. `'AGEN-TESTRUN'`), sedangkan `--user` adalah akun login/automation Checkmk (mis. `'cmkadmin'`).
  - Di PowerShell, bungkus nilai parameter dengan tanda kutip tunggal `'...'` dan **jangan** menyertakan tanda kurung sudut `<` atau `>`.

Daftar TLS setelah install / pada perangkat yang sudah terpasang:
```bash
# Linux
sudo cmkagent register --server '192.168.55.112:8000' --site 'cmk' --hostname 'NAMA-HOST-CHECKMK' --user 'cmkadmin' --password 'PASSWORD-CHECKMK' --trust-cert
sudo cmkagent tls-status
```
```powershell
# Windows (PowerShell Admin)
& "C:\Program Files\cmkagent\cmkagent.exe" register --server '192.168.55.112:8000' --site 'cmk' --hostname 'NAMA-HOST-CHECKMK' --user 'cmkadmin' --password 'PASSWORD-CHECKMK' --trust-cert
& "C:\Program Files\cmkagent\cmkagent.exe" tls-status
```

Install + register TLS sekaligus dalam 1 baris (Windows PowerShell Admin):
```powershell
Invoke-WebRequest -Uri "https://github.com/Dot-Dodik/public-rilis/releases/download/auto-latest/cmkagent-windows-x86_64.exe" -OutFile "$env:TEMP\cmkagent-setup.exe"; & "$env:TEMP\cmkagent-setup.exe" install --auto-tools --tls-server '192.168.55.112:8000' --tls-site 'cmk' --hostname 'NAMA-HOST-CHECKMK' --tls-user 'cmkadmin' --tls-password 'PASSWORD-CHECKMK' --trust-cert
```

Install + register TLS sekaligus dalam 1 baris (Linux):
```bash
curl -fsSL https://github.com/Dot-Dodik/public-rilis/releases/download/auto-latest/install-linux.sh | sudo bash -s -- --auto-tools --tls-server '192.168.55.112:8000' --tls-site 'cmk' --hostname 'NAMA-HOST-CHECKMK' --tls-user 'cmkadmin' --tls-password 'PASSWORD-CHECKMK' --trust-cert
```

---

### D. Subcommand CLI (`cmkagent` / `cmkagent.exe`)
- `cmkagent install [--auto-tools] [--tls-server ... --tls-site ... --tls-user ... --tls-password ...]` — Pasang/upgrade agen sebagai Background Service + opsional langsung registrasi mTLS.
- `cmkagent register --server <IP[:PORT]> --site <SITE> --user <USER> [--password <PASS> | --ott <TOKEN>] [--hostname <HOST>] [--trust-cert]` — Daftarkan enkripsi mTLS ke Checkmk 2.5 Agent Receiver.
- `cmkagent tls-status` — Tampilkan status koneksi mTLS, UUID, issuer CA, dan masa berlaku sertifikat.
- `cmkagent renew-cert [--force]` — Perbarui sertifikat mTLS secara manual ke server Checkmk.
- `cmkagent import-tls [PATH]` — Impor manual file `registered_connections.json` milik Checkmk agent bawaan.
- `cmkagent delete-tls [SITE|UUID|--all]` — Hapus registrasi TLS lokal.
- `cmkagent uninstall` — Hentikan dan hapus service `cmkagent`.
- `cmkagent run [--config PATH]` — Menjalankan agen di foreground (mode console/debug).
- `cmkagent service [--config PATH]` — Menjalankan agen sebagai daemon `systemd` atau Windows Service.
- `cmkagent dump [--config PATH]` — Mencetak seluruh payload (`<<<check_mk>>>` s/d `<<<local>>>`) ke stdout untuk pengujian.
- `cmkagent ping-once [--config PATH]` — Menjalankan 1 siklus ping 4x lalu mencetak status Checkmk.
- `cmkagent ping-log [--lines N]` — Menampilkan `N` baris terakhir dari log histori ping 30 hari (`ping_history.log`).
- `cmkagent memtest [--config PATH]` — Menjalankan uji verifikasi memori (`20%` Free RAM) untuk memperbarui `Health_RAM`.
- `cmkagent update-check [--config PATH]` — Mengecek dan menerapkan pembaruan binary terbaru langsung dari GitHub Release / mirror lokal secara manual.

---

## 5. Mekanisme Auto-Update Binary (Setiap 4 Jam, Tanpa Skrip `.sh`/`.ps1` Lokal)

Client Windows dan Linux **tidak lagi menggunakan skrip lokal `.sh`/`.ps1` eksternal**. Setiap penambahan fitur atau perbaikan dikompilasi menjadi binary baru (`cmkagent` / `cmkagent.exe`) dan otomatis diperbarui oleh agen dengan alur berikut:

1. **Kompilasi & Rilis Otomatis di GitHub Actions:** Setiap push ke branch `main` (atau tag `v*`) otomatis membuild binary Windows/Linux/Android, menghasilkan file checksum `SHA256SUMS.txt`, dan menerbitkannya ke GitHub Release (`auto-latest`).
2. **Pengecekan Berkala 4 Jam (`14400 detik`) + Random Jitter:** Worker `cmkagent-updater` di dalam binary mengecek `SHA256SUMS.txt` dari GitHub Releases (atau `mirror_base_url` server lokal cabang jika diatur di `cmkagent.toml`) setiap 4 jam dengan selisih acak (*jitter*) agar 3.000 perangkat tidak mengunduh secara bersamaan.
3. **Verifikasi Ketat SHA-256 & Magic Header:** Jika SHA-256 binary rilis berbeda dengan binary yang sedang berjalan, agen mengunduh file `.new`, memvalidasi header executable (`ELF` di Linux / `MZ` di Windows) serta kecocokan SHA-256 sebelum mengganti file aktif secara atomic.
4. **Auto-Restart Bersih:**
   - **Linux:** Memperbarui permission `+x`, menukar binary secara atomic, dan menjalankan `exec()` ulang secara bersih pada proses.
   - **Windows:** Memindahkan `cmkagent.exe` aktif ke `cmkagent.exe.old`, memasang `cmkagent.exe` baru, memicu restart service via Windows SCM (`net stop cmkagent & net start cmkagent`), lalu menghapus `.old` saat versi baru menyala.
   - Catatan aktivitas update dicatat di `<data_dir>/auto_update.log`.

---

## 6. Cara Build dari Source Code

### Build Binary Linux / Windows
```bash
cargo test --workspace
cargo build --release -p cmkagent-desktop
```

### Build Android APK (via Cargo NDK + Gradle)
```bash
cargo ndk -t arm64-v8a -t armeabi-v7a -t x86_64 -o android/app/src/main/jniLibs build --release -p cmkagent-android-jni
cd android && gradle assembleDebug
```
