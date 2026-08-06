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
