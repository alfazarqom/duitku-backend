Duitku POP API — Create Invoice

Dokumentasi implementasi endpoint Create Invoice Duitku POP API berdasarkan dokumentasi resmi Duitku.

1. Informasi Endpoint
Konfigurasi	Nilai
Method	POST
Content-Type	application/json
Sandbox URL	https://api-sandbox.duitku.com/api/merchant/createInvoice
Production URL	https://api-prod.duitku.com/api/merchant/createInvoice

Endpoint ini digunakan untuk membuat invoice pembayaran dan mendapatkan referensi transaksi Duitku serta URL pembayaran.

2. Environment Variables

Simpan kredensial Duitku di environment variables backend.

env
DUITKU_MERCHANT_CODE=DXXXX
DUITKU_API_KEY=your_duitku_api_key
DUITKU_BASE_URL=https://api-sandbox.duitku.com


Untuk production, ubah DUITKU_BASE_URL menjadi:

env
DUITKU_BASE_URL=https://api-prod.duitku.com


Jangan menyimpan API key di frontend, repository publik, atau log yang dapat diakses pengguna.

3. Request Headers

Setiap request harus menyertakan header autentikasi berikut.

Header	Tipe	Keterangan
Content-Type	string	application/json
x-duitku-merchantcode	string	Kode merchant dari Duitku
x-duitku-timestamp	string	UNIX timestamp dalam milidetik
x-duitku-signature	string	HMAC-SHA256 signature
Generate Signature

Gunakan formula berikut sesuai dokumentasi Duitku:

text
stringToSign = merchantCode + timestamp

signature = HMAC_SHA256(stringToSign, apiKey)


Contoh implementasi menggunakan Node.js:

javascript
const crypto = require("crypto");

function generateDuitkuSignature(merchantCode, timestamp, apiKey) {
  const stringToSign = `${merchantCode}${timestamp}`;

  return crypto
    .createHmac("sha256", apiKey)
    .update(stringToSign)
    .digest("hex");
}


Timestamp harus berupa UNIX timestamp dalam milidetik. Gunakan API key sebagai secret HMAC, bukan sebagai bagian dari string yang di-hash.

4. Request Body

Berikut payload Create Invoice yang digunakan sebagai contoh.

json
{
  "paymentAmount": 40000,
  "merchantOrderId": "1648542419",
  "productDetails": "Test Pay with duitku",
  "additionalParam": "",
  "merchantUserInfo": "",
  "paymentMethod": "",
  "customerVaName": "John Doe",
  "email": "test@test.com",
  "phoneNumber": "08123456789",
  "itemDetails": [
    {
      "name": "Test Item 1",
      "price": 10000,
      "quantity": 1
    },
    {
      "name": "Test Item 2",
      "price": 30000,
      "quantity": 3
    }
  ],
  "customerDetail": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "test@test.com",
    "phoneNumber": "08123456789",
    "merchantCustomerId": "",
    "billingAddress": {
      "firstName": "John",
      "lastName": "Doe",
      "address": "Jl. Kembangan Raya",
      "city": "Jakarta",
      "postalCode": "11530",
      "phone": "08123456789",
      "countryCode": "ID"
    },
    "shippingAddress": {
      "firstName": "John",
      "lastName": "Doe",
      "address": "Jl. Kembangan Raya",
      "city": "Jakarta",
      "postalCode": "11530",
      "phone": "08123456789",
      "countryCode": "ID"
    }
  },
  "creditCardDetail": {
    "saveCardToken": 0
  },
  "callbackUrl": "https://example.com/api-pop/backend/callback.php",
  "returnUrl": "https://example.com/api-pop/backend/redirect.php",
  "expiryPeriod": 10
}

