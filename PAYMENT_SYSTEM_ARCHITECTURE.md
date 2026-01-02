# Digital Payment System Architecture
## Dokumentasi Lengkap Integrasi Sistem Pembayaran

---

## 1. ARSITEKTUR SISTEM PEMBAYARAN

### 1.1 Diagram Arsitektur Keseluruhan

```
┌─────────────────────────────────────────────────────────────────┐
│                     APPLICATION LAYER                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Frontend   │  │  Controller  │  │    Models    │           │
│  │   (Blade)    │  │  (Payment)   │  │  (Payment)   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                  PAYMENT SERVICE LAYER                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  PaymentProcessor (Interface)                            │   │
│  │  ├── BankTransferProcessor                               │   │
│  │  ├── QRISProcessor                                       │   │
│  │  ├── EWalletProcessor (GoPay, DANA, OVO, ShopeePay)     │   │
│  │  └── CardProcessor (Visa, Mastercard)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                PAYMENT GATEWAY LAYER                             │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │
│  │  Midtrans    │ │   Xendit     │ │   Doku       │             │
│  │              │ │              │ │              │             │
│  │ (BCA, BNI,  │ │ (E-Wallet,   │ │ (QRIS, Card) │             │
│  │  BRI, Card)  │ │  QRIS)       │ │              │             │
│  └──────────────┘ └──────────────┘ └──────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│          EXTERNAL PAYMENT GATEWAY API (Real-Time)               │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │
│  │  Bank API    │ │  E-Wallet    │ │  Card Networks │           │
│  │              │ │  Provider    │ │              │             │
│  │  (Real-time) │ │  (Real-time) │ │  (Real-time) │             │
│  └──────────────┘ └──────────────┘ └──────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                  DATABASE & LOGGING                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │
│  │ Payment Logs │ │ Transactions │ │ Error Logs   │             │
│  └──────────────┘ └──────────────┘ └──────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Komponen Utama

#### A. Payment Gateway Provider (Rekomendasi untuk Indonesia)

**1. Midtrans (Recommended - Most Popular)**
- ✅ Mendukung: Bank Transfer, QRIS, E-Wallet, Kartu
- ✅ Settlement: 1-2 hari kerja
- ✅ Documentation: Sangat lengkap
- ✅ Support: 24/7
- Rate: 1.5% - 3% per transaksi

**2. Xendit**
- ✅ Mendukung: E-Wallet, QRIS, Bank Transfer
- ✅ Settlement: Real-time hingga 1 jam
- ✅ Dokumentasi: Lengkap & modern
- ✅ Support: Excellent
- Rate: 1.5% - 2.9% per transaksi

**3. Doku**
- ✅ Mendukung: QRIS, Kartu, E-Wallet
- ✅ Settlement: 1 hari kerja
- ✅ Dokumentasi: Lengkap
- Rate: 1.2% - 3% per transaksi

---

## 2. DATABASE SCHEMA

### 2.1 Tabel Payments
```sql
CREATE TABLE payments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    transaction_id VARCHAR(100) UNIQUE NOT NULL,
    order_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    
    -- Payment Details
    amount BIGINT NOT NULL COMMENT '(in rupiah)',
    currency VARCHAR(3) DEFAULT 'IDR',
    payment_method VARCHAR(50) NOT NULL,
    payment_provider VARCHAR(50) NOT NULL,
    
    -- Bank Transfer Specific
    bank_code VARCHAR(10) NULL,
    bank_name VARCHAR(100) NULL,
    account_number VARCHAR(20) NULL,
    account_holder VARCHAR(100) NULL,
    
    -- QRIS Specific
    qris_string LONGTEXT NULL,
    merchant_id VARCHAR(50) NULL,
    
    -- E-Wallet Specific
    wallet_type VARCHAR(50) NULL,
    phone_number VARCHAR(20) NULL,
    
    -- Card Specific
    card_token VARCHAR(255) NULL,
    card_last_four VARCHAR(4) NULL,
    card_brand VARCHAR(20) NULL,
    
    -- Status & Tracking
    status ENUM('pending', 'processing', 'success', 'failed', 'expired', 'cancelled') DEFAULT 'pending',
    gateway_response JSON NULL,
    
    -- Reference Numbers
    reference_id VARCHAR(100) NULL,
    merchant_reference VARCHAR(100) NULL,
    
    -- Timing
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    paid_at TIMESTAMP NULL,
    expired_at TIMESTAMP NULL,
    
    -- Additional
    description TEXT NULL,
    metadata JSON NULL,
    
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_transaction_id (transaction_id),
    INDEX idx_order_id (order_id),
    INDEX idx_status (status),
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
);
```

### 2.2 Tabel Payment Logs
```sql
CREATE TABLE payment_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    payment_id BIGINT NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    status_from VARCHAR(50) NULL,
    status_to VARCHAR(50) NULL,
    
    request_payload JSON NULL,
    response_payload JSON NULL,
    error_message TEXT NULL,
    
    ip_address VARCHAR(45) NULL,
    user_agent TEXT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (payment_id) REFERENCES payments(id),
    INDEX idx_payment_id (payment_id),
    INDEX idx_event_type (event_type),
    INDEX idx_created_at (created_at)
);
```

### 2.3 Tabel Bank Accounts (Untuk Bank Transfer)
```sql
CREATE TABLE payment_bank_accounts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    bank_code VARCHAR(10) NOT NULL,
    bank_name VARCHAR(100) NOT NULL,
    account_number VARCHAR(20) NOT NULL,
    account_holder VARCHAR(100) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY unique_bank_account (bank_code, account_number),
    INDEX idx_bank_code (bank_code)
);
```

---

## 3. PAYMENT METHOD FLOW DETAILS

### 3.1 TRANSFER BANK

```
User Flow:
┌─────────────────────────────────────────────────────────────┐
│ 1. USER CHECKOUT                                            │
│    └─ Pilih "Transfer Bank"                                │
│       └─ Pilih Bank (BCA/BNI/BRI)                          │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. SYSTEM CREATE VIRTUAL ACCOUNT                            │
│    └─ POST /midtrans/create-payment                        │
│       └─ Response: Virtual Account Number + QR Code        │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. DISPLAY PAYMENT DETAILS TO USER                          │
│    └─ Virtual Account Number                               │
│    └─ Amount                                                │
│    └─ QR Code for Mobile                                    │
│    └─ Countdown Timer (typically 24 hours)                 │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. USER TRANSFER VIA BANK                                   │
│    └─ Open Mobile Banking / ATM                            │
│    └─ Transfer ke Virtual Account                          │
│    └─ Confirmation code atau receipt                       │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. BANK CONFIRMATION (REAL-TIME)                            │
│    └─ Payment Gateway menerima transfer                    │
│    └─ Webhook ke aplikasi: status = SUCCESS               │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. UPDATE ORDER STATUS                                      │
│    └─ Update payments table: status = paid                 │
│    └─ Update orders table: status = paid                   │
│    └─ Send confirmation email/SMS                          │
│    └─ Trigger fulfillment process                          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 QRIS (Quick Response Code Indonesian Standard)

