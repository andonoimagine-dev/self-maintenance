# Catatan Perubahan / Upgrade

Log ringkas tiap perubahan signifikan ke sistem TPM (HSB/CND/dashboard) di repo ini. Entri terbaru di paling atas. Setiap entri diawali tag `[nama-file.html]` — **cek tag ini dulu** sebelum baca, karena beberapa file di repo ini punya app/spreadsheet/password terpisah dan tidak selalu saling berkaitan (lihat tabel referensi di bawah).

## Referensi file HTML di repo ini

| File | Aplikasi | Backend/password |
|---|---|---|
| `hsb-management-system.html` | TPM HSB (ampere impeller, sparepart, limbah, steelshot, trolley, preventive) | GAS sendiri, pwd `hsb2026` |
| `cnd-management-system.html` | TPM CND — paralel struktur dengan HSB tapi app/spreadsheet terpisah | GAS sendiri, pwd `cnd2026` |
| `hsb-test-koneksi.html` | Diagnostic tool 5-step (koneksi→auth→init→read→write) untuk backend gaya HSB/CND | — (isi URL/pwd manual saat pakai) |
| `1.html` | "Self MTC Dashboard" (Chart.js) | Sama dengan `2.html` |
| `2.html` | "Form Temuan Self-Maintenance" | Sama dengan `1.html` |
| `index.html`, `dashboard_selfmtc.html` | Redirect stub ke portal eksternal, bukan app aktif | — |

Jangan asumsikan perbaikan di satu file berlaku juga ke file lain — masing-masing independen kecuali disebutkan eksplisit.

---

## [3.html] 2026-09-24 — Jenis "Shotblast Ulang", dan perbaikan cacat rumus "Belum"

**Latar:** portal TPM HSB perusahaan (`castingace/homepage.php?module=TPMHSB`)
memakai TIGA jenis di shootblast ulang, aplikasi ini hanya dua. Akibatnya 21
baris dari portal tidak punya tempat sama sekali di sini. Jenis ketiganya
**"Shotblast Ulang"** — barang yang harus diproses kembali karena kurang
bersih, padahal sudah lewat jalur shootblast.

**CACAT LATEN yang ketemu saat menambahkannya, dan diperbaiki lebih dulu.**
`sbuStockAt()` menghitung stok "Belum" begini:

```js
(x.j === 'Masuk' ? x.v : -x.v)
```

Cabang `else` **MENGURANGI apa pun yang bukan "Masuk"**. Aman selama hanya ada
dua nilai — tapi begitu nilai ketiga masuk, ia ikut dikurangkan diam-diam dan
angka "Belum" jadi salah **tanpa satu pun gejala**. Sekarang tiap nilai disebut
tegas, dan yang tidak dikenal tidak dihitung.

**Keputusan user:** "Shotblast Ulang" **MENAMBAH** antrean — barang yang harus
diproses kembali tetap menunggu di area HSB, sama seperti yang baru masuk.
Rumusnya jadi:

```
Belum = (Masuk + Shotblast Ulang) − Selesai
```

Keterangan rumus di halaman ikut menyebutkan bahwa batang harian Masuk dan
Selesai tetap per-jenis, jadi "Belum" tidak selalu sama dengan selisih keduanya
— kalau tidak disebut, angkanya terbaca seperti salah hitung.

**Tujuh tempat diubah:** rumus stok, pilihan di form entri, pilihan di form
edit, label tabel riwayat, konfirmasi hapus, toast, dan keterangan rumus.

**Data dari portal ikut disalin masuk** lewat project `auto-task` (repo
terpisah, privat) — satu arah portal → aplikasi, tidak ada yang ditulis ke
portal. Yang masuk ke spreadsheet ini:

| tabel | baris | hasil |
|---|---|---|
| sparepart | 32 | 147 → 179 |
| ampere | 1.109 | 3.207 → 4.316 |
| ikomi | 9 | 96 → 105 |
| steelshot | 60 | 156 → 216 |
| shootblast_ulang | 68 | 70 → 138 |
| preventive | 28 | 464 → 492 |

