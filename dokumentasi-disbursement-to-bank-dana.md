# Dokumentasi Implementasi Disbursement to Bank — DANA

Dokumentasi ini merangkum seluruh alur, API, dan format settlement untuk fitur **Disbursement to Bank Account** DANA, ditulis ulang dalam Bahasa Indonesia agar mudah dipahami tim implementasi.

---

## 1. Apa itu Disbursement?

**Disbursement** adalah proses transfer dana dari merchant ke pengguna (user), baik ke:

1. **Disbursement to Balance** — dana masuk ke saldo akun DANA milik user (top up). Merchant bisa verifikasi akun, memicu top up, dan memantau status transaksi dalam satu integrasi.
2. **Disbursement to Bank Account** — dana ditransfer dari merchant, lewat DANA, langsung ke rekening bank milik user. Prosesnya otomatis dan minim langkah manual.

> Dokumentasi ini fokus pada **Disbursement to Bank Account**.

### Daftar API yang tersedia

| API | Method | Fungsi |
|---|---|---|
| Check Disbursement Account | POST | Cek saldo akun merchant di DANA |
| Transfer to Bank Account Inquiry | POST | Cek/validasi info rekening bank tujuan via DANA |
| Transfer to Bank | POST | Memicu permintaan transfer ke bank via DANA |
| Transfer to Bank Notify (Webhook) | POST | DANA mengirim notifikasi status transfer ke merchant |
| Transfer to Bank Inquiry Status | POST | Cek status transaksi transfer ke bank secara manual |

---

## 2. Persiapan Sebelum Mulai (Prasyarat)

Sebelum mulai integrasi, pastikan langkah berikut sudah dilakukan di **Merchant Portal** DANA:

1. Selesaikan registrasi perusahaan dan pilih **Disbursement to Bank** sebagai solusi pembayaran.
2. **Merchant Disbursement Account (MDA)** akan otomatis dibuat untuk menyimpan saldo Anda.
   - Sandbox: hubungi tim DANA untuk top up saldo testing.
   - Production: top up lewat **Virtual Account (VA)** yang tampil di Merchant Portal.
3. Konfigurasi **webhook** dan **redirect URL** untuk menerima hasil pembayaran dan mengarahkan user setelah transaksi.
4. Ambil kredensial testing (Client ID, Client Secret, Private Key, dsb.) dari Merchant Portal.

---

## 3. Alur Proses (Process Flow)

Ada dua skenario utama dalam Disbursement to Bank:

### Skenario 1: Transfer to Bank (alur normal end-to-end)

Berikut urutan lengkap interaksi antara **Merchant**, **DANA**, dan **Bank**:

**Fase A — Check Disbursement Account**
1. Merchant memanggil API **Check Disbursement Account**.
2. DANA mengembalikan informasi saldo merchant.

**Fase B — Transfer to Bank**

3. Merchant menerima permintaan transfer dari user (internal).
4. Merchant memanggil API **Transfer to Bank Account Inquiry**.
5. DANA meneruskan query informasi rekening ke Bank.
6. Bank mengirim balik hasil query.
7. DANA mengembalikan hasil inquiry ke merchant.
8. Merchant memanggil API **Transfer to Bank**.
9. DANA melakukan query ulang rekening & memotong (deduct) saldo merchant.
10. DANA meneruskan permintaan transfer ke Bank.
11. Bank memproses transfer ke rekening bank user (beneficiary).
12. DANA mengembalikan hasil (respons submission, **bukan** hasil akhir) ke merchant.
13. **Kondisional** — jika `needNotify == true`, DANA mengirim status *"Request in Progress"* ke merchant.

**Fase C — Transfer Notify**

14. Bank mengirim hasil status transfer final ke DANA.
15. DANA memanggil **Transfer to Bank Notify API** ke endpoint webhook merchant untuk memberi tahu hasil transfer.

> **Poin penting:** respons dari API **Transfer to Bank** (langkah 12) hanya mengonfirmasi bahwa **permintaan transfer telah diterima**, bukan bahwa dana sudah sampai. Status final baru dipastikan lewat **Notify** (webhook) atau **Inquiry Status** (manual check).

### Skenario 2: Transfer to Bank Inquiry Status (fallback jika timeout)

Digunakan **setelah** merchant melakukan retry pada Transfer to Bank namun tetap mendapat **timeout**:

1. Merchant memanggil API **Transfer to Bank Inquiry Status**.
2. DANA melakukan query transaksi secara internal.
3. DANA mengembalikan status transfer terbaru (`latestTransactionStatus`) ke merchant.

