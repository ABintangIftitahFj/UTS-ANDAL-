# README: Mesin Pencari Internal Organisasi Berbasis Web

Proyek ini adalah implementasi mesin pencari internal untuk website organisasi. Sistem melakukan penelusuran (crawling) halaman web dimulai dari satu URL awal (Seed URL) menggunakan algoritma Breadth-First Search (BFS) dan menyimpan hasilnya ke Firebase Firestore. Pengguna kemudian dapat mencari halaman berdasarkan kata kunci dan melihat rute tautan dari Seed URL ke halaman hasil.

## Fitur Utama

1.  **Konfigurasi Seed URL**: Pengguna dapat menentukan URL awal untuk penelusuran.
2.  **Penelusuran BFS**: Menggunakan algoritma BFS untuk menjelajahi halaman-halaman web.
3.  **Pembatasan Domain**: Hanya menelusuri halaman dalam domain organisasi yang sama (termasuk subdomain) dengan Seed URL.
4.  **CORS Proxy**: Menggunakan proxy publik (`api.allorigins.win`) untuk membantu mengatasi masalah CORS saat melakukan fetch halaman.
5.  **Ekstraksi Informasi**: Mengekstrak judul halaman, konten teks, dan tautan.
6.  **Pelacakan Rute Tautan**: Menyimpan dan menampilkan rute tautan dari Seed URL ke setiap halaman yang ditemukan.
7.  **Penyimpanan Data**: Hasil penelusuran (URL, judul, konten, rute) disimpan ke Firebase Firestore.
8.  **Pencarian Kata Kunci**: Pengguna dapat mencari halaman berdasarkan kata kunci.
9.  **Tampilan Hasil**: Menampilkan daftar halaman yang relevan beserta tombol untuk melihat rute tautan.
10. **Antarmuka Web**: UI sederhana dan mudah digunakan, dibangun dengan HTML dan Tailwind CSS.
11. **Konfigurasi Tambahan**: Pengaturan batas maksimal halaman yang ditelusuri dan batas maksimal hasil pencarian.
12. **Tombol Stop Crawl**: Memungkinkan pengguna menghentikan proses crawling yang sedang berjalan.

## Teknologi yang Digunakan

* **HTML**: Struktur dasar halaman web.
* **Tailwind CSS**: Framework CSS untuk styling antarmuka pengguna.
* **JavaScript (ES Modules)**: Logika utama aplikasi, termasuk:
    * Algoritma BFS.
    * Manipulasi DOM.
    * Interaksi dengan Firebase.
    * Penanganan event.
* **Firebase**:
    * **Authentication**: Untuk otentikasi anonim pengguna.
    * **Firestore**: Sebagai database NoSQL untuk menyimpan data hasil crawling.

## Penjelasan Proses Kode

Berikut adalah penjelasan mengenai alur kerja dan proses utama dalam kode:

### 1. Inisialisasi dan Otentikasi (`DOMContentLoaded`, `authenticateUser`)

* Ketika halaman selesai dimuat (`DOMContentLoaded`), fungsi `authenticateUser` dipanggil.
* `authenticateUser`: Melakukan sign-in anonim ke Firebase menggunakan `signInAnonymously` (atau `signInWithCustomToken` jika token disediakan). Ini penting agar aplikasi memiliki izin untuk berinteraksi dengan Firestore sesuai aturan keamanan. UID pengguna disimpan dalam variabel `userId`.

### 2. Konfigurasi Pengguna (UI)

* Pengguna memasukkan **Seed URL** (misal, `https://undip.ac.id`).
* Pengguna mengatur **Maks. Halaman Ditulusuri** (jumlah halaman maksimal yang akan di-crawl oleh BFS).
* Pengguna mengatur **Maks. Hasil Pencarian** (jumlah hasil maksimal yang ditampilkan saat mencari).

### 3. Proses Penelusuran (`startCrawling`)

Fungsi ini dipicu ketika tombol "Mulai Penelusuran (Crawl)" diklik.

1.  **Validasi Input**: Memeriksa apakah Seed URL telah diisi. Menambahkan skema `https://` jika belum ada.
2.  **Inisialisasi Crawler**:
    * Membersihkan log dan hasil pencarian sebelumnya.
    * Mengambil domain organisasi dari Seed URL (`getOrganizationalDomain`) untuk membatasi penelusuran.
    * Membuat antrian (`queue`) untuk BFS, diinisialisasi dengan `(seedUrl, pathAwal)`. `pathAwal` adalah array yang berisi objek `{url: seedUrl, label: "Seed URL"}`.
    * Membuat himpunan (`visitedUrls`) untuk melacak URL yang sudah dikunjungi agar tidak terjadi duplikasi atau loop tak terbatas.
    * Mengatur `crawledCount = 0` dan flag `stopCrawling = false`.
    * Menginisialisasi `batch` Firestore untuk operasi tulis berkelompok.
