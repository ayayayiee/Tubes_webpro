# Payment System: Best Practices & Troubleshooting

---

## BEST PRACTICES KEAMANAN

### 1. NEVER DO THIS ❌
```php
// JANGAN PERNAH SIMPAN KARTU MENTAH
$cardNumber = "4111111111111111";
$cvv = "123";
DB::table('cards')->insert([
    'number' => $cardNumber,
    'cvv' => $cvv
]);

// JANGAN COMMIT KEYS KE GIT
echo "Server Key: SB-Mid-server-xxxxx"; // Jangan!

// JANGAN PROCESS WEBHOOK TANPA VERIFY
if ($payload['status'] === 'settlement') {
    $payment->markAsPaid(); // DANGEROUS!
}

// JANGAN TRUST CLIENT DATA
$payment = Payment::find($request->payment_id);
// Jangan langsung update tanpa verify!
```

### 2. ALWAYS DO THIS ✅
```php
// SELALU GUNAKAN TOKENIZED PAYMENT
$payment = Payment::create([
    'card_token' => $tokenFromGateway, // Bukan kartu asli!
    'card_last_four' => '1111',
    'card_brand' => 'visa'
]);

// SELALU VERIFY WEBHOOK SIGNATURE
if (!$this->gateway->verifySignature($payload)) {
    return response()->json(['error' => 'Invalid'], 403);
}

// SELALU AUTHORIZE USER
$this->authorize('pay', $order);

// SELALU ENCRYPT SENSITIVE DATA
$encrypted = encrypt($sensitiveData);

// SELALU LOG SEMUA TRANSAKSI
PaymentLog::create([
    'payment_id' => $payment->id,
    'request' => $request,
    'response' => $response
]);

// SELALU USE HTTPS
'url' => 'https://your-api.com/callback' // Not http://
```

---

## ERROR HANDLING SCENARIOS

### Scenario 1: Payment Timeout (Customer Waited Too Long)

```
Problem:
- User submit form
- Network slow / API timeout
- User confused, doesn't know if payment sent
- Duplicate payment risk

Solution:
1. Store transaction_id BEFORE sending to gateway
2. Show spinner with countdown
3. Allow manual verification button
4. Check status via API after 10 seconds
5. If still pending, show "Checking payment status..."
6. If confirmed, show success
7. If failed, allow retry

Implementation:
```php
// routes/web.php
Route::post('/payment/initialize', [PaymentController::class, 'initialize'])->name('payment.initialize');

// app/Http/Controllers/PaymentController.php
public function initialize(Request $request)
{
    // 1. Create payment record FIRST
    $payment = Payment::create([
        'transaction_id' => 'ORD-' . time(),
        'status' => 'pending'
    ]);

    // 2. Then send to gateway
    try {
        $processor = PaymentProcessorFactory::create($payment);
        $result = $processor->create();

        return response()->json([
            'success' => true,
            'payment_id' => $payment->id,
            'check_status_url' => route('payment.verify', $payment->id)
        ]);
    } catch (\Exception $e) {
        // Payment still created, user can check later
        return response()->json([
            'success' => false,
            'payment_id' => $payment->id,
            'check_status_url' => route('payment.verify', $payment->id),
            'message' => 'Payment initiated but verify status manually'
        ]);
    }
}

// Frontend
async function checkPaymentStatus(paymentId) {
    let attempts = 0;
    const maxAttempts = 30; // 30 seconds
    
    const interval = setInterval(async () => {
        attempts++;
        
        const response = await fetch(`/payment/${paymentId}/verify`);
        const result = await response.json();
        
        if (result.status === 'paid') {
            clearInterval(interval);
            window.location.href = '/payment/success?order_id=' + orderId;
        } else if (attempts >= maxAttempts) {
            clearInterval(interval);
            showMessage('Payment still pending. You can check later.');
        }
    }, 1000);
}
```

### Scenario 2: Duplicate Transaction (User Clicked "Pay" Twice)

```
Problem:
- User impatient, click submit multiple times
- Two VA created, two charge requests sent
- Bank deducts money twice

