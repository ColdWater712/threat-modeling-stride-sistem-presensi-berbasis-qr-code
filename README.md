# Threat Modeling Berbasis STRIDE — Sistem Presensi Perkuliahan Berbasis QR Code (Kasus A)

Repository ini berisi model arsitektur dan Data Flow Diagram (DFD) untuk mata kuliah **Keamanan Siber (CAK3CAB3)** menggunakan **OWASP Threat Dragon**.

## 1. Gambaran Umum Sistem & Alur Bisnis
Sistem presensi perkuliahan berbasis web yang memfasilitasi pencatatan kehadiran mahasiswa di kelas melalui pemindaian QR code:
1. **Pembukaan Sesi:** Dosen pengampu membuka sesi presensi untuk kelas yang diampu, lalu sistem membangkitkan QR code yang berlaku selama **5 menit** untuk ditampilkan di ruang kelas.
2. **Pemindaian & Validasi:** Mahasiswa melakukan login menggunakan akun kampus dan memindai QR code. Aplikasi mengirimkan identitas sesi dan informasi autentikasi ke **Backend Presensi**.
3. **Pencatatan Kehadiran:** Backend memverifikasi token melalui **Layanan Autentikasi Kampus**, memeriksa masa berlaku sesi serta status keanggotaan kelas, lalu menyimpan catatan kehadiran ke **Database Akademik**.
4. **Pemantauan & Rekap:** Dosen pengampu dapat memantau daftar hadir serta menutup sesi lebih awal, sementara admin akademik dapat melihat rekapitulasi kehadiran untuk seluruh kelas melalui antarmuka web.

## 2. Ruang Lingkup, Aset, dan Batas Analisis
* **Dalam Lingkup:** Pembukaan dan penutupan sesi, pembuatan dan penggunaan QR code, pengiriman permintaan presensi, validasi, penyimpanan catatan kehadiran, serta pembacaan daftar hadir dan rekap.
* **Di Luar Lingkup:** Pendaftaran mata kuliah (KRS), perubahan nilai, infrastruktur fisik, deployment cloud, kode sumber, pentest, dan eksploitasi nyata.
* **Aset yang dilindungi:** identitas, akun, dan atribut peran pengguna; token/QR code sesi presensi; data keanggotaan kelas, jadwal, dan status sesi; catatan kehadiran dan rekap presensi; hak akses dosen dan admin akademik; ketersediaan Backend Presensi dan Database Akademik; jejak audit pembukaan sesi, pemindaian, penutupan, dan pembacaan rekap.
* **Browser/aplikasi web** menjadi media interaksi pengguna dan direpresentasikan menyatu dengan aktor pada DFD.

## 3. Asumsi Kerja
| Kode | Asumsi |
| :--- | :--- |
| A1 | Backend Presensi menjadi pihak yang menentukan keputusan akhir autentikasi dan otorisasi; pemeriksaan di browser tidak dianggap cukup. |
| A2 | Detail MFA, durasi sesi login, dan pencabutan sesi pada layanan autentikasi kampus belum ditentukan. |
| A3 | QR code dapat dilihat atau difoto oleh orang yang berada di sekitar ruang kelas. |
| A4 | Mekanisme audit untuk pembukaan sesi, pemindaian, penutupan, dan pembacaan rekap belum dijelaskan. |
| A5 | Enkripsi komunikasi, pembatasan laju permintaan, dan perlindungan terhadap pengiriman ulang token belum dijelaskan. |

## 4. Arsitektur Data Flow Diagram (DFD Level 1)

### 4.1 Komponen Utama
| Tipe Elemen | Nama Elemen | Peran / Deskripsi |
| :--- | :--- | :--- |
| **Actor** | Dosen Pengampu | Membuka/menutup sesi presensi dan memantau daftar hadir kelas yang diampu. |
| **Actor** | Mahasiswa | Memindai QR code sesi dan mengirimkan bukti kehadiran. |
| **Actor** | Admin Akademik | Mengakses dan melihat rekapitulasi presensi seluruh kelas. |
| **Process** | Backend Presensi | Komponen inti yang memproses logika bisnis, validasi sesi 5 menit, dan otorisasi. |
| **Process** | Autentikasi Kampus | Layanan verifikasi kredensial, validitas token sesi, dan atribut peran pengguna. |
| **Data Store** | Database Akademik | Menyimpan data jadwal, keanggotaan kelas, sesi aktif, dan catatan kehadiran. |