Gunakan skenario ini sebagai **pengaman** ketika API Transfer to Bank tidak memberi respons yang jelas dalam batas waktu (timeout) — jangan asumsikan transaksi gagal, tapi cek status sebenarnya lewat API ini.

---

## 4. Detail Tiap API

### 4.1 Check Disbursement Account

- **Endpoint:** `dana/merchant/queryMerchantResource.htm`
- **Tipe API:** DANA Open API
- **Timeout:** 8 detik
- **Fungsi:** Mengecek saldo akun disbursement merchant sebelum melakukan transfer.

**Request Body Penting:**
| Field | Tipe | Keterangan |
|---|---|---|
| `requestMerchantId` | string (21 char) | ID merchant |
| `merchantResourceInfoList` | array of string | Jenis saldo yang ingin dicek: `MERCHANT_AVAILABLE_BALANCE`, `MERCHANT_TOTAL_BALANCE`, `MERCHANT_DEPOSIT_BALANCE` |

**Contoh Request:**
```json
{
  "request": {
    "head": {
      "version": "2.0",
      "function": "dana.merchant.queryMerchantResource",
      "clientId": "201xxxx",
      "clientSecret": "201xxx",
      "reqTime": "2019-09-18T10:21:53+07:00",
      "reqMsgId": "1234567asdfasdf1123fda",
      "reserve": "{}"
    },
    "body": {
      "requestMerchantId": "216xxxxxxxxxxxxxx",
      "merchantResourceInfoList": ["MERCHANT_DEPOSIT_BALANCE"]
    }
  },
  "signature": "signature string"
}
```

**Contoh Response:**
```json
{
  "response": {
    "head": { "...": "sama seperti request head" },
    "body": {
      "resultInfo": {
        "resultStatus": "S",
        "resultCodeId": "00000000",
        "resultCode": "SUCCESS",
        "resultMsg": "success"
      },
      "merchantResourceInformations": [
        {
          "resourceType": "MERCHANT_DEPOSIT_BALANCE",
          "value": "{\"amount\":\"0\",\"currency\":\"IDR\"}"
        }
      ]
    }
  },
  "signature": "signature string"
}
```

**Result Code Penting:**
| Status | Code | Arti | Tindakan |
|---|---|---|---|
| S | `00000000` SUCCESS | Berhasil | Tandai proses sukses |
| F | `00000004` PARAM_ILLEGAL | Parameter salah | Retry dengan parameter benar |
| U | `00000900` SYSTEM_ERROR | Error sistem | Retry berkala |
| F | `12158006901` MERCHANT_NOT_EXIST | Merchant tidak ditemukan | Cek konfigurasi merchant |

---

### 4.2 Transfer to Bank Account Inquiry

- **Endpoint:** `/v1.0/emoney/bank-account-inquiry.htm`
- **Tipe API:** SNAP API
- **Timeout:** 8 detik
- **SNAP Service Code:** 42
- **Fungsi:** Memvalidasi info rekening bank tujuan **sebelum** transfer dieksekusi.

**Header Wajib:** `Content-Type`, `X-TIMESTAMP`, `X-SIGNATURE`, `X-PARTNER-ID`, `X-EXTERNAL-ID`, `CHANNEL-ID` (`Authorization` bersifat kondisional untuk pendekatan signature simetris).

**Request Body Penting:**
| Field | Wajib | Keterangan |
|---|---|---|
| `partnerReferenceNo` | - | ID transaksi unik di sistem partner. **Saat retry setelah timeout, gunakan nilai yang sama** |
| `customerNumber` | ✔ | Nomor akun customer, format `628xxx` (isi dengan nomor HP bisnis merchant) |
| `beneficiaryAccountNumber` | ✔ | Nomor rekening tujuan |
| `amount` | ✔ | Objek `{ value, currency }` |
| `additionalInfo.fundType` | ✔ | Selalu `MERCHANT_WITHDRAW_FOR_CORPORATE` untuk disbursement ke bank |
| `additionalInfo.externalDivisionId` | Kondisional | Wajib jika `chargeTarget = DIVISION` |
| `additionalInfo.chargeTarget` | - | `null` / `DIVISION` / `MERCHANT` |
| `additionalInfo.beneficiaryBankCode` | ✔ | Kode bank tujuan |
| `additionalInfo.beneficiaryAccountName` | - | Nama pemilik rekening (untuk validasi) |

