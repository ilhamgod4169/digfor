# [B] Panduan CTF: Digital Forensics

Kategori ini fokus pada analisis barang bukti digital seperti file gambar, rekaman jaringan (pcap), dan file log.

---

## 📁 1. Forensik File & Sembunyian (Artifacts)
Mencari data yang disembunyikan di dalam file lain.

| MASALAH | PERINTAH | PENJELASAN |
| :--- | :--- | :--- |
| **Cek Metadata** | `exiftool <NAMA_FILE>` | Melihat informasi pembuat file, GPS, dan tanggal asli. |
| **Ekstrak File** | `binwalk -e <NAMA_FILE>` | Mengekstrak file yang disembunyikan di dalam file lain (Steganografi). |
| **Cari String** | `strings <NAMA_FILE> | tail` | Melihat teks di akhir file (sering ada flag di sana). |

---

## 🌐 2. Forensik Jaringan (Network)
Menganalisis file hasil rekaman traffic jaringan (biasanya berformat `.pcap`).

| MASALAH | PERINTAH | PENJELASAN |
| :--- | :--- | :--- |
| **Dengar Jaringan** | `tcpdump -r <FILE_PCAP> -A` | Melihat konten paket data dalam bentuk teks (ASCII). |
| **Extract Flag** | `strings <FILE_PCAP> | grep -i "lks{"` | Cara cepat mencari flag di ribuan paket data. |
| **Visual Analysis** | `wireshark <FILE_PCAP> &` | Membuka Wireshark untuk analisis mendalam (Follow TCP Stream). |

---

## 📜 3. Forensik Log (Offline)
Membaca file log server untuk mencari aktivitas penyerang.

```bash
# Mencari IP penyerang di log akses web
cat access.log | awk '{print $1}' | sort | uniq -c

# Mencari kata kunci "flag" di file log yang sangat besar
grep -rni "flag" /path/ke/folder/log/
```

---

## 🧠 4. Memory Forensics (Volatility)
Jika Anda diberikan file "Memory Dump" (umumnya akhiran `.raw` atau `.mem`).
```bash
# Mengetahui profil OS dari memory dump tersebut
volatility -f <FILE_MEMORY> imageinfo

# Melihat proses yang sedang berjalan saat komputer mati
volatility -f <FILE_MEMORY> --profile=<PROFIL> pslist
```
