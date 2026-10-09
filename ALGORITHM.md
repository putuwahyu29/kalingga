# Alur Kerja & Spesifikasi Algoritma Kalingga (Tahap 1 - Tahap 4)

Dokumen ini memuat dokumentasi teknis mendalam mengenai arsitektur pemrosesan, formulasi matematis, dan tahapan algoritma Kalingga mulai dari pembacaan citra, penamaan berkas, pembuatan koordinat spasial (*georeferencing*), hingga pengunggahan terproteksi ke cloud storage Satker.

---

## 1. Diagram Alur Keseluruhan (Tahap 1 s.d. Tahap 4)

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│ TAHAP 1: EKSTRAKSI & CEK ID SLS (AI OCR & BARCODE)                              │
│ • Normalisasi Fisik EXIF Tag 274 (Reset = 1, Paritas GDAL/QGIS)                 │
│ • Pemindaian Barcode QR-LOC (Centroid GPS @lat,lon via ZXing-C++)               │
│ • Deep Learning OCR: DBNet (Deteksi Teks) + SVTR (Pengenalan Karakter)          │
│ • Heuristik Typo Scanner & Validasi 16 Digit SLS Resmi BPS                      │
└───────────────────────────────────────┬─────────────────────────────────────────┘
                                        │ [ID SLS 16 Digit Valid]
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ TAHAP 2: GANTI NAMA BERKAS (BATCH RENAME & SAFE ROLLBACK)                       │
│ • Standardisasi Template Nama: {kode_sls}_WSS.jpg / .png                        │
│ • Pencegahan Tabrakan Nama (Unique Conflict Resolution)                         │
│ • Jurnal Riwayat Transaksional untuk Fitur Undo (Safe Rollback)                 │
└───────────────────────────────────────┬─────────────────────────────────────────┘
                                        │ [Berkas Peta Bernama Standar]
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ TAHAP 3: PEMBUATAN WORLD FILE GEOREFERENSI (.jgw / .pgw)                        │
│ • Kunci Orientasi Fisik (16-Digit SLS Header Anchor pada Margin)                │
│ • Segmentasi Garis Merah Dual-Space (HSV + CIE-LAB) & Morfologi Closing 3x3     │
│ • Kalkulasi Skala Nominal Cetak (A4/A3) & Kunci Rasio Isotropik 1:1             │
│ • Chamfer Distance Transform L2 & Multi-Step Coordinate Descent                 │
│ • Ribbon Proximity Mask ±40px & Robust 75th Percentile Trimmed Consensus        │
│ • Ekspor 6-Parameter Affine ESRI World File & QGIS Layer Style (.qml)           │
└───────────────────────────────────────┬─────────────────────────────────────────┘
                                        │ [Hanya Berkas yang Terverifikasi PRESISI]
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ TAHAP 4: UNGGAH KE FOLDER SATKER (DRIVE PROTEKSI PRESISI)                       │
│ • Filter Proteksi: isItemPreciseForUpload (Tahan 'Perlu Cek' & Melenceng)       │
│ • Ekstraksi Kode Satker 4 Digit (Misal 3514 -> Kab. Pasuruan)                   │
│ • Transmisi WebDAV File Drop via Jalur HTTPS Terenkripsi (HTTP PUT)             │
│ • Kendali Antrean Pekerja (Worker Pool & Throttling) Anti-Rate Limit            │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Rincian Teknis Tahap 1 s.d. Tahap 4

### TAHAP 1: Ekstraksi & Cek ID SLS

Tujuan utama Tahap 1 adalah mengidentifikasi identitas wilayah, membaca kode SLS 16 digit, dan menemukan koordinat awal peta secara akurat dari citra scan/foto.

#### 1. Normalisasi Fisik EXIF Transpose (`normalize_image_exif_upright`)
* **Masalah:** Kamera HP dan scanner sering menyimpan citra dalam orientasi tertentu dengan menyematkan metadata `EXIF Orientation` (Tag 274 bernilai 2 s.d. 8). Webview/browser memutar gambar otomatis secara visual, tetapi driver raster GDAL pada software GIS (QGIS & ArcGIS) membaca matriks piksel mentah langsung dari disk tanpa memperhitungkan tag EXIF. Hal ini menyebabkan peta di QGIS menjadi miring 90°/180°/270° dan gepeng.
* **Solusi:** Sistem memeriksa EXIF Tag 274. Jika bernilai $\ne 1$, piksel fisik langsung ditranspose nyata ke disk (`ImageOps.exif_transpose`) dan tag EXIF dinetralkan ke 1. Matriks piksel fisik di disk, tampilan di Kalingga, dan pembacaan GDAL di QGIS dijamin **100% identik**.