Deskripsi Parameter
Parameter	Tipe	Wajib	Keterangan
paymentAmount	integer	Ya	Nominal pembayaran
merchantOrderId	string	Ya	ID transaksi unik dari merchant, maksimal 50 karakter
productDetails	string	Ya	Deskripsi produk atau transaksi, maksimal 255 karakter
email	string	Ya	Email pelanggan
additionalParam	string	Tidak	Parameter tambahan; jika digunakan, nilainya harus URL-encoded
merchantUserInfo	string	Tidak	Informasi pengguna merchant
paymentMethod	string	Tidak	Kode metode pembayaran Duitku
customerVaName	string	Tidak	Nama pelanggan untuk tampilan konfirmasi pembayaran
phoneNumber	string	Tidak	Nomor telepon pelanggan
itemDetails	array	Tidak	Detail barang atau produk
customerDetail	object	Tidak	Informasi pelanggan dan alamat
creditCardDetail	object	Tidak	Parameter khusus transaksi kartu kredit
callbackUrl	string	Ya	URL notifikasi pembayaran server-to-server
returnUrl	string	Ya	URL tujuan setelah proses pembayaran
expiryPeriod	integer	Tidak	Masa berlaku invoice dalam menit
5. Struktur Item Details

Setiap item dalam itemDetails memiliki struktur berikut:

json
{
  "name": "Test Item 1",
  "price": 10000,
  "quantity": 1
}

Parameter	Tipe	Wajib	Keterangan
name	string	Ya	Nama barang, maksimal 50 karakter
price	integer	Ya	Harga barang tanpa desimal
quantity	integer	Ya	Jumlah barang

Catatan nominal: Dokumentasi Duitku menyatakan bahwa total nominal price pada itemDetails harus sama dengan paymentAmount. Payload contoh di atas memenuhi penjumlahan 10000 + 30000 = 40000. Namun, apabila price merupakan harga satuan yang dikalikan quantity, total kedua item menjadi 100000. Pastikan interpretasi harga sesuai dengan implementasi merchant dan konfirmasikan ke Duitku jika diperlukan sebelum production.

6. Struktur Customer Detail

Objek customerDetail berisi informasi pelanggan:

Parameter	Tipe	Keterangan
firstName	string	Nama depan
lastName	string	Nama belakang
email	string	Email pelanggan
phoneNumber	string	Nomor telepon
merchantCustomerId	string	ID pelanggan di sistem merchant
billingAddress	object	Alamat tagihan
shippingAddress	object	Alamat pengiriman

Objek alamat menggunakan struktur berikut:

json
{
  "firstName": "John",
  "lastName": "Doe",
  "address": "Jl. Kembangan Raya",
  "city": "Jakarta",
  "postalCode": "11530",
  "phone": "08123456789",
  "countryCode": "ID"
}


Parameter alamat bersifat opsional menurut dokumentasi. countryCode pada contoh menggunakan ID, sesuai contoh Duitku.

7. Contoh Implementasi Node.js

Contoh menggunakan Node.js native fetch dan crypto.

javascript
const crypto = require("crypto");

async function createDuitkuInvoice(payload) {
  const merchantCode = process.env.DUITKU_MERCHANT_CODE;
  const apiKey = process.env.DUITKU_API_KEY;
  const baseUrl = process.env.DUITKU_BASE_URL;

  if (!merchantCode || !apiKey || !baseUrl) {
    throw new Error("Konfigurasi Duitku belum lengkap");
  }

  const timestamp = Date.now().toString();

  const stringToSign = `${merchantCode}${timestamp}`;

  const signature = crypto
    .createHmac("sha256", apiKey)
    .update(stringToSign)
    .digest("hex");

  const response = await fetch(
    `${baseUrl}/api/merchant/createInvoice`,
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Accept": "application/json",
        "x-duitku-merchantcode": merchantCode,
        "x-duitku-timestamp": timestamp,
        "x-duitku-signature": signature
      },
      body: JSON.stringify(payload)
    }
  );

  const responseText = await response.text();

  let result;

  try {
    result = JSON.parse(responseText);
  } catch {
    throw new Error(
      `Respons Duitku bukan JSON. HTTP ${response.status}`
    );
  }

  if (!response.ok) {
    throw new Error(
      `Duitku API error: HTTP ${response.status} - ${
        result.statusMessage || responseText
      }`
    );
  }

  if (result.statusCode !== "00") {
    throw new Error(
      `Create invoice gagal: ${result.statusMessage}`
    );
  }

  return result;
}

module.exports = { createDuitkuInvoice };


Pemanggilan fungsi:

javascript
const { createDuitkuInvoice } = require("./duitku");