Solution:
Implement idempotent payment creation

Implementation:
```php
public function initialize(Request $request)
{
    $validated = $request->validate([
        'order_id' => 'required|exists:orders,id'
    ]);

    // Check if payment already exists for this order
    $existingPayment = Payment::where('order_id', $validated['order_id'])
        ->where('status', 'pending')
        ->first();

    if ($existingPayment) {
        // Return existing payment instead of creating new
        return response()->json([
            'success' => true,
            'payment_id' => $existingPayment->id,
            'message' => 'Using existing payment'
        ]);
    }

    // Create new payment if none exist
    $payment = Payment::create([...]);
    // ...
}
```

### Scenario 3: Bank Transfer Mismatched Amount

```
Problem:
- Virtual Account created for Rp 150.000
- User transfers Rp 150.001
- Payment gateway rejects (many banks are strict)
- User stuck, money not processed

Solution:
Allow small tolerance (typically 0-10000 rupiah)

Implementation:
```php
public function webhook(Request $request)
{
    $payload = $request->all();
    $payment = Payment::find($payload['order_id']);
    
    $expectedAmount = $payment->amount;
    $receivedAmount = $payload['gross_amount'];
    $tolerance = 10000; // Rp 10.000
    
    // Accept if within tolerance
    if (abs($expectedAmount - $receivedAmount) <= $tolerance) {
        $payment->markAsPaid();
    } else {
        // Log mismatch for manual review
        \Log::warning('Amount mismatch', [
            'expected' => $expectedAmount,
            'received' => $receivedAmount
        ]);
    }
}
```

### Scenario 4: Webhook Received Multiple Times

```
Problem:
- Payment gateway sends webhook multiple times (for redundancy)
- System processes same payment twice
- Order fulfilled twice, inventory corrupted

Solution:
Idempotent webhook processing

Implementation:
```php
public function webhook(Request $request)
{
    $payload = $request->all();
    $payment = Payment::where('transaction_id', $payload['order_id'])->first();
    
    // Only process if status actually changed
    if ($payment->status === 'success') {
        // Already processed, just acknowledge
        return response()->json(['success' => true]);
    }
    
    // First time receiving this
    if ($payload['transaction_status'] === 'settlement') {
        // Use transaction to ensure atomic update
        DB::transaction(function () use ($payment) {
            $payment->markAsPaid();
            $payment->order->markAsProcessing();
            // Send email, etc.
        });
    }
    
    return response()->json(['success' => true]);
}
```

### Scenario 5: Gateway API Down / Network Error

```
Problem:
- Cannot reach payment gateway
- Cannot verify transactions
- Customer payments stuck

Solution:
Implement robust retry logic and status checking

Implementation:
```php
class PaymentProcessor
{
    public function verifyWithRetry($maxAttempts = 3)
    {
        $attempt = 0;
        $lastException = null;
        
        while ($attempt < $maxAttempts) {
            try {
                return $this->verify(); // Call actual verification
            } catch (\Exception $e) {
                $lastException = $e;
                $attempt++;
                
                if ($attempt < $maxAttempts) {
                    // Exponential backoff
                    $delay = 5 * pow(2, $attempt - 1); // 5s, 10s, 20s
                    sleep($delay);
                } else {
                    throw $lastException;
                }
            }
        }
    }
}

// Use in controller
public function verify($paymentId)
{
    $payment = Payment::find($paymentId);
    
    try {
        $processor = PaymentProcessorFactory::create($payment);
        $result = $processor->verifyWithRetry();
        
        if ($result['status'] === 'settlement') {
            $payment->markAsPaid();
        }
        
        return response()->json([
            'success' => true,
            'status' => $result['status']
        ]);
    } catch (\Exception $e) {
        \Log::error('Payment verification failed after retries', [
            'payment_id' => $paymentId,
            'error' => $e->getMessage()
        ]);
        
        // Notify admin to check manually
        \Notification::send(Admin::all(), new PaymentVerificationFailedNotification($payment));
        
        return response()->json([
            'success' => false,
            'message' => 'Cannot verify payment, please contact support'
        ], 500);
    }
}
```

---

## SECURITY CHECKLIST

### Before Going to Production

```
DATABASE SECURITY:
☐ All payment data encrypted at rest
☐ Separate database user with limited permissions
☐ Regular backups of payment data
☐ Audit logs for all payment data access
☐ GDPR compliance (can delete user payment data)