**Contoh Request:**
```json
{
  "partnerReferenceNo": "2020102900000000000001",
  "customerNumber": "6281773628883",
  "beneficiaryAccountNumber": "01234567890",
  "amount": { "value": "10000.00", "currency": "IDR" },
  "additionalInfo": {
    "fundType": "MERCHANT_WITHDRAW_FOR_CORPORATE",
    "beneficiaryBankCode": "002"
  }
}
```

**Contoh Response (sukses):**
```json
{
  "responseCode": "2004200",
  "responseMessage": "Successful",
  "referenceNo": "2020102977770000000009",
  "partnerReferenceNo": "2020102900000000000001",
  "accountType": "SETTLEMENT_ACCOUNT",
  "beneficiaryAccountNumber": "01234567890",
  "beneficiaryAccountName": "JOHN DOE",
  "beneficiaryBankCode": "008",
  "beneficiaryBankShortName": "Mandiri",
  "beneficiaryBankName": "PT BANK MANDIRI (PERSERO) TBK",
  "amount": { "currency": "IDR", "value": "10000.00" },
  "additionalInfo": {
    "feeAmount": { "currency": "IDR", "value": "10000.00" }
  }
}
```

**Response Code Penting (ringkas):**
| Code | Arti | Tindakan |
|---|---|---|
| `2004200` | Sukses | Tandai proses sukses |
| `4004200`/`4004201`/`4004202` | Request/format/field salah | Retry dengan parameter benar |
| `4014200`–`4014204` | Masalah otorisasi/token | Retry dengan parameter benar |
| `4034202` | Melebihi limit jumlah transaksi | Retry dengan nominal sesuai |
| `4034214` | Saldo tidak cukup | Hubungi DANA untuk top up |
| `4034218` | Akun tidak aktif | Cek konfigurasi merchant ke DANA |
| `4034220` | Limit merchant harian terlampaui | Cek konfigurasi merchant ke DANA |
| `4044208` | Merchant tidak valid | Cek konfigurasi merchant ke DANA |
| `4044211` | Info kartu/rekening/VA tidak valid | Retry dengan info rekening yang benar |
| `4294200` | Terlalu banyak request | Tandai *pending*, retry berkala |
| `5004200`/`5004201` | Error umum/internal server | Retry proses baru |
| **Total timeout** | Tidak ada respons sama sekali | Retry maks. 3x, jika tetap gagal tandai **Pending** |
| **Unexpected response** | Field kosong/kode tidak dikenal | Jika prefix `202`/`5XX` → **Pending**; jika field kosong → **Pending** |

---

### 4.3 Transfer to Bank

- **Endpoint:** `/v1.0/emoney/transfer-bank.htm`
- **Tipe API:** SNAP API
- **Timeout:** 8 detik
- **SNAP Service Code:** 43
- **Fungsi:** Mengeksekusi transfer dana yang sesungguhnya dari akun merchant ke rekening bank user. DANA akan **memvalidasi ulang** rekening, memotong saldo merchant, lalu meneruskan permintaan ke bank.

> Response dari API ini **mengonfirmasi permintaan telah diterima**, bukan konfirmasi dana sudah sampai (completion). Status final didapat lewat Notify webhook atau Inquiry Status.

**Request Body Penting:**
| Field | Wajib | Keterangan |
|---|---|---|
| `partnerReferenceNo` | - | ID transaksi unik; **gunakan nilai sama saat retry setelah timeout** |
| `customerNumber` | - | Nomor akun customer format `628xxx` |
| `beneficiaryAccountNumber` | ✔ | Nomor rekening tujuan |
| `beneficiaryBankCode` | ✔ | Kode bank tujuan |
| `amount` | ✔ | Objek `{ value, currency }` |
| `additionalInfo.fundType` | ✔ | Selalu `MERCHANT_WITHDRAW_FOR_CORPORATE` |
| `additionalInfo.externalDivisionId` | Kondisional | Wajib jika `chargeTarget = DIVISION` |
| `additionalInfo.chargeTarget` | - | `null` / `DIVISION` / `MERCHANT` |
| `additionalInfo.needNotify` | - | Set `true` agar menerima notifikasi ketika transfer sedang diproses |
| `additionalInfo.beneficiaryAccountName` | - | Nama pemilik rekening untuk validasi |