3.  **Loop BFS Utama**:
    * Berjalan selama `queue` tidak kosong, `crawledCount` belum mencapai batas maksimal, dan `stopCrawling` bernilai `false`.
    * Mengambil URL dan path saat ini (`currentUrl`, `currentPath`) dari `queue`.
    * Jika `currentUrl` sudah ada di `visitedUrls`, lanjut ke iterasi berikutnya. Jika tidak, tambahkan ke `visitedUrls`.
4.  **Pengambilan Halaman (Fetch dengan CORS Proxy)**:
    * URL target (`currentUrl`) digabungkan dengan URL CORS proxy publik (misalnya, `https://api.allorigins.win/raw?url=`).
    * Menggunakan `Workspace(proxiedUrl)` untuk mengambil konten HTML halaman. Penggunaan proxy bertujuan untuk menghindari masalah CORS yang sering terjadi saat melakukan fetch dari sisi klien.
5.  **Parsing HTML**:
    * Jika fetch berhasil (`response.ok`), teks HTML diambil (`response.text()`).
    * `DOMParser` digunakan untuk mengubah string HTML menjadi dokumen DOM (`doc`) yang bisa diinterogasi.
6.  **Ekstraksi Informasi dari Halaman**:
    * **Judul**: `sanitizeText(doc.title)`.
    * **Konten Teks**: `sanitizeText(doc.body ? doc.body.innerText : '')`. Teks utama dari body halaman, digunakan untuk pencocokan kata kunci.
7.  **Penyimpanan ke Firestore**:
    * Data halaman (`url` asli, `title`, `textContent`, `pathFromSeed`, `crawledAt`, `seedUrlDomain`) disiapkan.
    * URL asli di-hash (`hashString(currentUrl)`) untuk dijadikan ID dokumen di Firestore (menghindari karakter ilegal).
    * Menggunakan `batch.set()` untuk menambahkan data ke batch Firestore. Batch akan di-commit setiap 20 halaman atau di akhir proses.
8.  **Ekstraksi dan Pemrosesan Tautan**:
    * Semua tag `<a>` diekstrak dari `doc` menggunakan `doc.querySelectorAll('a')`.
    * Untuk setiap tautan:
        * URL absolut dari atribut `href` dibuat (`new URL(href, currentUrl).href`). Fragment `#` dihapus.
        * Domain organisasi dari URL tautan diperiksa (`getOrganizationalDomain`).
        * Jika tautan valid (dalam domain yang sama dan belum dikunjungi):
            * Label tautan diambil (`link.textContent`).
            * URL tautan dan path baru (`[...currentPath, { url: absoluteUrl, label: linkLabel }]`) ditambahkan ke `queue` untuk diproses BFS selanjutnya.
9.  **Penanganan Error dan Jeda**:
    * Blok `try...catch` menangani error saat fetch atau parsing. Pesan error ditampilkan di log.
    * Ada jeda singkat (`setTimeout`) antar request untuk tidak membebani server target atau proxy.
10. **Selesai/Dihentikan**:
    * Setelah loop selesai (karena antrian kosong, batas halaman tercapai, atau pengguna menekan tombol stop), sisa batch Firestore di-commit.
    * Pesan status ditampilkan di log. Tombol "Mulai" diaktifkan kembali, tombol "Stop" dinonaktifkan.

### 4. Fungsi Penghentian Penelusuran (`handleStopCrawling`)

* Dipicu oleh tombol "Hentikan Penelusuran".
* Mengatur flag `stopCrawling = true`, yang akan menghentikan loop BFS utama pada iterasi berikutnya.

### 5. Proses Pencarian (`searchPages`)

Fungsi ini dipicu ketika tombol "Cari" diklik.

1.  **Validasi Input**: Memeriksa apakah kata kunci dan Seed URL (untuk konteks domain) telah diisi.
2.  **Pengambilan Data dari Firestore**:
    * Menentukan path koleksi di Firestore berdasarkan domain organisasi dari Seed URL saat ini (`/artifacts/${appId}/public/data/web_crawler_data/${searchOrgDomain}/pages`).
    * Mengambil semua dokumen dari koleksi tersebut (`getDocs(query(collection(...)))`).
3.  **Pencocokan Kata Kunci (Client-Side Filter)**:
    * Iterasi melalui setiap dokumen yang diambil dari Firestore.
    * Untuk setiap halaman, periksa apakah `data.title` (setelah diubah ke huruf kecil) atau `data.textContent` (jika ada, setelah diubah ke huruf kecil) mengandung `keywords` (kata kunci pencarian yang juga sudah diubah ke huruf kecil).
    * Jika cocok, data halaman ditambahkan ke array `results`.
4.  **Pengurutan dan Pembatasan Hasil**:
    * Hasil diurutkan berdasarkan panjang rute (`pathFromSeed.length`) secara menaik (opsional, bisa diubah).
    * Hasil dipotong sesuai **Maks. Hasil Pencarian** yang diatur pengguna.