API SECURITY:
☐ HTTPS enforced (no HTTP)
☐ Rate limiting on payment endpoints
☐ CSRF token on all forms
☐ Signature verification on webhooks
☐ IP whitelist for webhook sources
☐ Request timeout configured (30 seconds max)

CODE SECURITY:
☐ No hardcoded keys in code
☐ All keys in .env (not committed)
☐ SQL injection prevention (use ORM)
☐ XSS prevention (sanitize output)
☐ CSRF protection enabled
☐ Authorization on all endpoints
☐ No sensitive data in error messages
☐ Logging configured (not too much, not too little)

PAYMENT GATEWAY:
☐ Use latest SDK version
☐ Server-to-server communication only
☐ Client key only for web, not API
☐ Test thoroughly on sandbox first
☐ Enable fraud detection
☐ Enable 3D Secure for cards
☐ Configure webhook IP whitelist
☐ Test webhook signature verification

MONITORING:
☐ Payment success/failure alerts
☐ Webhook delivery monitoring
☐ API latency monitoring
☐ Error rate monitoring
☐ Daily reconciliation with gateway
☐ Monthly compliance audit
```

---

## PERFORMANCE OPTIMIZATION

### 1. Database Queries
```php
// Bad: N+1 query problem
$payments = Payment::all();
foreach ($payments as $payment) {
    echo $payment->user->name; // Extra query per payment!
}

// Good: Eager loading
$payments = Payment::with('user', 'order')->get();
foreach ($payments as $payment) {
    echo $payment->user->name; // No extra query
}
```

### 2. Caching
```php
// Cache frequently accessed data
Route::get('/payment-methods', function () {
    return cache()->remember('payment-methods', 3600, function () {
        return [
            'banks' => config('payment.banks'),
            'ewallets' => config('payment.ewallets')
        ];
    });
});
```

### 3. Async Processing
```php
// Don't wait for email sending
$payment->markAsPaid();

// Send email in background job
Mail::send(...); // Remove this

// Queue it instead
Mail::queue('emails.payment-confirmed', ['payment' => $payment]);

// Process later via: php artisan queue:work
```

---

## MONITORING & ALERTS

### Setup Basic Monitoring
```php
// app/Http/Middleware/PaymentMonitor.php
class PaymentMonitor
{
    public function handle($request, $next)
    {
        $start = now();
        $response = $next($request);
        $duration = now()->diffInMilliseconds($start);
        
        // Log slow requests
        if ($duration > 5000) {
            \Log::warning('Slow payment request', [
                'url' => $request->url(),
                'duration_ms' => $duration
            ]);
        }
        
        return $response;
    }
}

// app/Jobs/DailyPaymentReconciliation.php
class DailyPaymentReconciliation implements ShouldQueue
{
    public function handle()
    {
        $pendingPayments = Payment::where('status', 'pending')
            ->where('created_at', '<', now()->subDays(7))
            ->get();
        
        foreach ($pendingPayments as $payment) {
            // Check status with gateway
            // Resolve discrepancies
            // Alert admin if needed
        }
    }
}

// Schedule in app/Console/Kernel.php
$schedule->job(new DailyPaymentReconciliation)->daily();
```

---

## TESTING PAYMENT SYSTEM

### Unit Tests
```php
<?php
// tests/Unit/PaymentProcessorTest.php

class PaymentProcessorTest extends TestCase
{
    public function test_bank_transfer_processor_creates_payment()
    {
        $payment = Payment::factory()->create([
            'payment_method' => 'bank_transfer'
        ]);
        
        $processor = new BankTransferProcessor($payment);
        $result = $processor->create();
        
        $this->assertTrue($result['success']);
        $this->assertNotNull($payment->account_number);
    }