**Contoh Request:**
```json
{
  "partnerReferenceNo": "2020102900000000000001",
  "beneficiaryAccountNumber": "01234567890",
  "beneficiaryBankCode": "002",
  "amount": { "value": "10000.00", "currency": "IDR" },
  "additionalInfo": {
    "fundType": "MERCHANT_WITHDRAW_FOR_CORPORATE",
    "needNotify": "true"
  }
}
```

**Contoh Response (sukses):**
```json
{
  "responseCode": "2004300",
  "responseMessage": "Successful",
  "referenceNo": "2020102977770000000009",
  "partnerReferenceNo": "2020102900000000000001",
  "transactionDate": "2020-12-21T17:48:41+07:00",
  "referenceNumber": "2020102977770000000009",
  "additionalInfo": {}
}
```

**Response Code Penting:**
| Code | Arti | Tindakan |
|---|---|---|
| `2004300` | Sukses | Tandai proses sukses |
| `2024300` | Transaksi sedang diproses (in progress) | Tandai **Pending**, tahan dana, tunggu notify |
| `4004300`–`4004302` | Request/field salah | Retry dengan parameter benar |
| `4014300`–`4014304` | Masalah otorisasi/token | Retry dengan parameter benar |
| `4034302` | Melebihi limit transaksi | Retry dengan nominal benar |
| `4034303` | Dugaan fraud | Hubungi DANA untuk cek status user/merchant |
| `4034314` | Saldo tidak cukup | Hubungi DANA untuk top up |
| `4034318` | Akun tidak aktif | Cek konfigurasi ke DANA |
| `4034320` | Limit merchant harian terlampaui | Cek konfigurasi ke DANA |
| `4044303` | Bank tidak didukung switch | Retry transaksi baru dengan bank lain |
| `4044311` | Info rekening/VA tidak valid | Retry dengan info rekening benar |
| `4044318` | Request tidak konsisten untuk `partnerReferenceNo` yang sama | **Tandai sukses**, hubungi DANA untuk cek status |
| `4294300` | Terlalu banyak request | Tandai **Pending**, tahan dana, retry berkala |
| `5004300` | Error umum | Retry proses baru |
| `5004301` | Internal server error | Tandai **Pending**, tahan dana, retry berkala |
| **Total timeout** | Tidak ada respons | Retry maks. 3x, jika gagal tandai **Pending** & tahan dana |
| **Unexpected response** | Field kosong/kode tak dikenal | Prefix `202`/`5XX` → **Pending** & tahan dana; field kosong → **Pending** & tahan dana |

> ⚠️ **Catatan penting soal "tahan dana" (hold the money):** pada kondisi timeout/error tak terduga, merchant **tidak boleh** langsung menganggap transaksi gagal dan mengembalikan/mencairkan dana ke tempat lain. Statusnya harus dipastikan dulu lewat **Transfer to Bank Inquiry Status**.

---

### 4.4 Transfer to Bank Notify (Webhook)

- **Endpoint (di sisi merchant):** `/v1.0/debit/emoney/transfer-bank/notify.htm` (format wajib sesuai ketentuan ASPI)
- **Tipe API:** SNAP API
- **Timeout:** 8 detik
- **SNAP Service Code:** 43
- **Fungsi:** DANA mengirim notifikasi status transfer & informasi terkait ke sistem merchant secara **asynchronous**, setelah proses bank selesai.

**Request Body (dikirim DANA ke merchant):**
| Field | Keterangan |
|---|---|
| `originalPartnerReferenceNo` | Harus sama dengan `partnerReferenceNo` yang dipakai di Transfer to Bank |
| `originalReferenceNo` | ID transaksi asli di sistem partner |
| `latestTransactionStatus` | Kode 2 karakter status (lihat tabel di bawah) |
| `transactionStatusDesc` | Deskripsi status |
| `createdTime` | Waktu transaksi dibuat |
| `finishedTime` | Waktu transaksi selesai |
| `additionalInfo` | Info tambahan |

**Kode Status Transaksi (`latestTransactionStatus`):**
| Kode | Arti |
|---|---|
| `00` | Success |
| `01` | Initiated |
| `02` | Paying |
| `03` | Pending |
| `04` | Refunded |
| `05` | Cancelled |
| `06` | Failed |
| `07` | Not found |

**Contoh Notify dari DANA:**
```json
{
  "originalPartnerReferenceNo": "2020102900000000000001",
  "originalReferenceNo": "2020102977770000000009",
  "latestTransactionStatus": "00",
  "transactionStatusDesc": "success",
  "createdTime": "2020-12-21T17:48:41+07:00",
  "finishedTime": "2020-12-21T17:50:41+07:00",
  "additionalInfo": {}
}
```