### 4.2 Trust Boundary (Batas Kepercayaan)
* **Trust Boundary: Jaringan Internal Kampus** (kotak garis putus-putus) — Memisahkan zona perangkat pengguna/publik (`Dosen Pengampu`, `Mahasiswa`, `Admin Akademik`) yang berada di luar kendali penuh institusi dengan zona layanan internal kampus (`Backend Presensi`, `Autentikasi Kampus`, `Database Akademik`).

### 4.3 Daftar Aliran Data (Data Flow)
| Sumber | Tujuan | Label Aliran Data | Melintasi Trust Boundary |
| :--- | :--- | :--- | :---: |
| Dosen Pengampu | Backend Presensi | Permintaan buka/tutup sesi & token dosen | Ya |
| Backend Presensi | Dosen Pengampu | Data QR code sesi (5 menit) & daftar hadir | Ya |
| Mahasiswa | Backend Presensi | ID/token sesi QR & token autentikasi mahasiswa | Ya |
| Backend Presensi | Mahasiswa | Status konfirmasi kehadiran | Ya |
| Admin Akademik | Backend Presensi | Permintaan rekap kehadiran seluruh kelas | Ya |
| Backend Presensi | Admin Akademik | Data rekap presensi seluruh kelas | Ya |
| Backend Presensi | Autentikasi Kampus | Permintaan verifikasi token & peran pengguna | Tidak |
| Autentikasi Kampus | Backend Presensi | Status validasi sesi & atribut peran | Tidak |
| Backend Presensi | Database Akademik | Query cek kelas/sesi, simpan hadir & query rekap | Tidak |
| Database Akademik | Backend Presensi | Data jadwal, status kelas & hasil rekap | Tidak |
| Dosen Pengampu | Mahasiswa | QR Code sesi (layar kelas → browser) | Tidak |


## 5. Analisis Ancaman STRIDE

Ancaman dimasukkan langsung ke file model Threat Dragon (tautan per elemen DFD) sehingga laporan dapat di-generate melalui fitur **Report**. Skor risiko (kolom *Score*) = kemungkinan (1–3) × dampak (1–3); 1–2 rendah, 3–4 sedang, 6–9 tinggi.

| ID | Ancaman | STRIDE | Elemen DFD | Kemungkinan | Dampak | Skor | Risiko |
| :--- | :--- | :---: | :--- | :---: | :---: | :---: | :---: |
| T1 | Penggunaan akun mahasiswa yang dicuri | S | Mahasiswa (Actor) | 2 | 3 | 6 | Tinggi |
| T2 | Pembukaan sesi oleh pengguna bukan dosen | E | Backend Presensi (Process) | 2 | 3 | 6 | Tinggi |
| T3 | Pemakaian ulang atau pemalsuan token QR | T/S | Aliran "ID/token sesi QR & token autentikasi mahasiswa" | 2 | 3 | 6 | Tinggi |
| T4 | Perubahan catatan kehadiran tanpa otorisasi | T | Aliran "Query cek kelas/sesi, simpan hadir & query rekap" | 2 | 3 | 6 | Tinggi |
| T5 | Penyangkalan terhadap tindakan presensi atau perubahan data | R | Database Akademik (Data Store) | 2 | 2 | 4 | Sedang |
| T6 | Kebocoran rekap presensi kepada pihak yang tidak berwenang | I | Aliran "Data rekap presensi seluruh kelas" | 2 | 3 | 6 | Tinggi |
| T7 | Pengungkapan token melalui QR Code yang difoto | I | Aliran "QR Code sesi (layar kelas → browser)" | 3 | 3 | 9 | Tinggi |
| T8 | Banjir permintaan yang menghambat layanan | D | Aliran "ID/token sesi QR & token autentikasi mahasiswa" | 2 | 3 | 6 | Tinggi |

Setiap ancaman ditautkan ke satu elemen DFD sesuai kolom "Elemen DFD" pada laporan analisis; elemen yang ditandai merah adalah elemen yang memiliki ancaman. Deskripsi (skenario dan dampak) serta mitigasi tersedia pada isian *Description* dan *Mitigations* di model.

### Kesimpulan Evaluasi Risiko
Ancaman T7 (pengungkapan token melalui QR code yang difoto) memperoleh skor tertinggi (9) sehingga perlu mendapat perhatian utama. T1, T2, T3, T4, T6, dan T8 juga berisiko tinggi karena dapat menyebabkan presensi tidak sah, perubahan data, kebocoran informasi, atau terganggunya layanan, sedangkan T5 berisiko sedang karena dampak utamanya adalah kesulitan membuktikan tindakan pengguna saat sengketa. Mitigasi yang disusun merupakan rekomendasi kontrol keamanan yang dapat diterapkan sesuai kebutuhan sistem.
