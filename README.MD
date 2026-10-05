# Kalingga

---

## Fitur Utama

- **Ekstraksi ID SLS & Wilayah**: Membaca kode SLS 16 digit, nama SLS, desa, kecamatan, dan kabupaten via AI OCR lokal serta barcode QR-LOC.
- **Batch Renaming**: Mengganti nama berkas secara massal sesuai format standar (`{kode_sls}_WSS.jpg`) dengan fitur pemulihan (*Undo*).
- **Auto-Georeferensi (.jgw / .pgw)**: Mencocokkan batas merah peta hasil scan dengan layer referensi `Final_SLS.geojson` untuk menghasilkan World File secara otomatis.
- **GIS Overlay Studio**: Pratinjau tumpangsusun citra peta dengan citra satelit dan poligon GeoJSON, dilengkapi tombol koreksi manual, koreksi otomatis, dan shortcut buka langsung ke QGIS.
- **Upload ke Folder Satker**: Pengunggahan berkas peta dan world file langsung ke cloud storage Satker tujuan dengan proteksi presisi data.
- **100% Offline & Mandiri**: Komputasi OCR dan georeferensi berjalan lokal di perangkat tanpa memerlukan koneksi internet.

---

## Alur Kerja

### 1. Mode Otomatis (All-in-One Pipeline)
Mode ini digunakan untuk memproses banyak berkas sekaligus:
1. Pilih folder berkas peta scan dan muat berkas `Final_SLS.geojson`.
2. Klik **Mulai Semua Proses**. Aplikasi akan menjalankan alur:
   $$\text{Cek ID SLS (OCR)} \longrightarrow \text{Ganti Nama Berkas} \longrightarrow \text{Georeferensi (.jgw)} \longrightarrow \text{Unggah Drive}$$
3. **Proteksi Unggah Drive**: Hanya berkas yang berstatus **Presisi (Pas)** yang akan diunggah ke folder Satker. Berkas yang masih berstatus **Perlu Cek** atau memiliki peringatan otomatis ditahan demi menjaga kualitas data.
4. Laporan ringkasan akhir menampilkan daftar berkas presisi, berkas yang perlu dicek, dan tombol untuk koreksi otomatis massal atau inspeksi visual.

### 2. Mode Manual
Setiap fitur dapat dijalankan secara terpisah dari menu utama:
- **1. Cek ID SLS**: Menjalankan ekstraksi teks dan barcode tanpa mengubah nama berkas.
- **2. Ganti Nama**: Mengganti nama berkas yang telah tervalidasi.
- **3. Georeferensi**: Membuat berkas `.jgw` untuk berkas yang sudah memiliki ID SLS dan data GeoJSON.
- **4. Unggah Drive**: Membuka modal pengunggahan ke folder Satker (dengan proteksi presisi aktif secara bawaan).

---

## Status Akurasi Georeferensi

| Status | Indikator | Kriteria | Tindakan |
| :--- | :---: | :--- | :--- |
| **Presisi (Pas)** | Hijau | Skor $\ge 80\%$, koordinat menempel pas pada batas GeoJSON | Siap dibuka di QGIS dan otomatis diizinkan untuk diunggah ke Drive Satker. |
| **Perlu Cek** | Oranye | Skor $50-79\%$ atau menggunakan aproksimasi QR-LOC | Ditahan dari unggahan Drive. Buka Studio Spasial atau jalankan koreksi otomatis. |
| **Melenceng / Gagal** | Merah | Skor $< 50\%$ atau ID SLS tidak ditemukan di GeoJSON | Periksa kembali berkas GeoJSON atau sesuaikan posisi di Studio Spasial. |

---

## Cara Kerja Algoritma (Tahap 1 - Tahap 4)

Secara garis besar, Kalingga memproses peta melalui 4 tahapan berurutan:
1. **Tahap 1 (Cek ID SLS):** Normalisasi EXIF fisik citra, pemindaian barcode QR-LOC (centroid GPS), dan ekstraksi 16 digit ID SLS via deep learning OCR (DBNet + SVTR).
2. **Tahap 2 (Ganti Nama Berkas):** Standardisasi nama berkas massal ke pola `{kode_sls}_WSS.jpg` dengan jurnal riwayat *safe rollback (undo)*.
3. **Tahap 3 (Auto-Georeferensi .jgw):** Kunci orientasi fisik 0°, segmentasi batas merah dual-space (HSV + LAB), kalkulasi skala cetak (A4/A3) rasio isotropik 1:1, dan pencocokan kontur Chamfer Distance Transform ICP dengan konsensus tangguh 75th percentile.
4. **Tahap 4 (Unggah Folder Satker):** Pengunggahan terproteksi ke cloud storage Satker tujuan via WebDAV HTTPS, di mana hanya berkas berstatus **Presisi (Pas)** yang diizinkan untuk dikirim.

> Penjelasan teknis mendalam mengenai formula matematis, normalisasi EXIF, dan metode optimasi tiap tahapan dapat dibaca di berkas **[ALGORITHM.md](ALGORITHM.md)**.

---

## Lisensi
Hak Cipta © 2026 putuwahyu29. Dikembangkan untuk unit kerja Badan Pusat Statistik (BPS).