```
User Flow:
┌─────────────────────────────────────────────────────────────┐
│ 1. USER CHECKOUT                                            │
│    └─ Pilih "QRIS"                                         │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. SYSTEM GENERATE QRIS CODE                                │
│    └─ POST /payment-gateway/qris/generate                  │
│    └─ Response: QRIS String + Generated QR Code Image      │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. DISPLAY QR CODE TO USER                                  │
│    └─ QR Code Image (for scanning)                         │
│    └─ QRIS String (for manual entry)                       │
│    └─ Countdown timer (15 minutes - 24 hours)              │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. USER PAY WITH ANY BANK APP                               │
│    └─ Open any bank app (BCA, BNI, BRI, etc)              │
│    └─ Scan QR Code atau input QRIS string                 │
│    └─ Konfirmasi jumlah & rekening penerima               │
│    └─ Enter PIN                                            │
│    └─ Transaction complete                                 │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. INSTANT CONFIRMATION (REAL-TIME, <5 SECONDS)            │
│    └─ Payment Gateway konfirmasi                          │
│    └─ Webhook sent immediately                            │
│    └─ User mendapat notification                          │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 E-WALLET (GoPay, DANA, OVO, ShopeePay)

```
User Flow (Redirect Method):
┌─────────────────────────────────────────────────────────────┐
│ 1. USER CHECKOUT                                            │
│    └─ Pilih E-Wallet (GoPay/DANA/OVO/ShopeePay)           │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. CREATE PAYMENT LINK                                      │
│    └─ POST /payment-gateway/ewallet/create                 │
│    └─ Request: order_id, amount, wallet_type              │
│    └─ Response: redirect_url, payment_token               │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. REDIRECT USER TO PAYMENT PAGE                            │
│    └─ Redirect to E-Wallet provider's payment page        │
│    └─ User sees: Amount + Merchant info                   │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. USER AUTHENTICATE & PAY                                  │
│    └─ User logs into E-Wallet app (GoPay/DANA/etc)       │
│    └─ User confirms payment                                │
│    └─ User enters PIN / OTP                                │
│    └─ Payment processed                                    │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. REDIRECT BACK TO APPLICATION                             │
│    └─ Success: Redirect to /payment/success?order_id=XX   │
│    └─ Failure: Redirect to /payment/failed?order_id=XX    │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. WEBHOOK CONFIRMATION                                     │
│    └─ Payment gateway sends webhook                        │
│    └─ Update payment status in DB                          │
│    └─ Verify amount & signature                            │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 KARTU PEMBAYARAN (Visa & Mastercard)

