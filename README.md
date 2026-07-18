# Nuclei Scan → Discord Webhook (GitHub Actions)

## Cara setup

1. Buat repo GitHub baru (atau pakai repo yang sudah ada), lalu masukkan folder `.github/workflows/nuclei-scan.yml` dan file `targets.txt` ke dalamnya.

2. Tambahkan secret di repo:
   - Buka **Settings > Secrets and variables > Actions > New repository secret**
   - Nama: `DISCORD_WEBHOOK`
   - Value: URL webhook Discord kamu (contoh: `https://discord.com/api/webhooks/xxxx/yyyy`)

3. Edit `targets.txt` dan isi dengan domain/URL milikmu sendiri atau target yang memang **berwenang/berizin untuk kamu scan** (contoh: aset milik sendiri atau program bug bounty yang sah).

4. Jalankan workflow:
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
