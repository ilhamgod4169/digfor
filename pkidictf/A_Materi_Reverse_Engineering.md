# [A] Panduan CTF: Reverse Engineering (RE/Pwn)

Kategori ini fokus pada membedah file biner (ELF untuk Linux atau EXE untuk Windows) untuk memahami alur program dan mencari Flag.

---

## 🛠️ 1. Tahap Statis (Lihat Saja, Jangan Jalankan)
Gunakan tool ini di terminal Kali Linux untuk melihat isi file tanpa mengeksekusinya.

| MASALAH | PERINTAH | PENJELASAN |
| :--- | :--- | :--- |
| **Cek Info Biner** | `file <NAMA_FILE>` | Mengetahui apakah file ini 64-bit, 32-bit, atau script. |
| **Cek Proteksi** | `checksec --file=<NAMA_FILE>` | Melihat apakah ada proteksi seperti NX (No-Execute) atau ASLR. |
| **Cari Flag** | `strings <NAMA_FILE> | grep -i "lks{"` | Mengekstrak teks yang bisa dibaca. Seringkali flag ada di sini. |

---

## 🏃 2. Tahap Dinamis (Jalankan & Pantau)
Gunakan tool ini untuk memantau perilaku program saat sedang berjalan di sistem.

| MASALAH | PERINTAH | PENJELASAN |
| :--- | :--- | :--- |
| **Trace System Call** | `strace ./<NAMA_FILE>` | Melihat file apa yang dibuka dan koneksi apa yang dibuat program. |
| **Trace Library** | `ltrace ./<NAMA_FILE>` | Melihat fungsi pemrograman (seperti `strcmp` atau `printf`) yang dipanggil. |
| **Mode Debug** | `gdb -q ./<NAMA_FILE>` | Memasuki mode debug untuk melihat isi memori dan register. |

---

## 🏗️ 3. Tahap Analisis Berat (Ghidra)
Jika file sangat kompleks, gunakan tool visual:
1. Ketik `ghidra` di terminal Kali Linux.
2. New Project -> Import File.
3. Jalankan **Auto-Analyze**.
4. Lihat jendela **Decompiler** (Kode sebelah kanan) untuk membaca alur program dalam bahasa C.

---
> [!TIP]
> **Tujuan Akhir:** Temukan fungsi di mana input Anda dibandingkan dengan Flag yang benar.