5.  **Penampilan Hasil**:
    * Area hasil pencarian di-clear.
    * Jika tidak ada hasil, pesan "Tidak ada hasil ditemukan" ditampilkan.
    * Jika ada hasil, setiap item hasil (judul, URL, snippet konten, tombol "Lihat Rute Link") dibuat secara dinamis sebagai elemen HTML dan ditambahkan ke area hasil pencarian.
    * Tombol "Lihat Rute Link" memiliki event listener yang memanggil `displayRouteModal` dengan data `pathFromSeed` halaman tersebut.

### 6. Tampilan Rute Tautan (`displayRouteModal`, `closeRouteModal`)

* `displayRouteModal(path)`:
    * Dipanggil saat tombol "Lihat Rute Link" diklik.
    * Menampilkan modal pop-up.
    * Mengisi modal dengan daftar tautan (URL dan labelnya) yang membentuk rute dari Seed URL ke halaman hasil.
* `closeRouteModal()`: Menyembunyikan modal.

### 7. Fungsi Pembantu (Utilities)

* `getOrganizationalDomain(url)`: Mengekstrak domain utama organisasi dari sebuah URL (misalnya, `upi.edu` dari `https://informatika.upi.edu/beranda`).
* `hashString(str)`: Membuat hash SHA-256 dari sebuah string (digunakan untuk ID dokumen Firestore).
* `sanitizeText(text)`: Membersihkan teks (menghapus spasi berlebih).
* `displayMessage(message, type)`: Menampilkan pesan di area log dengan pewarnaan sesuai tipe pesan (info, error, success).

## Cara Menjalankan

1.  **Konfigurasi Firebase**:
    * Pastikan Anda memiliki proyek Firebase yang sudah di-setup.
    * Aktifkan Firebase Authentication (Sign-in method: Anonymous).
    * Aktifkan Firebase Firestore (buat database dalam mode produksi atau tes, sesuaikan aturan keamanannya jika perlu. Aturan default biasanya memerlukan otentikasi untuk tulis/baca).
    * Jika variabel `__firebase_config` tidak disediakan oleh environment tempat Anda menjalankan kode (misalnya, di luar Canvas), Anda **harus** mengganti placeholder konfigurasi Firebase di dalam kode JavaScript (`const firebaseConfig = ...`) dengan kredensial proyek Firebase Anda.
2.  **Simpan File**: Simpan seluruh kode sebagai satu file dengan ekstensi `.html` (misalnya, `mesin_pencari.html`).
3.  **Buka di Browser**: Buka file `.html` tersebut menggunakan web browser modern (Chrome, Firefox, Edge, dll.).
4.  **Gunakan Aplikasi**:
    * Masukkan Seed URL.
    * Atur konfigurasi lainnya (Maks Halaman, Maks Hasil).
    * Klik "Mulai Penelusuran (Crawl)". Amati log.
    * Setelah crawling selesai, masukkan kata kunci dan klik "Cari".
    * Klik "Lihat Rute Link" pada hasil pencarian untuk melihat detail rute.

## Batasan dan Catatan Penting

* **CORS (Cross-Origin Resource Sharing)**: Karena penelusuran dilakukan dari sisi klien (browser), permintaan `Workspace` ke domain lain (atau subdomain yang berbeda secara origin) dapat diblokir oleh kebijakan CORS server target. Kode ini menggunakan proxy publik (`api.allorigins.win`) untuk mencoba mengatasi ini, namun keberhasilan bergantung pada proxy tersebut dan server target. Proxy publik bisa tidak stabil atau memiliki batasan.
* **Kinerja Sisi Klien**: Penelusuran banyak halaman dari browser bisa memakan waktu dan sumber daya (CPU, memori) browser. Proses ini mungkin lambat untuk website yang sangat besar.
* **Firestore**: Semua data hasil crawl disimpan di Firestore. Pastikan Anda memantau penggunaan kuota Firestore Anda.
* **Keandalan Proxy**: Proxy publik bisa berhenti bekerja atau diblokir kapan saja. Untuk solusi yang lebih andal, pertimbangkan membuat proxy CORS sendiri atau menggunakan crawler sisi server (misalnya dengan Python).
* **Pencocokan Kata Kunci Sederhana**: Pencocokan kata kunci saat ini bersifat case-insensitive substring matching. Untuk hasil yang lebih relevan, metode seperti TF-IDF atau pencarian fuzzy bisa dipertimbangkan (namun lebih kompleks untuk diimplementasikan murni di sisi klien tanpa library besar).

## Kemungkinan Pengembangan Lanjutan

* Implementasi algoritma DFS sebagai opsi.
* Integrasi metode pencocokan kata kunci yang lebih canggih.
* Penyimpanan cache hasil pencarian untuk query yang sering.
* Paging untuk hasil pencarian.
* UI yang lebih interaktif untuk status crawling (misalnya, progress bar).
* Kemampuan untuk melanjutkan crawling yang terputus.

---

Semoga README ini membantu menjelaskan alur kerja kode tersebut!