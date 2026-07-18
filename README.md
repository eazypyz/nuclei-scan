# Nuclei Scan → Discord Webhook (GitHub Actions)

## Optimasi kecepatan (penting untuk ribuan subdomain)

Workflow sekarang dipecah jadi 3 job supaya tidak kena limit **6 jam max per job** di GitHub Actions:

1. **`prepare`** — ambil domain dari asset-dashboard → **probing host hidup pakai `httpx`** (host yang mati/tidak respons langsung dibuang, biasanya motong 30-70% waktu) → split jadi beberapa chunk (`CHUNKS: 6` di bagian `env`, bisa dinaikkan sampai ~20)
2. **`scan`** — tiap chunk di-scan **paralel** di runner terpisah (matrix strategy), dengan tuning:
   - `-c 50` → concurrency (jumlah template/host diproses bersamaan)
   - `-rl 150` → rate limit request/detik
   - `-timeout 5 -retries 1` → jangan tunggu lama untuk host yang lambat/mati
   - `-severity low,medium,high,critical` → sesuaikan/kurangi kalau mau lebih cepat lagi (misal cuma `high,critical`)
   - Template nuclei di-cache supaya tidak download ulang tiap run
3. **`notify`** — gabung semua hasil chunk jadi satu file `.txt`, lalu kirim ke Discord

**Cara atur jumlah paralel:**
- Ubah `CHUNKS: 6` di bagian `env:` sesuai kebutuhan
- Ubah juga daftar `matrix.index: [0, 1, 2, 3, 4, 5]` di job `scan` — jumlah angkanya **harus sama atau lebih banyak** dari `CHUNKS` (index yang tidak dipakai otomatis di-skip)
- Contoh: untuk 10 chunk paralel, set `CHUNKS: 10` dan `index: [0,1,2,3,4,5,6,7,8,9]`

**Tips tambahan kalau masih lambat:**
- Kurangi severity yang di-scan (misal hanya `high,critical`) untuk mengurangi jumlah template yang dijalankan
- Gunakan `-tags` untuk fokus ke kategori tertentu (misal `-tags cve,exposure,misconfig`)
- Pertimbangkan **self-hosted runner** (VPS sendiri) kalau butuh scan sangat besar tanpa batas 6 jam sama sekali
- Jadwalkan scan bertahap (misal per 500 domain per hari) daripada semua sekaligus tiap hari

## Sumber target domain

Workflow ini **otomatis mengambil daftar subdomain** dari repo:
`https://github.com/eazypyz/asset-dashboard/tree/main/data/history`

Format file JSON di folder itu (contoh `target.json`, `subdomain.json`) berupa array of object seperti:
```json
[
  {
    "type": "new_subdomain",
    "host": "account-api.bandainamcoid.com",
    "timestamp": "2026-07-17T19:39:01.075Z"
  }
]
```

Cara kerja workflow (job `prepare`):
1. Repo `asset-dashboard` di-checkout (sparse checkout, hanya folder `data/history`)
2. Semua file `*.json` di folder itu dibaca
3. Field `.host` dari tiap object diambil dengan `jq -r '.[]?.host'`
4. Semua host digabung, duplikat dihapus, lalu di-probe dengan `httpx` untuk cek mana yang hidup
5. Host yang hidup dipecah jadi beberapa chunk, discan paralel di job `scan`, hasilnya digabung & dikirim ke Discord di job `notify`

Kalau ternyata ada file JSON lain di folder tersebut dengan struktur berbeda (bukan array of object dengan key `host`), kasih tahu saya formatnya biar filter `jq`-nya disesuaikan.

Kalau repo `asset-dashboard` itu **private**, workflow butuh token tambahan (Personal Access Token / PAT) dengan akses read ke repo tersebut, disimpan sebagai secret (misal `DASHBOARD_REPO_TOKEN`), lalu ditambahkan di step checkout kedua dengan `token: ${{ secrets.DASHBOARD_REPO_TOKEN }}`. Kabari saya kalau repo-nya private supaya saya update workflow-nya.

## Cara setup

1. Buat repo GitHub baru (atau pakai repo yang sudah ada) untuk workflow ini, lalu masukkan folder `.github/workflows/nuclei-scan.yml` ke dalamnya. (`targets.txt` tidak wajib lagi karena akan digenerate otomatis, tapi tetap disediakan sebagai fallback/contoh format.)

2. Tambahkan secret di repo:
   - Buka **Settings > Secrets and variables > Actions > New repository secret**
   - Nama: `DISCORD_WEBHOOK`
   - Value: URL webhook Discord kamu (contoh: `https://discord.com/api/webhooks/xxxx/yyyy`)

3. Jalankan workflow:
   - Manual: buka tab **Actions** di repo → pilih workflow **Nuclei Scan to Discord** → **Run workflow**
   - Bisa isi field `target` kalau mau scan 1 target spesifik tanpa pakai `targets.txt`
   - Otomatis: workflow juga akan jalan tiap hari jam 02:00 UTC sesuai jadwal `cron` (bisa diubah/dihapus di file yml)

5. Hasil scan akan:
   - Disimpan sebagai **artifact** `nuclei-result.txt` (bisa didownload dari halaman run di tab Actions)
   - Dikirim otomatis ke **Discord webhook** sebagai file `.txt` beserta ringkasan jumlah baris hasil

## Catatan penting

- **Gunakan hanya untuk aset/domain yang kamu miliki atau punya izin resmi untuk melakukan security testing.** Scanning tanpa izin terhadap sistem milik orang lain melanggar hukum di banyak negara (termasuk UU ITE di Indonesia).
- File `nuclei-result.txt` akan berisi "Tidak ada temuan (no findings)." jika scan tidak menemukan kerentanan, supaya proses upload/kirim tetap berjalan.
- Kamu bisa menambahkan filter severity, misalnya:
  ```
  nuclei -l targets.txt -severity critical,high -o results/nuclei-result.txt -silent
  ```
- Jika ingin membatasi ukuran file yang dikirim ke Discord (limit 8MB untuk webhook biasa), bisa tambahkan `head -c 7000000` sebelum upload jika hasil scan sangat besar.