    public function test_webhook_signature_verification()
    {
        $gateway = new MidtransGateway();
        
        $payload = [
            'order_id' => '123',
            'status_code' => '200',
            'gross_amount' => '150000',
            'signature_key' => '...'
        ];
        
        $this->assertTrue($gateway->verifySignature($payload));
    }
}
```

### Feature Tests
```php
<?php
// tests/Feature/PaymentFlowTest.php

class PaymentFlowTest extends TestCase
{
    public function test_complete_bank_transfer_flow()
    {
        $user = User::factory()->create();
        $order = Order::factory()->create(['user_id' => $user->id]);
        
        // 1. Initiate payment
        $response = $this->actingAs($user)->post('/payment/initialize', [
            'order_id' => $order->id,
            'payment_method' => 'bank_transfer',
            'bank_code' => 'bca'
        ]);
        
        $response->assertSuccessful();
        $paymentId = $response->json('payment_id');
        
        // 2. Simulate webhook
        $payment = Payment::find($paymentId);
        $this->post('/api/payment/webhook', [
            'order_id' => $payment->transaction_id,
            'transaction_status' => 'settlement',
            'gross_amount' => $order->total_price,
            'signature_key' => '...'
        ]);
        
        // 3. Verify payment marked as paid
        $payment->refresh();
        $this->assertEquals('success', $payment->status);
        $this->assertTrue($order->refresh()->isPaid());
    }
}
```

---

## TROUBLESHOOTING GUIDE

### Problem: Webhook tidak diterima

**Cek:**
1. Server key benar? → Verify di dashboard Midtrans
2. Webhook URL benar? → Check di dashboard, match dengan config
3. IP whitelist? → Add Midtrans IPs jika ada restrictions
4. SSL certificate? → Webhook must be HTTPS with valid cert
5. Server busy? → Check logs, rate limiting

**Solusi:**
```php
// Enable webhook logging
Route::post('/api/payment/webhook', function (Request $request) {
    \Log::info('Webhook received', $request->all());
    // ...
})->name('payment.webhook');

// Check logs
tail -f storage/logs/laravel.log | grep webhook
```

### Problem: Signature verification always fails

**Cek:**
1. Server key match? → Use env variable
2. Data not modified? → Hash exactly as gateway expects
3. Empty body? → Body must match exactly

**Solusi:**
```php
public function verifySignature($payload): bool
{
    $serverKey = config('payment.midtrans.server_key');
    
    // Exact order matters!
    $signature = hash('sha512',
        $payload['order_id'] .
        $payload['status_code'] .
        $payload['gross_amount'] .
        $serverKey
    );
    
    return $signature === $payload['signature_key'];
}
```

### Problem: Virtual Account number tidak muncul

**Cek:**
1. Response dari gateway valid?
2. Array structure correct? → `response['va_numbers'][0]`
3. Bank code valid? → Only 'bca', 'bni', 'bri'

**Solusi:**
```php
$response = $processor->create();
\Log::info('Gateway response', $response); // Debug response

// Print response structure
dd($response); // Check if va_numbers exists
```

---

## QUICK REFERENCE

### Environment Variables
```
PAYMENT_GATEWAY=midtrans
PAYMENT_ENV=sandbox
MIDTRANS_SERVER_KEY=SB-Mid-server-...
MIDTRANS_CLIENT_KEY=SB-Mid-client-...
MIDTRANS_MERCHANT_ID=M...
```

### Testing Credentials
```
Midtrans Sandbox:
- URL: https://app.sandbox.midtrans.com
- Email: your-registered-email
- Password: your-password
- No real money charged!

Test Cards:
- Visa: 4111 1111 1111 1111 (exp: any future, cvv: any)
- Mastercard: 5555 5555 5555 4444
```

### Useful APIs
```
Check Payment Status:
GET https://api.sandbox.midtrans.com/v2/{transaction_id}/status

Create Snap Token:
POST https://app.sandbox.midtrans.com/snap/v1/transactions

Get Bank Accounts:
GET https://api.sandbox.midtrans.com/v2/account/bank_accounts
```

---

Dokumentasi lengkap selesai! Siap untuk production! 🚀
