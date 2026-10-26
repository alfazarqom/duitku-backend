# Duitku POP — Source of Truth

Dokumen ini mencatat fakta yang ditemukan pada dokumentasi resmi Duitku POP berbahasa Indonesia, memisahkannya dari keputusan desain backend dan pertanyaan yang masih perlu dikonfirmasi. Ini bukan pengganti dokumentasi resmi; periksa kembali dokumentasi sebelum perubahan implementasi karena kontrak dapat berubah.

- **Dokumentasi utama:** https://docs.duitku.com/pop/id/
- **Dokumentasi API Duitku umum:** https://docs.duitku.com/api/id/ — jangan menganggap setiap endpoint/kontrak pada dokumentasi API umum berlaku untuk POP.
- **Tanggal pemeriksaan:** 2026-10-27
- **Cakupan:** Create Invoice, request signature, HTTP Callback, frontend callback/redirect, respons dan error yang relevan.
- **Status:** catatan awal; beberapa ketidakkonsistenan resmi ditandai untuk konfirmasi.

## 1. Fakta resmi yang terdokumentasi

### 1.1 Create Invoice

Dokumentasi POP menyebut Create Invoice sebagai HTTP **POST** dengan body **application/json**.

| Lingkup | Fakta terdokumentasi |
| --- | --- |
| Endpoint sandbox | https://api-sandbox.duitku.com/api/merchant/createInvoice |
| Endpoint production | https://api-prod.duitku.com/api/merchant/createInvoice |
| Header | Content-Type: application/json |
| Header | x-duitku-timestamp — timestamp UNIX dalam milidetik |
| Header | x-duitku-signature |
| Header | x-duitku-merchantcode |

Dokumentasi menyebut timestamp dalam milidetik pada zona waktu Jakarta. Formula signature request yang ditampilkan:

~~~text
stringToSign = merchantCode + timestamp
signature = HMAC_SHA256(stringToSign, apiKey)
~~~

Parameter Create Invoice yang ditandai wajib pada tabel dokumentasi:

| Parameter | Tipe / batas yang tercantum | Catatan |
| --- | --- | --- |
| paymentAmount | integer | Nominal pembayaran |
| merchantOrderId | string(50) | Order ID merchant |
| productDetails | string(255) | Detail produk |
| email | string(255) | Email pelanggan merchant |
| callbackUrl | string(255) | URL untuk callback transaksi |
| returnUrl | string(255) | URL redirect setelah transaksi selesai atau dibatalkan |

Parameter opsional yang disebut pada tabel Create Invoice antara lain additionalParam, merchantUserInfo, customerVaName, phoneNumber, itemDetails, customerDetail, expiryPeriod, paymentMethod, dan creditCardDetail. Persyaratan dapat berbeda untuk kanal tertentu; dokumentasi menyebut customerDetail dan itemDetails wajib untuk kanal kredit, serta customerVaName wajib untuk kanal E-Commerce.

Dokumentasi juga menyatakan nominal paymentAmount harus setara dengan jumlah nominal pada itemDetails ketika parameter tersebut digunakan. Jangan mengasumsikan itemDetails otomatis menggantikan perhitungan total order merchant.

### 1.2 Respons Create Invoice dan halaman pembayaran

Dokumentasi menjelaskan bahwa respons Create Invoice menyediakan nomor referensi Duitku dan URL pembayaran, serta menampilkan contoh properti seperti paymentUrl, reference, statusCode, dan statusMessage.

Dokumentasi POP menjelaskan dua cara menampilkan halaman pembayaran:
- Menggunakan Duitku JS dengan checkout.process(duitkuReference, options).
- Mengarahkan pengguna ke nilai paymentUrl dari respons Create Invoice.

Nama properti dan contoh di atas mengikuti contoh dokumentasi; implementasi perlu memvalidasi respons aktual dan tidak boleh menganggap setiap properti selalu ada tanpa memeriksa kontrak respons yang relevan.

### 1.3 HTTP Callback

Dokumentasi menyebut HTTP Callback sebagai notifikasi server-ke-server setelah pelanggan melakukan pembayaran. Spesifikasi yang ditampilkan:

- Method: HTTP POST.
- Content type: application/x-www-form-urlencoded.
- Endpoint callback merchant harus dapat diakses secara publik.
- Dokumentasi meminta callback mengembalikan HTTP 200 OK.
- Duitku menyatakan akan mengirim ulang callback jika belum menerima HTTP 200, hingga maksimum 5 percobaan; setelah maksimum tersebut, notifikasi kegagalan callback dikirim melalui email. Dokumentasi menyebut fitur resend tersedia pada dashboard Duitku.

Parameter callback yang didokumentasikan meliputi:

| Parameter | Keterangan resmi |
| --- | --- |
| merchantCode | Kode merchant Duitku |
| amount | Nominal pembayaran |
| merchantOrderId | Order ID merchant |
| productDetail | Detail produk (singular pada dokumentasi callback) |
| additionalParam | Parameter tambahan, opsional |
| paymentCode | Metode pembayaran |
| resultCode | Status transaksi; terdapat ketidakkonsistenan pada dokumentasi, lihat §4 |
| merchantUserId | User ID merchant |
| reference | Nomor referensi Duitku; dokumentasi menyarankan menyimpannya untuk pengecekan transaksi |
| signature | Signature callback |
| publisherOrderId | Nomor unik pembayaran dari Duitku; dokumentasi menyarankan menyimpannya untuk pencatatan/pelacakan |

Dokumentasi juga mencantumkan parameter tambahan yang dapat dikirim, antara lain spUserHash, settlementDate, issuerCode, customerName, bankAppCode, bankOrderId, bankRespCode, bankRespMsg, cardName, cardType, maskedNumber, tokenId, transactionState, transactionStateStatus, merchantCustomerId, dan expiryDate. Tidak semua parameter berlaku pada setiap kanal pembayaran.

Formula signature callback yang ditampilkan:

~~~text
stringToSign = merchantCode + amount + merchantOrderId
signature = HMAC_SHA256(stringToSign, apiKey)
~~~

Dokumentasi menyatakan metode signature lama berbasis MD5 sudah usang (obsolete). Contoh resmi membandingkan signature yang dihitung dengan signature yang diterima; implementasi harus menggunakan perbandingan yang sesuai untuk validasi signature dan tidak memproses perubahan status sebelum validasi berhasil.

### 1.4 Frontend callback dan redirect bukan sumber status pembayaran

Dokumentasi POP secara eksplisit menyatakan respons JavaScript callback hanya boleh dipakai untuk notifikasi kepada pelanggan dan tidak diperkenankan mengubah status pada sistem merchant. Untuk tujuan itu, dokumentasi mengarahkan merchant ke HTTP Callback.

Dokumentasi redirect membedakan hasil redirect, termasuk 00 success, 01 process/belum terbayar, dan 02 canceled/failed. Nilai redirect ini tidak boleh dicampuradukkan dengan kode hasil pada HTTP Callback.

### 1.5 HTTP dan API error yang terdokumentasi

Dokumentasi mencantumkan respons HTTP berikut:

| Kode | Keterangan pada dokumentasi |
| --- | --- |
| 400 | Bad Request; termasuk nominal berbeda saat request ulang dengan order ID yang sama |
| 401 | Unauthorized |
| 404 | Not Found |
| 409 | Transaksi masih diproses; request ulang dengan order ID yang sama saat request pertama masih berlangsung |
| 500 | Internal Server Error |

Dokumentasi juga mencatat bahwa mengirim ulang atau membuat invoice dengan merchantOrderId yang sama pada saat bersamaan, tetapi dengan nominal berbeda, dapat menghasilkan error. Perlakuan idempotensi dan retry pada sisi merchant tetap harus dirancang agar tidak membuat transaksi ganda atau mengubah nominal invoice yang sudah ada.

### 1.6 Changelog dokumentasi yang relevan

Pada halaman POP yang diperiksa, changelog mencantumkan:
- Juni 2026: penambahan parameter callback customerName.
- Februari 2026: penambahan kanal Tokopedia Card Payment, Tokopedia E-Wallet, dan Tokopedia Others.
- Desember 2025: penambahan persyaratan callback.
- Oktober 2025: penambahan parameter terkait tokenisasi dan pelanggan.

Periksa changelog terbaru sebelum menambahkan parameter atau kanal pembayaran.

## 2. Interpretasi desain backend (bukan kutipan kontrak Duitku)

Keputusan berikut adalah rekomendasi implementasi berdasarkan fakta di atas, bukan field atau persyaratan yang diklaim berasal langsung dari Duitku:

1. **Rahasiakan API key.** Simpan kredensial di secret/environment backend Supabase; jangan kirim API key ke browser, jangan commit nilainya, dan jangan mencatatnya ke log.
2. **Validasi request Create Invoice di backend.** Ambil total dari data order tepercaya di database, bukan mempercayai jumlah dari browser. Cocokkan order ID, nominal, mata uang dan status order sesuai model data yang disepakati.
3. **Validasi callback sebelum mutasi status.** Parse format form-urlencoded sesuai spesifikasi, validasi field wajib, verifikasi signature HMAC-SHA256, lalu cocokkan merchant code, order ID, nominal, dan referensi dengan transaksi yang disimpan.
4. **Idempotensi callback.** Callback dapat dicoba ulang. Terapkan perubahan status secara atomik dan idempoten; callback yang sama tidak boleh menggandakan efek bisnis.
5. **Jangan mengandalkan frontend.** JavaScript callback, URL return, parameter redirect, atau halaman sukses hanya untuk pengalaman pengguna; status pembayaran final harus ditentukan oleh backend melalui jalur callback yang tervalidasi dan rekonsiliasi tepercaya.
6. **Gunakan allowlist endpoint per environment.** Pilih endpoint sandbox atau production dari konfigurasi server yang eksplisit. Jangan membiarkan input publik memilih URL tujuan request ke Duitku.
7. **Penanganan retry yang aman.** Untuk timeout, 409, atau respons ambigu, periksa state transaksi sebelum membuat request ulang. Jangan membuat invoice baru atau mengganti nominal secara otomatis.
8. **Respons callback 200 hanya setelah pemrosesan aman.** Pastikan definisi “berhasil diproses” dan strategi retry internal jelas. Jangan mengembalikan sukses untuk signature invalid atau event yang gagal dipersistenkan.
9. **Minimalkan data sensitif.** Simpan hanya data callback yang dibutuhkan untuk audit dan rekonsiliasi; batasi akses dan redaksi data sensitif dari log.
10. **Tes sandbox terlebih dahulu.** Uji signature valid/tidak valid, callback berulang, nominal/order ID tidak cocok, hasil gagal, timeout, dan respons HTTP sebelum menyatakan integrasi siap produksi.