#### 2. Pemindaian Barcode QR-LOC (ZXing-C++)
* Barcode Wilkerstat dipindai menggunakan *binding* lokal ZXing-C++ dengan waktu komputasi sangat cepat (15–30 ms).
* String URL yang tersimpan diekstrak menggunakan ekspresi reguler:
  ```text
  @(-?\d+\.\d+),(-?\d+\.\d+)
  ```
* Nilai lintang (*latitude*) dan bujur (*longitude*) yang diperoleh dijadikan jangkar koordinat titik tengah (*geodesic anchor*) awal peta.

#### 3. Deep Learning OCR (DBNet + SVTR berbasis ONNX Runtime)
* **DBNet (Differentiable Binarization):** Menghasilkan peta probabilitas batas teks secara diferensial untuk mengisolasi teks pada tabel legenda, kotak judul, dan nomor SLS, tahan terhadap latar belakang citra satelit yang padat.
* **SVTR (Single Visual Model for Text Recognition):** Mengenali urutan karakter alfanumerik latin dan numerik.

#### 4. Heuristik Normalisasi Karakter & Validasi 16 Digit
* Mengoreksi ambiguitas karakter cetak scanner yang sering tertukar:
  * Huruf `O`, `D`, `Q` $\to$ angka `0`
  * Huruf `I`, `l`, `|` $\to$ angka `1`
  * Huruf `S`, `s` $\to$ angka `5`
  * Huruf `B` $\to$ angka `8`
  * Huruf `Z`, `z` $\to$ angka `2`
* Kode SLS divalidasi ketat harus berupa **16 digit angka** sesuai hierarki wilayah BPS:
  $$\underbrace{\text{PP}}_{\text{Provinsi (2)}} \; \underbrace{\text{KK}}_{\text{Kab/Kota (2)}} \; \underbrace{\text{CCC}}_{\text{Kecamatan (3)}} \; \underbrace{\text{DDD}}_{\text{Desa/Kelurahan (3)}} \; \underbrace{\text{SSSS}}_{\text{Kode SLS (4)}} \; \underbrace{\text{SS}}_{\text{Sub-SLS (2)}}$$

#### 5. Fleksibilitas Metode Pemeriksaan & Deteksi Peta Tertukar
Kalingga menyediakan 3 mode pemeriksaan identitas SLS pada Tahap 1:
1. **Mode Offline (Teks Cetak Peta):**
   * Menjalankan ekstraksi OCR penuh (DBNet + SVTR) pada citra dokumen. Digunakan untuk peta tanpa barcode atau ketika berkas GeoJSON acuan belum tersedia.
2. **Mode Cepat QR Spasial (~30–80 ms per berkas):**
   * Mengutamakan pemindaian instan koordinat GPS dari barcode QR-LOC.
   * Koordinat $(\text{lat}, \text{lon})$ langsung diuji ke dalam batas poligon `Final_SLS.geojson` menggunakan algoritma *Point-in-Polygon (Ray Casting)*. Jika titik jatuh tepat di dalam poligon SLS resmi, kode SLS dan identitas wilayah langsung diperoleh dalam hitungan milidetik tanpa membebani komputasi OCR teks.
3. **Mode Verifikasi Ganda (*Dual Check* & Deteksi Peta Tertukar):**
   * Membaca teks fisik cetak sekaligus memvalidasi titik GPS barcode QR terhadap poligon GeoJSON.
   * **Deteksi Peta Tertukar (*Swapped Map*):** Jika teks fisik tertulis SLS $A$ namun titik barcode QR jatuh di dalam batas wilayah SLS $B$ (misal lembar peta tertukar saat penomoran manual di lapangan), sistem otomatis mendeteksi anomali ini, memberi label peringatan *Peta Tertukar*, dan menyajikan opsi pemilihan acuan (Gunakan Teks Cetak vs Gunakan Barcode QR) di antarmuka inspektur.
