# Catatan: NFS Privilege Escalation & Permission Bits

## 1. NFS (Network File System)

Protokol yang memungkinkan sebuah mesin (client) me-mount direktori dari mesin lain (server) lewat jaringan, seolah-olah direktori itu ada di filesystem lokal. Server menentukan direktori mana yang di-*export* (dibagikan) dan opsi akses apa yang berlaku, biasanya dikonfigurasi di `/etc/exports`.

Contoh isi `/etc/exports`:
```
/var/www/html  192.168.1.0/24(rw,sync,no_root_squash)
```

Cara mengecek share yang di-export dari sisi client:
```
showmount -e <target-ip>
```

## 2. RCE (Remote Code Execution)

Kondisi di mana penyerang bisa mengeksekusi perintah/kode arbitrary di mesin target dari jarak jauh, biasanya lewat celah di aplikasi (web app vulnerable, service misconfigured, dll). RCE adalah *hasil* dari sebuah eksploitasi — bukan alat, melainkan kemampuan yang didapat. Contoh: upload `shell.php` ke web server vulnerable → begitu diakses lewat browser, perintah shell bisa dijalankan di server = RCE tercapai.

RCE sering jadi **Tahap 1 (initial access)** sebelum privilege escalation, karena RCE biasanya cuma memberi privilege selevel user yang menjalankan service tersebut (`www-data`, dll), bukan root.

## 3. "Mount" pada Teknik `no_root_squash` (bukan "shadowmount")

Istilah yang tepat untuk teknik ini adalah **NFS mount**, bukan "shadowmount" — mungkin salah dengar/salah istilah. Konsepnya: penyerang me-mount direktori yang di-export target ke mesin sendiri, memanfaatkan privilege root di mesin sendiri untuk membuat file dengan owner root di direktori tersebut.

```
mount -t nfs <target-ip>:/var/www/html /mnt/target-share
```

Setelah di-mount, semua operasi file (create, chown, chmod) di direktori itu dari sisi attacker akan terefleksi ke server NFS target — termasuk kemampuan membuat file ber-owner root, yang normalnya mustahil dilakukan attacker sebagai user biasa di sisi target.

## 4. `no_root_squash`

Opsi konfigurasi NFS yang menentukan bagaimana server memperlakukan UID root dari client.

- **Default (root squash aktif):** UID root dari client di-*squash* (diturunkan) jadi UID `nobody` saat mengakses share. Ini proteksi standar supaya root di client tidak otomatis jadi root di file server.
- **`no_root_squash`:** proteksi ini dimatikan. UID root dari client tetap dianggap root oleh server. Inilah yang membuat teknik privesc di atas bisa terjadi — attacker yang root di mesinnya sendiri bisa membuat file ber-owner root di server target lewat mount.

## 5. Permission Bits — Selain 3 Digit Biasa

Permission Linux standar (`rwx` untuk owner/group/other) itu 3 digit (misal `755`). Tapi ada **digit ke-4 (special permission bit)** di depannya yang sering dipakai di privesc:

| Digit | Nama | Simbol di `ls -l` | Fungsi |
|---|---|---|---|
| `4` | **SUID** (Set User ID) | `s` di posisi owner-execute | Binary jalan dengan privilege **owner file**, bukan privilege user yang mengeksekusi. Contoh: `4755` → `-rwsr-xr-x` |
| `2` | **SGID** (Set Group ID) | `s` di posisi group-execute | Untuk file: jalan dengan privilege **group** file. Untuk direktori: file baru yang dibuat di dalamnya otomatis mewarisi group direktori tersebut |
| `1` | **Sticky Bit** | `t` di posisi other-execute | Biasa dipakai di direktori shared (seperti `/tmp`) — user hanya bisa hapus/rename file miliknya sendiri, walau direktori writable oleh semua orang |

**Cara membaca contoh:**
- `4755` → SUID + rwxr-xr-x → tampil sebagai `-rwsr-xr-x`
- `2755` → SGID + rwxr-xr-x → tampil sebagai `-rwxr-sr-x`
- `1755` → Sticky + rwxr-xr-x → tampil sebagai `-rwxr-xr-t`
- `6755` → SUID + SGID sekaligus → `-rwsr-sr-x`

**Catatan penting (dari case kamu kemarin):**
- Lupa menulis digit ke-4 (misal cuma `755` bukan `4755`) = special bit tidak ke-set sama sekali, walau permission dasar terlihat "benar"
- SUID **hanya efektif untuk compiled binary (ELF)**, kernel modern mengabaikan SUID pada script berbasis shebang (Python, Bash, PHP, dll) sebagai proteksi keamanan
- Owner file harus benar (`root`) — SUID tanpa owner root tidak ada gunanya untuk privesc ke root
- Verifikasi selalu dengan `ls -l` — cari huruf `s` (bukan `x`) di posisi yang sesuai