## 3. Aturan sumber kebenaran untuk repository ini

Saat mengembangkan file backend:
- Gunakan dokumentasi **Duitku POP** sebagai acuan kontrak POP.
- Pisahkan fakta kontrak dari pilihan arsitektur Supabase dan aturan bisnis merchant.
- Jangan menebak field wajib, enum status, response body, retry semantics, atau endpoint yang tidak dinyatakan jelas.
- Jika ada konflik dalam halaman resmi, dokumentasikan konflik tersebut dan minta konfirmasi Duitku atau bukti kontrak yang lebih spesifik sebelum mengunci perilaku produksi.
- Catat tanggal pemeriksaan dan tautan ke bagian sumber yang relevan.
- Dokumentasi resmi yang tersedia dapat berubah; ulangi pemeriksaan saat implementasi dan sebelum rilis.

## 4. Ambiguitas dan hal yang perlu dikonfirmasi

### 4.1 Ketidakkonsistenan resultCode callback

Halaman POP yang sama tampak tidak konsisten:
- Tabel parameter pada bagian HTTP Callback mencantumkan 00 - Success dan 01 - Failed.
- Bagian **Errors → Respon API → Callback** mencantumkan 00 - Success dan 02 - Failed.
- Bagian **Errors → Respon API → Redirect** mencantumkan 00 - Success, 01 - Process, dan 02 - Canceled/Failed.
- Bagian frontend JS menjelaskan kode pada konteks callback browser: 00 sukses, 01 pending, 02 popup ditutup tanpa menyelesaikan pembayaran.

Jangan menyimpulkan sendiri apakah HTTP Callback benar-benar dapat mengirim 01, 02, atau keduanya untuk status gagal/pending. Sebelum memetakan semua kode callback ke status internal, konfirmasikan kontrak HTTP Callback aktif kepada Duitku atau gunakan spesifikasi merchant yang eksplisit dan terkini. Sampai dikonfirmasi, jangan menyamakan kode redirect/JS dengan HTTP Callback.

### 4.2 Perbedaan nama field produk

Contoh Create Invoice memakai productDetails (plural), sedangkan parameter HTTP Callback mencantumkan productDetail (singular). Perlakukan keduanya sebagai konteks yang berbeda; jangan mengubah nama field callback agar menyerupai request invoice.

### 4.3 Detail yang belum dapat dipastikan hanya dari halaman ini

Konfirmasikan atau verifikasi dengan pengujian sandbox/dokumentasi merchant yang relevan sebelum implementasi final:
- Apakah ada batas toleransi timestamp atau aturan penolakan timestamp lama/masa depan untuk header request.
- Apakah perbandingan signature memiliki aturan format/case khusus di luar contoh HMAC-SHA256 hex yang ditampilkan.
- Semua variasi payload callback untuk setiap kanal dan apakah field tertentu selalu hadir atau hanya kondisional.
- Kontrak respons lengkap Create Invoice dan variasi respons error untuk semua kasus.
- Semantik tepat untuk callback yang signature-nya valid tetapi order/nominal tidak cocok, termasuk respons HTTP yang diharapkan.
- Kebijakan retry/reconciliation merchant untuk timeout saat Create Invoice jika Duitku mungkin sudah menerima request.
- Kebijakan perubahan nominal, invoice kedaluwarsa, pembayaran terlambat, refund, dan transaksi ganda di tingkat merchant; hal ini memerlukan keputusan bisnis merchant, bukan sekadar menyalin dokumentasi POP.

## 5. Referensi resmi

1. [Duitku POP — API Reference (Bahasa Indonesia)](https://docs.duitku.com/pop/id/) — bagian Integrasi Backend atau Server → Create Invoice, Callback; Integrasi Frontend atau View → JS Callback dan Window Redirection; Errors; Changelog.
2. [Duitku API documentation](https://docs.duitku.com/api/id/) — dokumentasi API umum. Gunakan hanya bila produk/endpoint yang dimaksud cocok; halaman ini bukan pengganti spesifikasi POP.