**Response yang wajib dikirim balik merchant ke DANA:**
```json
{
  "responseCode": "2004300",
  "responseMessage": "Successful"
}
```

**Pemetaan status → tindakan merchant:**
| `latestTransactionStatus` | Tindakan |
|---|---|
| `00` Success | Tandai Transfer sebagai **Success** |
| `01`/`02`/`03` (masih diproses) | Tandai **Pending**, retry berkala |
| `04` Refunded | Tandai **Failed** |
| `05` Cancelled | Tandai **Failed** |
| `06` Failed | Tandai **Failed** |
| `07` Not found | Tandai **Failed** |

Untuk error umum lainnya (`4004300`, `4014300`, `4294300`, `5004300`, dsb.), pola penanganannya sama: tandai proses Notify **Failed/Pending** dan tandai Transfer sebagai **Pending**, lalu retry sesuai jenis error.

---

### 4.5 Transfer to Bank Inquiry Status

- **Endpoint:** `/v1.0/emoney/transfer-bank-status.htm`
- **Tipe API:** SNAP API
- **Timeout:** 4 detik
- **SNAP Service Code:** 00
- **Fungsi:** Mengecek status transaksi transfer ke bank secara **manual**, biasanya dipakai ketika Transfer to Bank timeout atau notify tidak kunjung diterima.

**Mekanisme Retry (wajib):**
- Retry maksimal **5 kali**.
- Interval waktu antar retry: **5, 10, 20, 40, 60 detik**, hingga batas waktu (*cut off*) merchant.

**Request Body Penting:**
| Field | Keterangan |
|---|---|
| `originalPartnerReferenceNo` | Wajib diisi sama dengan `partnerReferenceNo` pada Transfer to Bank |
| `originalReferenceNo` | ID transaksi asli di DANA |
| `originalExternalId` | ID eksternal asli di header |
| `serviceCode` | Selalu `00` |
| `additionalInfo` | Info tambahan |

**Contoh Request:**
```json
{
  "originalPartnerReferenceNo": "2021072342358089475892734",
  "serviceCode": "00"
}
```

**Contoh Response:**
```json
{
  "responseCode": "2000000",
  "responseMessage": "Successful",
  "originalPartnerReferenceNo": "2021072342358089475892734",
  "originalReferenceNo": "2021072342358089475892091",
  "originalExternalId": "2ads-2da-d23dasd-21dadjoiq-23ij4oinfoen",
  "serviceCode": "00",
  "amount": { "value": "40000.00", "currency": "IDR" },
  "latestTransactionStatus": "00",
  "transactionStatusDesc": "Success",
  "additionalInfo": {}
}
```

**Pemetaan status → tindakan (berdasarkan `latestTransactionStatus`):**
| Kode | Arti | Tindakan |
|---|---|---|
| `00` | Success | Tandai Transfer to Bank **Success** |
| `01`/`02`/`03` | Masih diproses | Tandai **Pending**, tahan dana, retry berkala |
| `04` | Refunded | Tandai **Failed** |
| `05` | Cancelled | Tandai **Failed** |
| `06` | Failed | Tandai **Failed** |
| `07` | Not found | Tandai **Failed** |

**Response Code lain:**
| Code | Arti | Tindakan |
|---|---|---|
| `4000000`–`4000002` | Request/field salah | Tandai proses **Failed**, Transfer **Pending**, tahan dana, retry dengan parameter benar |
| `4010000`/`4010001` | Masalah otorisasi | Sama seperti di atas |
| `4040001` | Transaksi tidak ditemukan | Tandai proses & Transfer **Failed**, mulai proses baru |
| `4290000` | Terlalu banyak request | Tandai **Pending**, tahan dana, retry berkala |
| `5000001` | Internal server error | Tandai **Pending**, tahan dana, retry berkala |
| **Total timeout** | Tidak ada respons | Retry maks. 5x, jika gagal tetap **Pending**, tahan dana |
| **Unexpected response** | Field kosong/kode tak dikenal | Prefix `202`/`5XX` → **Pending**, tahan dana; field kosong → **Pending**, tahan dana |

---

## 5. Langkah Implementasi (Node.js)

### Step 1 — Instalasi Library

**Requirement:**
- Node.js versi 18 ke atas
- Kredensial testing dari Merchant Portal

```bash
npm install dana-node@latest --save
```