```
User Flow (Hosted Payment Page):
┌─────────────────────────────────────────────────────────────┐
│ 1. USER CHECKOUT                                            │
│    └─ Pilih "Kartu Kredit/Debit"                          │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. CREATE CARD PAYMENT TOKEN                                │
│    └─ POST /payment-gateway/card/create-token             │
│    └─ Response: token_id, payment_page_url                │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. REDIRECT TO SECURE PAYMENT PAGE                          │
│    └─ Redirect to payment gateway's hosted page           │
│    └─ Page is SSL/TLS encrypted                           │
│    └─ PCI-DSS compliant                                   │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. USER ENTER CARD DETAILS                                  │
│    └─ Card Number (without spaces)                         │
│    └─ Expiry Date (MM/YY)                                  │
│    └─ CVV/CVC                                              │
│    └─ Cardholder Name                                      │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. PROCESS PAYMENT                                          │
│    └─ Payment gateway processes securely                   │
│    └─ May trigger 3D Secure authentication                 │
│    └─ User confirms with bank's additional security       │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. RETURN TO APPLICATION                                    │
│    └─ Success: Redirect to /payment/success               │
│    └─ Failure: Redirect to /payment/failed                │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. API REQUEST & RESPONSE EXAMPLES

### 4.1 Transfer Bank

**CREATE PAYMENT REQUEST:**
```json
{
  "transaction_details": {
    "order_id": "ORDER-20260102-001",
    "gross_amount": 150000
  },
  "customer_details": {
    "first_name": "Budi",
    "last_name": "Santoso",
    "email": "budi@example.com",
    "phone": "081234567890"
  },
  "bank_transfer": {
    "bank": "bca",
    "required": true
  },
  "expiry": {
    "start_time": "2026-01-02T10:00:00+07:00",
    "duration": 86400,
    "unit": "seconds"
  }
}
```

**CREATE PAYMENT RESPONSE:**
```json
{
  "status_code": "201",
  "status_message": "Success, Bank Transfer transaction is created",
  "transaction_id": "1893a82a-7609-42f0-9ca3-6a4e70df9b5e",
  "order_id": "ORDER-20260102-001",
  "merchant_id": "M12345",
  "gross_amount": "150000.00",
  "currency": "IDR",
  "payment_type": "bank_transfer",
  "payment_method_type": "bank_transfer",
  "transaction_time": "2026-01-02 10:00:00",
  "transaction_status": "pending",
  "fraud_status": "accept",
  "va_numbers": [
    {
      "bank": "bca",
      "va_number": "91019019201"
    }
  ],
  "expiration_time": "2026-01-03 10:00:00"
}
```

### 4.2 QRIS

**GENERATE QRIS REQUEST:**
```json
{
  "amount": 150000,
  "currency": "IDR",
  "order_id": "ORDER-20260102-001",
  "merchant_id": "M12345",
  "merchant_name": "Aybi Supermarket",
  "description": "Payment for order #ORDER-20260102-001"
}
```

**GENERATE QRIS RESPONSE:**
```json
{
  "status": "success",
  "qris_string": "00020126360014com.midtrans.qr011051254060010303UME51300012com.midtrans7027302011300911201512130404081501051400506abc123456301560010A95B01062401022345670303UME5130012com.midtrans70263020120413100411C35F20300008630412123456789020C0102000050300000000000000000000000063047E7E",
  "qr_code_image": "https://gateway.example.com/api/qris/qr_code_image",
  "transaction_id": "qris-20260102-001",
  "expires_at": "2026-01-02T11:00:00+07:00"
}
```

### 4.3 E-Wallet (GoPay)

**CREATE PAYMENT LINK REQUEST:**
```json
{
  "amount": 150000,
  "currency": "IDR",
  "order_id": "ORDER-20260102-001",
  "payment_method": "gopay",
  "redirect_urls": {
    "success": "https://myapp.com/payment/success",
    "failure": "https://myapp.com/payment/failed"
  },
  "customer": {
    "name": "Budi Santoso",
    "email": "budi@example.com",
    "phone": "081234567890"
  }
}
```

**CREATE PAYMENT LINK RESPONSE:**
```json
{
  "id": "pay_12345678910",
  "status": "PENDING",
  "created_at": "2026-01-02T10:15:00+07:00",
  "expires_at": "2026-01-02T10:30:00+07:00",
  "amount": 150000,
  "currency": "IDR",
  "order_id": "ORDER-20260102-001",
  "payment_method": "gopay",
  "actions": {
    "redirect_url": "https://gateway.example.com/payment/pay_12345678910",
    "deeplink_url": "gojek://payment/pay_12345678910"
  }
}
```

### 4.4 Card Payment

**CREATE CARD PAYMENT TOKEN REQUEST:**
```json
{
  "amount": 150000,
  "currency": "IDR",
  "order_id": "ORDER-20260102-001",
  "customer": {
    "name": "Budi Santoso",
    "email": "budi@example.com"
  },
  "payment_page": {
    "display_type": "redirect",
    "return_url": "https://myapp.com/payment/callback"
  }
}
```

**CREATE CARD PAYMENT TOKEN RESPONSE:**
```json
{
  "token": "t_1234567890abc",
  "status": "ACTIVE",
  "payment_page_url": "https://gateway.example.com/pay/t_1234567890abc",
  "expires_at": "2026-01-02T11:15:00+07:00",
  "amount": 150000,
  "currency": "IDR"
}
```

---

## 5. WEBHOOK / CALLBACK MECHANISM

### 5.1 Webhook Structure

```php
POST /api/payment/webhook
Content-Type: application/json
X-Signature-Key: <calculated-signature>