Semuanya diverifikasi dengan membaca ulang dan menghitung sesudah mengirim,
bukan dengan percaya jawaban Apps Script — `saveX` MENAMBAH baris, dan sheet
`preventive` pernah membengkak jadi 31.296 baris justru karena kiriman yang
diulang (lihat entri 2026-08-06 di bawah).

**Yang SENGAJA tidak disalin:**
- `limbah` — portal mencatat tiap trolley satu baris, aplikasi satu baris per
  shift. Beda tingkat rincian; keputusan user: ikuti aplikasi.
- 67 baris `steelshot` yang angkanya berselisih dengan portal untuk hari dan
  shift yang sama — keputusan user: aplikasi yang benar.
- 284 item `preventive` bernilai `pending` — "belum diperiksa" bukan hasil
  pemeriksaan, dan aplikasi ini memang tidak mengenal nilai itu.

**File yang diubah:** `3.html`. **Tidak menyentuh** `4.html`, `1.html`,
`2.html` — backend/spreadsheet-nya terpisah.

---

## [hsb-management-system.html] 2026-08-06 — Fix bug retry-sync duplikat data

**Masalah:** User melaporkan HSB Management System tidak bisa sync. Investigasi menemukan dua hal:

1. **Bug kode**: `pushToSheets()` / `flushQueue()` di `hsb-management-system.html` akan mengulang kirim data yang timeout di sisi browser, walau request itu mungkin sudah sukses ditulis ke Apps Script di server — menyebabkan baris yang sama tertulis berkali-kali tiap siklus auto-sync tanpa batas.
2. **Dampak nyata**: sheet `preventive` di spreadsheet "HSB Monitor - Casting 2" sudah membengkak ke **31.296 baris** (seharusnya ~432), sebagian tanggal terduplikasi 100-200x lipat. Payload `getAll` jadi 5,4 MB / ~9,3 detik fetch — mepet ke batas timeout 15 detik, jadi lingkaran setan (makin lambat → makin sering timeout → makin banyak duplikat).

**Perbaikan:**
- `hsb-management-system.html`:
  - Timeout fetch `api()` dinaikkan 15s → 30s.
  - Tiap write (`pushToSheets`) dan retry (`queueSync`) sekarang dikasih `reqId` unik untuk pelacakan.
  - `flushQueue()` dibatasi maksimal **8x percobaan** (`MAX_QUEUE_TRIES`) per item — bukan retry tanpa henti. Setelah itu, item di-drop dan user diberi toast peringatan, bukan diam-diam menduplikasi data selamanya.
  - Di-merge ke `main`, live di GitHub Pages.
- Dibuat `dedupe-preventive.gs.txt` — skrip Apps Script standalone (dry-run + apply) untuk membersihkan sheet `preventive` di spreadsheet HSB. Dijalankan manual oleh user di editor Apps Script (bukan Claude — kebijakan: penghapusan data permanen harus dieksekusi user sendiri).
  - Percobaan pertama (`dedupePreventiveApply`, hapus baris satu-per-satu via `deleteRow`) kena timeout 6 menit Apps Script, baru selesai ~1.477/30.864 baris.
  - Ditambahkan `dedupePreventiveApplyFast` (bulk rewrite unique rows + satu `deleteRows()`) — selesai ~20 detik. User jalankan, sheet `preventive` bersih jadi 432 baris.

**Hasil:** Payload `getAll` turun 5,4 MB → 666 KB. Sheet `preventive`: 31.296 → 432 baris.

**File yang diubah:** `hsb-management-system.html`, `dedupe-preventive.gs.txt` (baru). `hsb-test-koneksi.html` dipakai untuk diagnosa awal, tidak diubah. **Tidak menyentuh** `cnd-management-system.html`, `1.html`, `2.html` — backend/spreadsheet-nya terpisah, bug ini spesifik ke HSB.
