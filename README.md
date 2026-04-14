# digfor1. INFRASTRUKTUR & IDENTIFIKASI
Langkah pertama wajib sebelum menyentuh file.
file [nama_file]: Identifikasi tipe file (Disk, RAM, atau data biner).
mmls [file.img]: Cek tabel partisi. Catat angka START sector.
fsstat -o [offset] [file.img]: Cek tipe filesystem (FAT, NTFS, EXT4) dan metadata.
2. EKSPLORASI FILE (Sleuthkit)
Gunakan jika file sistem masih terbaca.
fls -o [offset] -r [file.img]: List file secara rekursif.
fls -o [offset] -r -d [file.img]: Cari file yang *DIPENGAL (Deleted).
icat -o [offset] [file.img] [inode] > hasil.ext: Ekstrak file berdasarkan nomor Inode.
3. MOUNTING (Akses Seperti Folder Biasa)
Paling enak buat nyari file di banyak folder atau kalau mau pakai perintah git.
mkdir mnt_point: Buat folder tujuan.
sudo mount -o loop,offset=$((START_SECTOR*512)) file.img mnt_point: Pasang disk ke folder.
sudo umount mnt_point: Lepas setelah selesai.
4. RECOVERY & CARVING (Jika Sistem File Rusak)
Gunakan jika fls gagal atau file dihapus permanen.
foremost -i [file.img] -o output: Cari file berdasarkan header (otomatis).
binwalk -e [file.img]: Cari dan ekstrak data tersembunyi (sering buat stego/firmware).
extundelete --restore-all [file.img] --offset [byte_offset]: Khusus partisi Linux (EXT3/4).
5. STRING SEARCH (Jurus "Grepe-Grepe")
Kadang flag ada di teks polos atau komentar.
strings -t d -o [offset] [file.img] | grep -i "picoCTF{": Cari flag dan posisinya.
grep -a -o 'picoCTF{[^}]*}' [file.img]: Cari pola flag di dalam file biner.
exiftool [file.jpg/png]: Cek metadata gambar (koordinat GPS, author, pesan di komentar).
6. MEMORY FORENSICS (RAM Dump)
Gunakan jika file berupa .raw, .mem, atau .ad1.
vol3 -f [file] windows.info: Cek info OS (Volatility 3).
vol3 -f [file] windows.pslist: Lihat daftar proses yang berjalan.
vol3 -f [file] windows.filescan | grep -i "flag": Cari file di memori.
vol3 -f [file] windows.dumpfiles --virtaddr [address]: Ekstrak file dari RAM.
7. PASSWORD & STEGO (Bumbu Tambahan)
Jika ketemu file terkunci atau gambar mencurigakan:
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt file.zip: Crack password ZIP.
steghide extract -sf image.jpg: Ekstrak pesan tersembunyi di gambar (biasanya perlu pass).
stegsolve: (GUI) Ganti-ganti filter warna gambar buat nyari teks tersembunyi.
8. TRIK KHUSUS GIT FORENSIC
Jika ketemu folder .git (atau hasil recovery folder .git):
git log --all --full-history: Lihat semua sejarah commit (termasuk yang dihapus).
git reflog: Lihat semua pergerakan HEAD (sangat sakti buat nemu commit "hilang").
git show [hash_commit]: Lihat isi perubahan di commit tertentu.
git diff [commit_A] [commit_B]: Bandingkan perbedaan dua commit.
