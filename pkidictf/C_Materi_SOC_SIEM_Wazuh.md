# [C] Panduan CTF: SOC SIEM (Wazuh)

Kategori ini fokus pada penggunaan Wazuh sebagai alat pemantau (Monitoring), deteksi ancaman (Detection), dan pencarian flag di tumpukan log server.

---

## 🎨 1. Panduan Visual: Cara Pertama Kali Pakai Wazuh
Setelah instalasi selesai, ikuti urutan klik ini agar tidak bingung:

### A. Login & Akses
1. Buka Browser di Kali Linux, akses: `https://<IP_SERVER_WAZUH>`.
2. Masukkan Username: `admin` & Password yang Anda catat.

### B. Menghubungkan Agent (Klik Menu Berikut)
1. Klik **Menu (Garis Tiga Kiri Atas)** -> **Agents** -> **Deploy new agent**.
2. Masukkan IP Server Wazuh Anda di kolom yang tersedia.
3. Copy script yang muncul (satu baris panjang).
4. Jalankan script tersebut di terminal komputer target (Client).
5. Klik **Agents** kembali, pastikan statusnya sudah **Active** (warna hijau).

### C. Mencari Flag (Discovery)
1. Klik **Menu** -> **Security Events** -> **Events**.
2. Di bar pencarian atas, ketik: `lks{` lalu tekan Enter.
3. Semua log yang mengandung flag akan muncul di daftar bawah.

---

## 🏗️ 2. Struktur & Instalasi Komponen (Lengkap)

Wazuh terdiri dari 3 komponen utama yang harus terpasang agar SIEM berjalan:
- **Wazuh Indexer:** Penyimpan data log (Gudang data yang bisa dicari/search).
- **Wazuh Server:** Analis ancaman (Otak yang memicu alert).
- **Wazuh Dashboard:** Antarmuka Web (Visualisasi grafik & tabel).

### Langkah Instalasi (All-in-One):
```bash
# 1. Update sistem Kali/Ubuntu
sudo apt update && sudo apt install curl -y

# 2. Jalankan script instalasi lengkap
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```
> [!IMPORTANT]
> Simpan user **admin** dan **password** yang muncul di akhir instalasi. Jika terlewat, cek file `wazuh-install.sh.log`.

---

## 🔍 2. Jurus Mencari Flag (Flag Hunting)
Inilah bagian terpenting dalam CTF Blue Team. Jangan hanya pakai satu cara!

### A. Teknik Grep "Harta Karun" (Log Mentah)
Seringkali flag tidak muncul di Dashboard karena dianggap "log biasa". Paksa cari di file mentah Manager:
```bash
# Mencari string flag di semua log (aktifkan logall di ossec.conf dulu)
sudo grep -ri "lks{" /var/ossec/logs/archives/archives.json
```

### B. Teknik FIM (File Integrity Monitoring)
Jika soal menyebutkan penyerang menaruh file rahasia:
1. Buka Dashboard -> **Integrity Monitoring**.
2. Cari event **"Added"** atau **"Modified"**.
3. Lihat kolom **"File"**. Jika ada file mencurigakan di `/tmp/` atau `/var/www/html/`, baca isinya:
   `cat /path/ke/file/mencurigakan`

### C. Teknik Command Monitoring (Lacak Pengetikan)
Jika penyerang mengetik flag di terminal client, Wazuh bisa mencatatnya:
1. Dashboard -> **Discover**.
2. Masukkan filter: `data.command: "lks{"` atau `data.args: "lks{"`.
3. Ini akan menampilkan perintah apa yang dijalankan yang mengandung teks flag.

### D. Teknik SQL & Web Analysis
Jika flag ada di database atau web request:
- Filter Dashboard: `data.url: "lks{"` atau `data.payload: "lks{"`.
- Cek Rule ID **31103** (SQL Injection) atau **31106** (Path Traversal), biasanya flag ada di dekat serangan tersebut.

---

## 🏆 4. Workflow Menang: Cara Pakai Wazuh Saat Lomba SOC SIEM
Jangan panik saat serangan masuk. Ikuti urutan kerja ini:

### Langkah 1: Persiapan (Menit 0-5)
- Cek semua agent: `Wazuh -> Agents`. Pastikan statusnya **Active**.
- Buka tab **Wazuh -> Security Events -> Events**. Refresh setiap 10 detik.

### Langkah 2: Deteksi Serangan (Mencari Penyerang)
- Lihat grafik **"Top Alert Frameworks"**.
- Jika ada lonjakan warna merah, klik pada IP tersebut (`data.srcip`).
- **Tujuan:** Mengetahui siapa penyerangnya dan apa yang mereka incar (Web, SSH, atau File).

### Langkah 3: Investigasi (Mencari Flag)
- Setelah tahu IP-nya, filter log hanya dari IP tersebut.
- Masukkan kata kunci pencarian: `lks{` atau `flag`.
- Cek tab **JSON** pada log tersebut. Flag sering diselipkan di dalam pesan error atau URL.

### Langkah 4: Dokumentasi (Evidence Capture)
- Salin **Rule ID**, **Timestamp**, dan **Full Log**.
- Screenshot Dashboard yang menampilkan aktivitas serangan tersebut sebagai bukti ke Juri.

---

## 🛡️ 5. Tindakan Respon Cepat (SOC Action)
 Apa yang dilakukan setelah menemukan bukti?

| MASALAH | PERINTAH | PENJELASAN |
| :--- | :--- | :--- |
| **Cek Status Server** | `sudo systemctl status wazuh-manager` | Memastikan server SIEM tidak mati diserang. |
| **Blokir IP** | `sudo /var/ossec/active-response/bin/firewall-drop.sh add - <IP_PENYERANG>` | Memutus akses penyerang secara instan. |
| **Matikan Proses** | `sudo kill -9 <PID_JAHAT>` | Membunuh program malware atau backdoor yang jalan. |

---

## 💡 Tips Menang CTF Blue Team:
1. **Archives is King:** Selalu aktifkan `<logall>yes</logall>` di `/var/ossec/etc/ossec.conf` agar semua flag (sekecil apapun) terekam.
2. **Timestamp:** Perhatikan waktu di log. Flag biasanya muncul sesaat setelah ada aktivitas mencurigakan (merah/level tinggi).
3. **Filter Sederhana:** Jangan buat query rumit. Cukup cari `"lks{"` di semua kolom.