{
  "transaction_time": "2026-01-02 10:05:00",
  "transaction_status": "settlement",
  "transaction_id": "1893a82a-7609-42f0-9ca3-6a4e70df9b5e",
  "status_message": "Settlement notification",
  "status_code": "200",
  "signature_key": "e0a682ecb9c9e60b2ba7b29b72d5a80ff3d7e",
  "settlement_time": "2026-01-02 10:10:00",
  "payment_type": "bank_transfer",
  "order_id": "ORDER-20260102-001",
  "merchant_id": "M12345",
  "masked_card": null,
  "gross_amount": "150000.00",
  "fraud_status": "accept",
  "currency": "IDR",
  "channel_response_code": "0000",
  "channel_response_message": "Approved",
  "card_type": null,
  "approval_code": null
}
```

### 5.2 Webhook Statuses

| Status | Meaning | Action |
|--------|---------|--------|
| `pending` | Payment belum diterima | Tunggu atau timeout |
| `settlement` | Payment berhasil diterima | Update order ke PAID |
| `deny` | Payment ditolak | Notifikasi user, retry |
| `cancel` | Payment dibatalkan | Offer untuk restart |
| `expire` | Payment kadaluarsa | Offer untuk retry |
| `failure` | Payment gagal | Notifikasi error |

### 5.3 Webhook Signature Verification

```
Signature = sha512(body + ServerKey)