const payload = {
  paymentAmount: 40000,
  merchantOrderId: `ORDER-${Date.now()}`,
  productDetails: "Test Pay with duitku",
  email: "test@test.com",
  callbackUrl: "https://example.com/api-pop/backend/callback.php",
  returnUrl: "https://example.com/api-pop/backend/redirect.php",
  expiryPeriod: 10
};

createDuitkuInvoice(payload)
  .then((result) => {
    console.log("Reference:", result.reference);
    console.log("Payment URL:", result.paymentUrl);
  })
  .catch((error) => {
    console.error(error.message);
  });


Contoh ini hanya menyertakan parameter minimum untuk menggambarkan alur request. Lengkapi payload sesuai kebutuhan transaksi sebenarnya.

8. Response Berhasil

Contoh respons Create Invoice:

json
{
  "merchantCode": "DXXXX",
  "reference": "DXXXXS875LXXXX32IJZ7",
  "paymentUrl": "https://app-sandbox.duitku.com/redirect_checkout?reference=DXXXXS875LXXXX32IJZ7",
  "statusCode": "00",
  "statusMessage": "SUCCESS"
}

Parameter	Keterangan
merchantCode	Kode merchant Duitku
reference	Nomor referensi transaksi dari Duitku
paymentUrl	URL pembayaran
statusCode	Kode status; 00 menunjukkan sukses pada contoh dokumentasi
statusMessage	Pesan status transaksi

Setelah invoice berhasil dibuat:

Simpan merchantOrderId dan reference ke database.
Simpan nominal transaksi dan status awal pembayaran.
Tampilkan atau gunakan paymentUrl sesuai alur frontend.
Jangan menganggap transaksi sudah lunas hanya karena invoice berhasil dibuat.
9. Callback Pembayaran

callbackUrl adalah endpoint backend yang menerima notifikasi status transaksi dari Duitku.

Callback harus memproses setidaknya:

merchantCode
amount
merchantOrderId
resultCode
reference
signature

Nama parameter callback mengikuti payload yang dikirim Duitku.

Validasi Signature Callback

Dokumentasi menyediakan validasi signature callback menggunakan HMAC-SHA256 dengan string yang ditandatangani berupa gabungan:

text
merchantCode + amount + merchantOrderId


Gunakan API key sebagai secret HMAC.

Contoh:

javascript
const stringToSign =
  `${merchantCode}${amount}${merchantOrderId}`;

const expectedSignature = crypto
  .createHmac("sha256", process.env.DUITKU_API_KEY)
  .update(stringToSign)
  .digest("hex");


Bandingkan signature yang diterima dengan signature hasil perhitungan menggunakan perbandingan yang aman terhadap timing attack. Pastikan nominal, ID pesanan, dan referensi juga cocok dengan transaksi yang tersimpan.

Setelah callback terverifikasi:

Cari transaksi berdasarkan merchantOrderId.
Pastikan nominal sesuai.
Validasi signature.
Periksa resultCode sesuai dokumentasi callback.
Perbarui status transaksi secara idempotent.
Balas HTTP 200 OK setelah notifikasi berhasil diproses.

Jangan mengubah status pesanan menjadi lunas hanya berdasarkan parameter yang tidak terverifikasi. Hindari pemrosesan pembayaran ganda ketika callback dikirim ulang.

Lihat dokumentasi Callback Duitku untuk struktur request dan aturan status yang berlaku.

10. Hal yang Perlu Diperhatikan
merchantOrderId harus unik untuk setiap transaksi baru.
Duitku dapat mengembalikan respons yang sama selama sekitar lima menit untuk request dengan order ID yang sama.
Hindari mengirim banyak request Create Invoice secara bersamaan untuk order ID yang sama.
Jika pesanan berubah, ikuti aturan Duitku sebelum membuat ulang invoice.
expiryPeriod dinyatakan dalam menit.
Jangan menonaktifkan verifikasi sertifikat SSL pada koneksi HTTP API.
Jangan mengekspos API key, signature rahasia, atau data pelanggan di log publik.
Gunakan endpoint sandbox untuk pengujian dan endpoint production untuk transaksi nyata.
Untuk frontend, gunakan mekanisme integrasi Duitku POP yang sesuai dengan dokumentasi frontend.
11. Referensi
Create Invoice
Objek JSON
Callback
Payment Method
Respon API dan Error