4. **Arsitektur Stabilitas Mesin AI (`run_ocr` Graceful Fallback):**
   * Menggunakan CPU Multi-Core bawaan (OpenMP 2 threads per worker) sebagai mesin utama yang terbukti stabil, hemat memori (~150 MB), dan bebas dari risiko kehabisan memori grafis (*DirectML DX12 Out-of-Memory*).
   * Dilengkapi pembungkus *automatic graceful fallback*: jika terjadi kegagalan alokasi sumber daya pada akselerator perangkat keras, mesin otomatis memproses ulang potongan citra via CPU tanpa pernah menggagalkan antrean secara diam-diam (*silent failure*).

---

### TAHAP 2: Ganti Nama Berkas (Batch Rename & Safe Rollback)

Setelah berkas tervalidasi pada Tahap 1, sistem mengganti nama berkas secara terstruktur:

1. **Standardisasi Template Nama:**
   * Nama berkas diubah mengikuti format baku: `{kode_sls}_WSS.{ext}` (contoh: `3514170017000800_WSS.jpg`).
2. **Pencegahan Tabrakan Nama & Duplikasi:**
   * Jika terdapat berkas dengan kode SLS yang sama dalam folder (misal lembar lampiran kedua), sistem memberikan sufiks pembeda otomatis tanpa menimpa berkas asli.
3. **Pencatatan Transaksional & Safe Rollback (Undo):**
   * Setiap operasi ganti nama dicatat pada sesi riwayat memori (`renameHistory`).
   * Pengguna dapat menekan tombol **"Undo Ganti Nama"** kapan saja untuk mengembalikan seluruh berkas ke nama aslinya tanpa risiko kehilangan data.

---

### TAHAP 3: Pembuatan World File Georeferensi (.jgw / .pgw)

Tahap ini memetakan koordinat piksel raster $(x, y)$ citra ke koordinat geospasial bumi WGS84 $(\text{lon}, \text{lat})$:

#### 1. Kunci Orientasi Fisik (16-Digit SLS Header Anchor)
* Algoritma memeriksa 4 strip margin dokumen (Atas, Kanan, Bawah, Kiri) dengan regex `\b\d{16}\b`.
* Jika kode SLS 16 digit terdeteksi pada margin atas, posisi tegak (0°) dikunci permanen (`orientation_locked = True`). Sistem dilarang mencoba rotasi lain saat proses fitting kontur spasial untuk mencegah kesalahan pembalikan (*false-flipping*).

#### 2. Segmentasi Batas Merah SLS (Dual Color Space)
Batas SLS pada peta Wilkerstat dicetak menggunakan garis merah. Sistem menggabungkan dua ruang warna:
* **Ruang Warna HSV:** Mengisolasi nilai rona merah pada spektrum melingkar:
  $$\text{Mask}_{\text{HSV}} = \left(H \in [0, 14] \cup [165, 180]\right) \cap (S \ge 45) \cap (V \ge 50)$$
* **Ruang Warna CIE-LAB:** Kanal $A$ memetakan sumbu hijau-merah. Ambang batas $A > 133$ sangat stabil terhadap perubahan intensitas cahaya (bayangan lipatan kertas atau pencahayaan scanner redup).
* **Morfologi Penutup (Closing):** Diterapkan kernel penutup $3 \times 3$ untuk menyambung garis batas yang tercetak putus-putus (*dashed line*).

#### 3. Kalkulasi Skala Nominal Kertas & Kunci Rasio Isotropik (1:1)
* Skala cetak dibaca dari legenda dokumen (misal `1:2500`).
* Resolusi lapangan ($R_{\text{nom}}$ dalam meter/piksel) dihitung berdasarkan ukuran kertas cetak standar BPS (A4: $297\text{ mm}$, A3: $420\text{ mm}$):
  $$\text{DPMM} = \frac{\max(W_{\text{img}}, H_{\text{img}})}{\text{Panjang Kertas (A4: 297 mm, A3: 420 mm)}}$$
  $$R_{\text{nom}} = \frac{\text{Skala} / 1000}{\text{DPMM}} \quad (\text{meter/piksel})$$