Verify:
1. Concat request body dengan Server Key (rahasia)
2. Hash dengan SHA512
3. Compare dengan X-Signature-Key header
4. Jika match, webhook authentic
```

---

## 6. ERROR HANDLING & RETRY LOGIC

### 6.1 Error Categories

```
┌─────────────────────────────────────────────────────┐
│ PAYMENT ERRORS                                      │
├─────────────────────────────────────────────────────┤
│ 1. INSUFFICIENT_BALANCE                             │
│    └─ Action: User topup ewallet atau pilih method │
│                                                     │
│ 2. CARD_DECLINED                                    │
│    └─ Action: Try different card atau payment       │
│                                                     │
│ 3. INVALID_OTP / WRONG_PIN                          │
│    └─ Action: Allow retry (max 3x)                  │
│                                                     │
│ 4. BANK_UNAVAILABLE                                 │
│    └─ Action: Try later atau different bank        │
│                                                     │
│ 5. TIMEOUT (>30 detik)                              │
│    └─ Action: Check status via API, retry          │
│                                                     │
│ 6. DUPLICATE_TRANSACTION                            │
│    └─ Action: Return existing transaction result   │
│                                                     │
│ 7. FRAUD_DETECTED                                   │
│    └─ Action: Notify user, contact support         │
└─────────────────────────────────────────────────────┘
```

### 6.2 Retry Strategy

```
EXPONENTIAL BACKOFF:
Attempt 1: Immediate
Attempt 2: After 5 seconds
Attempt 3: After 25 seconds (5 * 5)
Attempt 4: After 125 seconds (25 * 5)
Max attempts: 4
Total time: ~3 minutes

Status check via API:
- If PENDING after 2 minutes: manual check
- If PENDING after 5 minutes: notify admin
- If PENDING after 24 hours: auto-expire + refund
```

---

## 7. SECURITY BEST PRACTICES

### 7.1 Data Protection

```
NEVER STORE:
❌ Card Number (full)
❌ CVV/CVC
❌ Card PIN
❌ Bank Account Password
❌ OTP/2FA Codes

ALWAYS STORE:
✅ Transaction ID
✅ Payment Status
✅ Amount
✅ Card last 4 digits (masked)
✅ Card brand (Visa, MC, etc)
✅ Tokenized card reference
```

### 7.2 Token & Signature Security

```
1. SERVER KEY (Environment Variable)
   - Store in .env file only
   - Never commit to git
   - Rotate regularly
   - Access control limited

2. SIGNATURE VERIFICATION
   - Verify every webhook
   - Use SHA512 hashing
   - Include timestamp in calculation
   - Reject old timestamps (>5 min)