**Environment variable yang dibutuhkan:**
```
PRIVATE_KEY atau PRIVATE_KEY_PATH         # Private key Anda
ORIGIN                                    # URL origin aplikasi Anda
X_PARTNER_ID                              # clientId dari proses onboarding
ENV atau DANA_ENV                         # 'sandbox' atau 'production'
DANA_PUBLIC_KEY atau DANA_PUBLIC_KEY_PATH # Public key DANA untuk parsing webhook
CLIENT_SECRET                             # Client secret dari registrasi
```

### Step 2 — Inisialisasi Library

```javascript
import { Dana } from 'dana-node';

const danaClient = new Dana({
    partnerId: "YOUR_PARTNER_ID",     // process.env.X_PARTNER_ID
    privateKey: "YOUR_PRIVATE_KEY",   // process.env.X_PRIVATE_KEY
    origin: "YOUR_ORIGIN",            // process.env.ORIGIN
    env: "sandbox",                   // 'sandbox' atau 'production'
    clientSecret: "YOUR_CLIENT_SECRET",
});
const { disbursementApi } = danaClient;
```

### Step 3 — Cek Saldo Merchant

Sebelum memproses transfer bank apapun, panggil **Check Disbursement Account** untuk memastikan saldo cukup. Jika kurang, top up lewat VA di Merchant Portal.

```javascript
import { Dana } from 'dana-node';
import { QueryMerchantResourceRequest, QueryMerchantResourceResponse } from 'dana-node/merchant_management/v1';

const { merchantManagementApi } = danaClient;

const request = {
    // isi parameter request di sini
};

const response = await merchantManagementApi.queryMerchantResource(request);
```

> **Opsional:** Tambahkan dana testing lewat tab "Disbursement Account" di Merchant Portal.

### Step 4 — Validasi Rekening Bank User

Sebelum eksekusi transfer, panggil **Transfer to Bank Account Inquiry** untuk memvalidasi data rekening. Wajib isi: `customerNumber`, `beneficiaryAccountNumber`, `amount`, `additionalInfo.fundType`, `additionalInfo.beneficiaryBankCode`.

```javascript
import { BankAccountInquiryRequest, BankAccountInquiryResponse } from 'dana-node/disbursement/v1';

const request = {
  // isi field sesuai detail API Transfer to Bank Account Inquiry
};

const response = await disbursementApi.bankAccountInquiry(request);
```

### Step 5 — Eksekusi Transfer Bank

Panggil **Transfer to Bank** untuk mengeksekusi transfer dana sesungguhnya. Set `additionalInfo.needNotify = true` bila ingin menerima notifikasi status "in progress".

```javascript
import { TransferToBankRequest, TransferToBankResponse } from 'dana-node/disbursement/v1';

const request = {
  // isi field sesuai detail API Transfer to Bank
};

const response = await disbursementApi.transferToBank(request);
```

### Step 6 — Terima Notifikasi Status Transfer

DANA akan mengirim notifikasi lewat endpoint webhook merchant di format wajib:
```
/v1.0/debit/emoney/transfer-bank/notify.htm
```

Contoh payload webhook sukses:
```json
{
  "responseCode": "2004300",
  "responseMessage": "Successful"
}
```

**Opsional — Cek status manual:**
Jika tidak menerima hasil dalam waktu yang diharapkan, atau panggilan API Transfer to Bank timeout, gunakan **Transfer to Bank Inquiry Status** dengan `originalPartnerReferenceNo` dan `serviceCode`.

```javascript
import { TransferToBankInquiryStatusRequest, TransferToBankInquiryStatusResponse } from 'dana-node/disbursement/v1';

const request = {
    // isi field sesuai detail API Transfer to Bank Inquiry Status
};

const response = await disbursementApi.transferToBankInquiryStatus(request);
```

> Lihat bagian **Mekanisme Retry** pada API Transfer to Bank Inquiry Status untuk aturan retry lengkap.

### Enum Tambahan yang Tersedia di Library

Di Node.js, enum berada di masing-masing model class (bukan file terpisah), dengan penamaan mengikuti model induknya.

```javascript
import { BankAccountInquiryRequestAdditionalInfoChargeTargetEnum } from 'dana-node/disbursement/v1';

const chargeTarget = BankAccountInquiryRequestAdditionalInfoChargeTargetEnum.Division;
```