* **Batas Skala Fisik:** Skala pencarian dibatasi ketat:
  $$r_{\text{min}} = \min(r_{\text{A4}}, r_{\text{A3}}) \times 0.88, \quad r_{\text{max}} = \max(r_{\text{A4}}, r_{\text{A3}}) \times 1.15$$
* **Rasio Isotropik:** Rasio sumbu $X$ dan $Y$ dikunci seimbang ($0.98 \le R_x/R_y \le 1.02$) agar peta tidak gepeng atau lonjong.

#### 4. Pencocokan Kontur (Chamfer Distance Transform ICP)
* Geometri poligon dari `Final_SLS.geojson` diekstrak (mendukung tipe `Polygon` dan `MultiPolygon` via `extract_feature_coords`).
* Titik keliling poligon diproyeksikan ke meter lokal WGS84 di lintang $\phi$:
  $$m_{\text{lat}} = 111132.954 - 559.822 \cos(2\phi) + 1.175 \cos(4\phi)$$
  $$m_{\text{lon}} = 111412.84 \cos(\phi) - 93.5 \cos(3\phi)$$
* Kontur di-resampling merata setiap $0.8\text{ meter}$.
* Dihitung matriks jarak Euclidean L2 (*Distance Transform*) dari citra biner batas merah:
  $$D(x, y) = \min_{(x', y') \in \text{Edge}} \sqrt{(x - x')^2 + (y - y')^2}$$
* Optimasi *Multi-Step Coordinate Descent* (langkah kasar 20 px $\to$ menengah 5 px $\to$ sub-piksel 0.5 px) mencari translasi $(\Delta x, \Delta y)$ terbaik dengan nilai loss jarak terkecil.

#### 5. Verifikasi Akurasi Spasial & Robust Trimmed Consensus
* **Ribbon Proximity Mask ($\pm 40\text{ px}$):** Evaluasi akurasi dibatasi hanya pada pita selebar $\pm 40$ piksel di sepanjang garis poligon. Elemen non-peta (legenda, logo BPS, catatan tepi) otomatis diabaikan.
* **Robust 75th Percentile Trimmed Consensus:** Membuang 25% deviasi titik terluar. Jika terdapat perbedaan revisi minor pada satu sisi batas antara cetakan fisik dan data master GeoJSON, 75% sisi batas lainnya yang cocok tetap mengunci georeferensi peta dengan presisi.
* **Perhitungan Skor Akurasi & Drift:**
  $$\text{Akurasi} = \operatorname{clip}\left((100 - d_{\text{drift}} \times 2) \times (0.35 + 0.65 \times \text{Recall}), \; 0, \; 99.7\right)\%$$
  * $d_{\text{drift}}$: Rata-rata jarak titik poligon ke garis merah terdekat (piksel).
  * $\text{Recall}$: Persentase titik poligon dalam toleransi $\le 12\text{ piksel}$ dari garis merah.
* **Smart Fallback QR-LOC:** Pada lembar peta yang tidak memiliki garis batas merah internal (hanya citra satelit pemukiman), sistem menambatkan koordinat berdasarkan titik GPS barcode QR-LOC dan skala nominal cetak (toleransi $\le 250\text{ m}$).

#### 6. Transformasi Affine 6-Parameter ESRI World File (.jgw / .pgw)
Transformasi koordinat piksel citra $(x, y)$ ke koordinat geospasial WGS84 $(\text{lon}, \text{lat})$ disimpan dalam format teks 6-baris:

$$\begin{bmatrix} X_{\text{geo}} \\ Y_{\text{geo}} \end{bmatrix} = \begin{bmatrix} A & B \\ D & E \end{bmatrix} \begin{bmatrix} x \\ y \end{bmatrix} + \begin{bmatrix} C \\ F \end{bmatrix}$$