3. API COMMUNICATION
   - Use HTTPS only (TLS 1.2+)
   - Certificate pinning recommended
   - Timeout: 30 seconds max
   - Rate limiting: 100 req/minute

4. WEBHOOK SECURITY
   - Whitelist IP addresses
   - Verify signature before processing
   - Idempotent processing (same webhook 2x = ok)
   - Log all webhooks
   - Implement retry with backoff
```

### 7.3 PCI-DSS Compliance

```
✅ Never handle raw card data
✅ Use payment gateway's hosted page
✅ Validate SSL certificates
✅ Encrypt sensitive data
✅ Implement access logging
✅ Regular security audits
✅ Keep dependencies updated
✅ SQL injection prevention (use prepared statements)
✅ XSS prevention (sanitize output)
✅ CSRF token verification
```

---

## 8. ENVIRONMENT CONFIGURATION

### 8.1 .env Variables

```env
# SANDBOX
PAYMENT_GATEWAY=midtrans
PAYMENT_ENV=sandbox
PAYMENT_SERVER_KEY=SB-Mid-server-xxxxxxxxxxxxxxxxxxx
PAYMENT_CLIENT_KEY=SB-Mid-client-xxxxxxxxxxxxxxxxxxx
PAYMENT_MERCHANT_ID=M12345678

# XENDIT (Alternative)
XENDIT_API_KEY=xnd_development_xxxxxxxxxxxxxxxxxxx
XENDIT_WEBHOOK_TOKEN=xnd_whsec_xxxxxxxxxxxxxxxxxxx

# CALLBACK URLS
PAYMENT_SUCCESS_URL=${APP_URL}/payment/success
PAYMENT_FAILURE_URL=${APP_URL}/payment/failed
PAYMENT_WEBHOOK_URL=${APP_URL}/api/payment/webhook

# SECURITY
PAYMENT_ENABLE_3DS=true
PAYMENT_FRAUD_DETECTION=true
PAYMENT_WEBHOOK_TIMEOUT=30

# BUSINESS LOGIC
PAYMENT_SETTLEMENT_DAYS=1
PAYMENT_EXPIRY_HOURS=24
PAYMENT_RETRY_ATTEMPTS=3
```

### 8.2 Environment-Specific Configuration

```
SANDBOX (Development)
├── Merchant ID: SB-Mid-xxxxx
├── Server Key: SB-Mid-server-xxxxx
├── Client Key: SB-Mid-client-xxxxx
├── Features: Semua available
├── Limits: Tidak ada
├── Settlement: Fake/Manual
└── Use Case: Testing & Development

STAGING (Pre-Production)
├── Merchant ID: Staging ID dari provider
├── Real bank connections
├── Real money test
├── Settlement: Real tapi rate khusus
├── Monitoring: Active
└── Use Case: Final QA sebelum production

PRODUCTION
├── Merchant ID: Live Merchant ID
├── Real money transactions
├── All security features ON
├── Fraud detection: Aggressive
├── Monitoring: 24/7
├── Support: Prioritas tinggi
└── Use Case: Live transactions
```

---

## SUMMARY & BEST PRACTICES

### Recommendation untuk Indonesian Market:

1. **Primary Gateway**: Midtrans
   - Alasan: Most popular, best support, integrated solutions

2. **Payment Methods Priority**:
   - QRIS (instant, all banks)
   - E-Wallet (popular, growing)
   - Bank Transfer (traditional, reliable)
   - Card (international support)

3. **Security**:
   - Always use HTTPS
   - Verify signatures
   - Never store raw card data
   - Keep keys in environment variables

4. **Monitoring**:
   - Log all transactions
   - Alert on failures
   - Reconcile daily with provider
   - Monitor API uptime

5. **Testing**:
   - Use sandbox thoroughly
   - Test all payment methods
   - Test error scenarios
   - Test webhook handling

---

Lanjut ke file implementasi kode! 👇