Enum yang tersedia untuk Disbursement to Bank:
- `BankAccountInquiryRequestAdditionalInfoChargeTargetEnum` (`chargeTarget`)
- `TransferToBankInquiryStatusResponseLatestTransactionStatusEnum` (`latestTransactionStatus`)
- `TransferToBankRequestAdditionalInfoChargeTargetEnum` (`chargeTarget`)

### Step 7 — Testing dengan Automated Test Suite

Regulator lokal mewajibkan pengujian integrasi mencakup semua skenario kritikal. Langkah-langkahnya:

1. Buka halaman **Integration Checklist** di Merchant Portal.
2. Selesaikan semua skenario testing wajib di checklist.
3. Setelah semua test lolos, tanda tangani **UAT Sign-Off Report** di Merchant Portal.
4. Selesaikan **Devsite Testing** lewat Merchant Portal agar sesuai standar SNAP Bank Indonesia.

> Tersedia UAT testing suite otomatis (lihat repo Github) yang dapat menjalankan seluruh skenario test dalam waktu kurang dari 15 menit.

### Step 8 — Submit Dokumen Testing & Ajukan Production

1. **Generate production keys** — buat private & public key production (lihat panduan Authentication - Production Credential).
2. **Lengkapi checklist UAT** — pastikan semua skenario testing di Merchant Portal sudah selesai.
3. **Isi Production Submission Form** — ikuti instruksi di Merchant Portal. Proses aplikasi 1–2 hari kerja.
4. **Terima kredensial production** — Merchant ID, Client ID (X-PARTNER-ID), dan Client Secret.
5. **Konfigurasi environment production** — ganti endpoint & kredensial dari sandbox ke production.
6. **Testing dengan kredensial production** — unduh template **Prod E2E Test Verification** di Merchant Portal, jalankan sesuai instruksi, dan unggah hasilnya.
7. **Live** — setelah semua approval diterima, integrasi Anda aktif dan siap menerima transaksi live dari customer.

---

## 6. Settlement File (Laporan Rekonsiliasi)

Settlement file memberi informasi detail ke merchant/Bank tentang setiap transaksi yang memengaruhi pencairan dana (settlement) ke akun mereka.

### Jenis Status Transaksi pada Settlement

| Status | Penjelasan |
|---|---|
| **Success** | Disbursement sudah menerima notifikasi sukses dari Bank. Biaya transaksi (charge amount) ditambahkan ke settlement. |
| **Pending** | Disbursement belum menerima notifikasi dari Bank selama lebih dari 1 hari (kasus yang jarang terjadi). Record tetap tampil tapi belum dikenakan biaya transaksi. |
| **Refund** | Transaksi awalnya *pending*, tapi Bank mengirim notifikasi hasil **gagal**. Karena belum dikenakan fee, `FUND_AMOUNT` dan `PAID_TOTAL_AMOUNT` dicatat sebagai **nilai negatif**. |
| **Reversal** | Transaksi awalnya **sukses**, tapi Bank mengirim notifikasi hasil gagal setelahnya. `FUND_AMOUNT`, `CHARGE_AMOUNT`, `FEE`, `VAT`, dan `PAID_TOTAL_AMOUNT` dicatat sebagai **nilai negatif**. |

### Informasi Umum File

| Atribut | Nilai |
|---|---|
| **Nama File** | `MERCHANT_DANA_DISBURSEMENT_SETTLEMENT_REPORT_MID_YYYYMMDD.csv` (`YYYYMMDD` = tanggal transaksi) |
| **Format** | `.csv` |
| **Waktu Generate** | T+1 dari tanggal transaksi |
| **Fungsi** | Menampilkan semua transaksi disbursement tercatat dalam periode tertentu |
| **Cut Off** | 00.00.00 |
| **Delimiter** | Koma (`,`) |
| **Layanan Terkait** | Disbursement |

### Struktur Field File