| Baris | Parameter | Keterangan Teknis |
| :---: | :---: | :--- |
| **1** | $A$ | Resolusi spasial sumbu $X$ ($\Delta \text{Lon}/\text{px}$) |
| **2** | $D$ | Komponen rotasi sumbu $Y$ ($0$ untuk dokumen tegak) |
| **3** | $B$ | Komponen rotasi sumbu $X$ ($0$ untuk dokumen tegak) |
| **4** | $E$ | Resolusi spasial sumbu $Y$ ($-\Delta \text{Lat}/\text{px}$, bernilai negatif) |
| **5** | $C$ | Bujur ($X$) titik tengah piksel kiri-atas ($x=0, y=0$) |
| **6** | $F$ | Lintang ($Y$) titik tengah piksel kiri-atas ($x=0, y=0$) |

Disertakan berkas styling `.qml` (*QGIS Layer Style*) berisi outline merah transparan untuk visualisasi langsung di QGIS.

---

### TAHAP 4: Unggah ke Folder Satker (Drive Proteksi Presisi)

Tahap akhir adalah menyetorkan berkas citra peta dan world file ke folder cloud storage Satker tujuan dengan perlindungan kualitas data ketat:

#### 1. Proteksi Validasi Presisi (`isItemPreciseForUpload`)
Sebelum berkas dimasukkan ke antrean unggah, sistem menerapkan filter verifikasi ketat:
* **Syarat Wajib:**
  1. Berkas wajib memiliki berkas `.jgw` yang valid.
  2. Status georeferensi wajib **`pas` (Presisi)** dengan skor akurasi $\ge 80\%$.
  3. Nama berkas wajib cocok dengan kode SLS 16 digit tervalidasi.
  4. Tidak ada anomali barcode tertukar (*spatial swapped/mismatch*).
* **Kebijakan Penahanan:**  
  Berkas yang masih berstatus **`ok_fallback`**, **`cukup` (50–79%)**, **`melenceng` (<50%)**, atau berstatus **Peringatan / Salah** **OTOMATIS DITAHAN (TIDAK DIUNGGAH)**. Hal ini menjamin bahwa hanya data yang sudah 100% presisi yang masuk ke repositori resmi Satker.

#### 2. Otomasi Routing per-Satker
* Sistem mengekstrak 4 digit awal kode SLS (misalnya `3514` untuk Kabupaten Pasuruan, `3509` untuk Kabupaten Jember).
* Berkas diarahkan ke endpoint folder Satker yang bersesuaian secara otomatis.

#### 3. Protokol Transmisi WebDAV (HTTPS)
* Berkas dikirim langsung menggunakan metode HTTP `PUT` ke endpoint WebDAV resmi:
  ```text
  /public.php/dav/files/{token}/{nama_file}
  ```
* Transmisi berjalan melalui jalur TLS/HTTPS terenkripsi tanpa perantara pihak ketiga.

#### 4. Manajemen Antrean & Resiliensi Jaringan (*Auto-Retry & Error Sanitization*)
* **Worker Pool & Throttling:**
  Proses unggah dibatasi menggunakan *worker pool* paralel (default 4 pekerja berurutan) dengan batas waktu timeout 5 menit per berkas. Pola ini mencegah pemblokiran lalu lintas (*rate limiting* / HTTP 429) serta menjaga stabilitas koneksi saat mengunggah ratusan dokumen sekaligus.
* **Auto-Retry Resiliency (Hitung Mundur 10 Detik):**
  Ketika koneksi ke server Satker terputus mendadak atau mengalami *socket timeout*, sistem tidak langsung menggagalkan antrean. Sistem otomatis memasuki mode *Waiting & Auto-Retry* dengan hitungan mundur 10 detik hingga maksimal 5 kali percobaan ulang.
* **Sanitasi Pesan Eror & Privasi URL:**
  Sistem menyaring pesan teknis mentah (*raw exception*, alamat URL WebDAV, token akses) dan menyajikannya dalam pesan ramah pengguna berbahasa Indonesia (misal: *"Koneksi ke server Satker terputus. Pastikan jaringan internet aktif..."*).
* **Kontrol "Lewati Menunggu" Responsif:**
  Pengguna dapat menekan tombol lewati kapan saja secara instan untuk melompati berkas bermasalah dan langsung melanjutkan pemrosesan berkas berikutnya tanpa membuat aplikasi macet atau *freeze*.

---

## 3. Arsitektur Eksekusi & Manajemen Penugasan

Selain algoritma komputasi inti per tahap, Kalingga mengimplementasikan arsitektur orkestrasi modern untuk efisiensi beban kerja skala besar:

### 1. Pemrosesan Otomatis per Berkas (*Streaming Pipeline*) vs Bertahap (*Stage-by-Stage*)
Kalingga menyediakan dua paradigma eksekusi:
1. **Mode Bertahap (*Stage-by-Stage*):**
   * Seluruh berkas diproses Tahap 1 secara batch $\to$ Pengguna meninjau hasil $\to$ Lanjut Tahap 2 $\to$ Lanjut Tahap 3 $\to$ Lanjut Tahap 4.
   * Sangat ideal untuk kontrol kualitas menyeluruh sebelum berkas diubah namanya secara fisik.
2. **Mode Otomatis per Berkas (*Streaming Pipeline*):**
   * Berkas diproses mengalir (*stream*): Berkas 1 (Tahap 1 $\to$ Tahap 2 $\to$ Tahap 3) langsung selesai, lalu berpindah ke Berkas 2.
   * **Manfaat:** Pemanfaatan *cache memory* optimal, pengurangan penumpukan I/O disk, dan pengguna langsung melihat berkas matang (*ready-to-use*) secara bertahap tanpa menunggu ribuan berkas lainnya selesai.

### 2. Deteksi & Komparasi Peta Ganda (*Duplicate Map Detection & Side-by-Side Comparison*)
* **Masalah Lapangan:** Dalam satu folder penugasan, sering terdapat lebih dari satu pindaian peta untuk SLS yang sama (misal hasil scan ulang karena scan pertama buram, atau lembar peta ganda).
* **Mekanisme Deteksi:**
  Sistem mengelompokkan berkas berdasarkan kode SLS 16 digit:
  $$\mathcal{G}(ID) = \{ B_1, B_2, \dots, B_n \} \quad \text{untuk } n \ge 2$$
* **Modal Inspeksi Komparatif (*Side-by-Side*):**
  Sistem menyediakan antarmuka perbandingan visual interaktif berdampingan:
  * Menampilkan resolusi gambar, ukuran berkas, kontras teks, dan skor akurasi georeferensi.
  * Operator dapat menentukan berkas mana yang dipertahankan (*keep primary*), berkas yang diarsipkan/dihapus, atau mengganti sufiks lampiran.

### 3. Simpan & Pindah Sesi Proyek (*Project Session Save / Resume*)
* Untuk proyek dengan ribuan peta yang dikerjakan oleh beberapa staf/operator, Kalingga menyediakan fitur simpan status proyek (`.kalingga` project bundle).
* Seluruh matriks hasil ekstraksi, posisi koordinat jgw, status verifikasi, dan riwayat revisi disimpan secara terstruktur.
* Berkas proyek dapat dipindahkan ke komputer lain (*hand-over penugasan*) dan dibuka kembali secara instan tanpa perlu menjalankan ulang OCR atau Distance Transform dari awal.

### 4. Fitur Bagi Wilayah & Arsip Peta (*Area Partitioning & Archiving*)
* **Bagi Wilayah:** Memilah ribuan peta yang tercampur dalam satu folder induk ke dalam struktur sub-folder terorganisir:
  `[Kode_Satker] / [Kecamatan] / [Desa] / {berkas_peta}`
* **Arsip Peta Otomatis:** Mengemas berkas peta terpilih beserta world file dan file style `.qml` ke dalam arsip terkompresi (.zip) siap setor atau kirim.

### 5. Pemrosesan Latar Belakang & Stabilitas UI 60 FPS
* **Isolasi Beban Komputasi:** Seluruh tugas berat (OCR, segmentasi garis merah, Chamfer distance, dan transfer WebDAV) didelegasikan ke *child process* backend (Go & Python Engine) yang berjalan asinkron.
* **Responsivitas Antarmuka:** Antarmuka web (React 19 + TypeScript) hanya menerima ringkasan metrik melalui IPC (*Inter-Process Communication*), mencegah UI *lagging* atau *freeze*.
* **Proteksi Anti-Crash (*React ErrorBoundary*):** Seluruh hierarki komponen dibungkus dengan batas penanganan galat tingkat atas (*root Error Boundary*) sehingga anomali render data parsial tidak akan membuat aplikasi keluar/crash tiba-tiba.
