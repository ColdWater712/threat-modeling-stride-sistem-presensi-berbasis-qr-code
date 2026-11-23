# Threat Modeling Berbasis STRIDE — Sistem Presensi Perkuliahan Berbasis QR Code (Kasus A)

Repository ini berisi model arsitektur dan Data Flow Diagram (DFD) untuk mata kuliah **Keamanan Siber (CAK3CAB3)** menggunakan **OWASP Threat Dragon**.

## 1. Gambaran Umum Sistem & Alur Bisnis
Sistem presensi perkuliahan berbasis web yang memfasilitasi pencatatan kehadiran mahasiswa di kelas melalui pemindaian QR code:
1. **Pembukaan Sesi:** Dosen pengampu membuka sesi presensi untuk kelas yang diampu, lalu sistem membangkitkan QR code yang berlaku selama **5 menit** untuk ditampilkan di ruang kelas.
2. **Pemindaian & Validasi:** Mahasiswa melakukan login menggunakan akun kampus dan memindai QR code. Aplikasi mengirimkan identitas sesi dan informasi autentikasi ke **Backend Presensi**.
3. **Pencatatan Kehadiran:** Backend memverifikasi token melalui **Layanan Autentikasi Kampus**, memeriksa masa berlaku sesi serta status keanggotaan kelas, lalu menyimpan catatan kehadiran ke **Database Akademik**.
4. **Pemantauan & Rekap:** Dosen pengampu dapat memantau daftar hadir serta menutup sesi lebih awal, sementara admin akademik dapat melihat rekapitulasi kehadiran untuk seluruh kelas melalui antarmuka web.

## 2. Ruang Lingkup Analisis
* **Dalam Lingkup:** Pembukaan/penutupan sesi presensi, pemindaian QR code, validasi kehadiran, autentikasi pengguna, dan pembacaan rekap kehadiran.
* **Di Luar Lingkup:** Proses pendaftaran mata kuliah (KRS) dan perubahan nilai akhir mahasiswa.

## 3. Asumsi Teknis
1. **Asumsi 1:** Komunikasi jaringan antara browser klien dan server kampus menggunakan protokol HTTPS (TLS), namun payload request pada sisi klien tetap dapat diinspeksi atau dimodifikasi pengguna menggunakan alat bantu seperti proxy.
2. **Asumsi 2:** QR code yang ditampilkan dosen berisi URL atau token sesi statis yang tidak berubah selama masa berlaku 5 menit sejak sesi dibuka.
3. **Asumsi 3:** Sistem belum menerapkan validasi lokasi fisik (GPS/Geofencing) maupun pembatasan alamat IP khusus jaringan Wi-Fi ruang kelas saat pemindaian QR code.
4. **Asumsi 4:** Layanan Autentikasi Kampus menerbitkan token sesi (seperti JWT atau session cookie) setelah pengguna berhasil login yang dikirimkan pada setiap permintaan ke Backend Presensi.
5. **Asumsi 5:** Sistem belum memiliki pencatatan jejak audit (*audit log*) terpisah yang bersifat *immutable* untuk mencatat riwayat perubahan status sesi oleh dosen atau admin.

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
* **Trust Boundary: Perangkat Klien (publik) ↔ Jaringan Internal Kampus** — Memisahkan zona perangkat pengguna/publik (`Dosen Pengampu`, `Mahasiswa`, `Admin Akademik`) yang berada di luar kendali penuh institusi dengan zona layanan internal kampus (`Backend Presensi`, `Autentikasi Kampus`, `Database Akademik`).

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


## 5. Analisis Ancaman STRIDE

Ancaman dimasukkan langsung ke file model Threat Dragon (tautan per elemen DFD) sehingga laporan dapat di-generate melalui fitur **Report**. Skor risiko = kemungkinan (1–3) × dampak (1–3); 1–2 rendah, 3–4 sedang, 6–9 tinggi.

| ID | Ancaman | STRIDE | Elemen DFD | Skor | Risiko |
| :--- | :--- | :--- | :--- | :---: | :---: |
| T1 | Presensi atas nama mahasiswa lain menggunakan akun/token yang dicuri atau dititipkan | Spoofing | Mahasiswa | 6 (K2 × D3) | Tinggi |
| T2 | Pembukaan/penutupan sesi kelas oleh pengguna yang bukan dosen pengampu | Elevation of privilege | Backend Presensi | 6 (K2 × D3) | Tinggi |
| T3 | Pemalsuan, manipulasi, atau pemakaian ulang token QR sesi | Tampering | ID/token sesi QR & token autentikasi mahasiswa | 6 (K2 × D3) | Tinggi |
| T4 | Perubahan catatan kehadiran di Database Akademik tanpa otorisasi | Tampering | Database Akademik | 6 (K2 × D3) | Tinggi |
| T5 | Penyangkalan atas pembukaan/penutupan sesi, pemindaian, atau perubahan data presensi | Repudiation | Backend Presensi | 4 (K2 × D2) | Sedang |
| T6 | Kebocoran rekap/daftar hadir kepada pihak yang tidak berwenang | Information disclosure | Data rekap presensi seluruh kelas | 6 (K2 × D3) | Tinggi |
| T7 | Token sesi bocor melalui QR code yang difoto dan disebarkan | Information disclosure | Data QR code sesi (5 menit) & daftar hadir | 9 (K3 × D3) | Tinggi |
| T8 | Banjir permintaan presensi/rekap yang membuat Backend Presensi lambat atau tidak tersedia | Denial of service | Backend Presensi | 6 (K2 × D3) | Tinggi |
| T9 | Catatan hadir ganda akibat pengiriman berulang/race condition | Tampering | Query cek kelas/sesi, simpan hadir & query rekap | 4 (K2 × D2) | Sedang |
| T10 | Layanan Autentikasi Kampus tidak tersedia saat jendela presensi 5 menit | Denial of service | Autentikasi Kampus | 4 (K2 × D2) | Sedang |

Skenario, dampak, asumsi, justifikasi skor, dan mitigasi lengkap setiap ancaman tersedia pada isian *Description* dan *Mitigations* di model.