| Field | Tipe | Keterangan |
|---|---|---|
| `REQUEST_ID` | String (1–64 char) | ID request unik dari partner |
| `FUND_AMOUNT` | Long (1–32 char) | Nominal transaksi asli |
| `FUND_AMOUNT_CURRENCY` | String (1–3 char) | Mata uang transaksi asli |
| `CHARGE_AMOUNT` | Long (1–32 char) | Total biaya (MDR termasuk pajak) |
| `FEE` | Long (1–32 char) | Biaya dasar MDR |
| `WHT` | Long (1–32 char) | Withholding tax |
| `VAT` | Long (1–32 char) | Pajak pertambahan nilai |
| `PAID_TOTAL_AMOUNT` | Long (1–5 char) | Total nominal dibayar (termasuk MDR) |
| `PAYER_ROLE_ID` | String (1–64 char) | ID pembayar (merchant) |
| `DEPOSIT_ACCOUNT_NO` | String (1–16 char) | Nomor akun deposit |
| `MERCHANT_NAME` | String (1–64 char) | Nama merchant |
| `TRANSACTION_DATE` | Datetime (17 char) | Format `YYYYMMDD HH:MM:SS` |
| `SOURCE` | String (1–64 char) | Nama merchant (sumber) |
| `REPORT_DATE` | Datetime (8 char) | Format `YYYYMMDD` |
| `BENEFICIARY_BANK_ACCOUNT` | 1–32 char | Nomor rekening tujuan (khusus Disbursement to Bank Account) |
| `BENEFICIARY_BANK_NAME` | 1–25 char | Nama bank tujuan (khusus Disbursement to Bank Account) |

### Contoh Baris CSV per Status

**Success** (charge amount ditambahkan):
```
AQYAAIdZuL1sUk9Sttx3o33xzpdwC,6000.0,IDR,550.0,500.0,0.0,50.0,6550.0,216620000180483821815,20070000016604140817,Helo,20210429 08:31:11,Helo,20210429,,
```

**Pending** (charge amount = 0, belum dikenai fee):
```
AQYAAIdZuL1sUk9Sttx3o33xzpdwC,6000.0,IDR,0.0,0.0,0.0,0.0,6000.0,216620000180483821815,20070000016604140817,Helo,20210429 08:31:11,Helo,20210429,,
```

**Refund** (`FUND_AMOUNT` & `PAID_TOTAL_AMOUNT` negatif):
```
AQYAAIdZuL1sUk9Sttx3o33xzpdwC,-6000.0,IDR,0.0,0.0,0.0,0.0,-6000.0,216620000180483821815,20070000016604140817,Helo,20210429 08:31:11,Helo,20210429,,
```

**Reversal** (semua field nominal negatif):
```
AQYAAIdZuL1sUk9Sttx3o33xzpdwC,-6000.0,IDR,-550.0,-500.0,0.0,-50.0,-6550.0,216620000180483821815,20070000016604140817,Helo,20210429 08:31:11,Helo,20210429,,
```

> **Tips rekonsiliasi:** saat membangun proses rekonsiliasi otomatis, cek kombinasi `CHARGE_AMOUNT`/`FEE` = `0` sebagai penanda status **Pending**, dan tanda nilai negatif pada `FUND_AMOUNT` sebagai penanda **Refund** atau **Reversal** — bedakan keduanya lewat ada/tidaknya nilai negatif pada `CHARGE_AMOUNT`, `FEE`, dan `VAT`.

---

## 7. Ringkasan Praktik Terbaik (Best Practices)

1. **Selalu cek saldo dulu** lewat Check Disbursement Account sebelum inisiasi transfer, untuk menghindari kegagalan akibat saldo kurang.
2. **Selalu lakukan Account Inquiry** sebelum Transfer to Bank — ini membantu menghindari salah kirim ke rekening yang tidak valid.
3. **Gunakan `partnerReferenceNo` yang sama saat retry** setelah timeout — jangan generate ID baru, karena ini bisa menyebabkan duplikasi transaksi atau race condition.
4. **Jangan langsung anggap gagal saat timeout.** Tandai sebagai **Pending**, tahan dana, lalu konfirmasi status lewat Transfer to Bank Inquiry Status dengan mekanisme retry berjenjang (5, 10, 20, 40, 60 detik, maksimal 5x).
5. **Implementasikan endpoint webhook Notify** sesuai format wajib ASPI (`/v1.0/debit/emoney/transfer-bank/notify.htm`) agar bisa menerima update status secara real-time tanpa harus polling terus-menerus.
6. **Rekonsiliasi harian** menggunakan settlement file (T+1) untuk memvalidasi kecocokan data antara sistem internal dan laporan DANA — terutama untuk mendeteksi kasus refund/reversal yang mengubah status transaksi setelah awalnya tercatat sukses/pending.
7. **Selesaikan seluruh UAT testing checklist** di Merchant Portal sebelum mengajukan ke production, karena ini menjadi syarat wajib kepatuhan terhadap Bank Indonesia SNAP.

---

*Dokumen ini disusun berdasarkan materi API Reference dan Integration Guide Disbursement to Bank dari DANA.*
