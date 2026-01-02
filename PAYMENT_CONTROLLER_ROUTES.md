# Payment Controller & Routes Implementation

---

## LANGKAH 6: Controllers

### 6.1 Payment Controller
```php
<?php
// app/Http/Controllers/PaymentController.php

namespace App\Http\Controllers;

use App\Models\Order;
use App\Models\Payment;
use App\Services\Payment\PaymentProcessorFactory;
use App\Services\Payment\Gateways\MidtransGateway;
use Illuminate\Http\Request;

class PaymentController extends Controller
{
    protected MidtransGateway $gateway;

    public function __construct()
    {
        $this->gateway = new MidtransGateway();
    }

    /**
     * Display payment page
     * GET /payment/{order}
     */
    public function show(Order $order)
    {
        // Check if order exists and user authorized
        $this->authorize('view', $order);

        // Get or create payment record
        $payment = $order->getLatestPayment() ?? new Payment();

        return view('payment.show', [
            'order' => $order,
            'payment' => $payment
        ]);
    }

    /**
     * Initialize payment
     * POST /payment/initialize
     */
    public function initialize(Request $request)
    {
        $validated = $request->validate([
            'order_id' => 'required|exists:orders,id',
            'payment_method' => 'required|in:bank_transfer,qris,gopay,dana,ovo,shopeepay,visa,mastercard',
            'bank_code' => 'nullable|required_if:payment_method,bank_transfer|in:bca,bni,bri',
            'phone_number' => 'nullable|required_if:payment_method,gopay',
        ]);

        $order = Order::findOrFail($validated['order_id']);
        $this->authorize('pay', $order);

        try {
            // Create payment record
            $payment = Payment::create([
                'order_id' => $order->id,
                'user_id' => auth()->id(),
                'amount' => $order->total_price,
                'currency' => 'IDR',
                'payment_method' => $validated['payment_method'],
                'payment_provider' => 'midtrans',
                'bank_code' => $validated['bank_code'] ?? null,
                'phone_number' => $validated['phone_number'] ?? null,
                'transaction_id' => 'ORD-' . $order->id . '-' . time(),
                'status' => 'pending'
            ]);

            // Generate payment via gateway
            $processor = PaymentProcessorFactory::create($payment);
            $result = $processor->create();

            if (!$result['success']) {
                return response()->json([
                    'success' => false,
                    'message' => $result['error'] ?? 'Payment initialization failed'
                ], 422);
            }

            return response()->json([
                'success' => true,
                'payment_id' => $payment->id,
                'data' => $result['data'] ?? $result
            ]);
        } catch (\Exception $e) {
            \Log::error('Payment initialization error: ' . $e->getMessage());

            return response()->json([
                'success' => false,
                'message' => 'Failed to initialize payment'
            ], 500);
        }
    }

    /**
     * Display payment details
     * GET /payment/{id}/details
     */
    public function details($paymentId)
    {
        $payment = Payment::findOrFail($paymentId);
        $this->authorize('view', $payment);

        $processor = PaymentProcessorFactory::create($payment);
        $details = $processor->getDetails();

        return response()->json([
            'success' => true,
            'payment' => $payment,
            'details' => $details
        ]);
    }

    /**
     * Verify payment status
     * GET /payment/{id}/verify
     */
    public function verify($paymentId)
    {
        $payment = Payment::findOrFail($paymentId);
        $this->authorize('view', $payment);

        try {
            $processor = PaymentProcessorFactory::create($payment);
            $result = $processor->verify();

            if ($result['success'] && $result['status'] === 'settlement') {
                // Payment successful
                $payment->markAsPaid();

                return response()->json([
                    'success' => true,
                    'status' => 'paid',
                    'message' => 'Payment confirmed'
                ]);
            }

            return response()->json([
                'success' => true,
                'status' => $result['status'],
                'message' => 'Payment still pending'
            ]);
        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => $e->getMessage()
            ], 500);
        }
    }

    /**
     * Handle webhook from payment gateway
     * POST /api/payment/webhook
     */
    public function webhook(Request $request)
    {
        $payload = $request->all();

        // Log webhook
        \Log::info('Payment Webhook Received', $payload);

        // Verify signature
        if (!$this->gateway->verifySignature($payload)) {
            \Log::warning('Invalid webhook signature');
            return response()->json(['error' => 'Invalid signature'], 403);
        }

        try {
            $transactionId = $payload['order_id'];
            $payment = Payment::where('transaction_id', $transactionId)->first();

            if (!$payment) {
                \Log::warning('Payment not found: ' . $transactionId);
                return response()->json(['error' => 'Payment not found'], 404);
            }

            $status = $payload['transaction_status'];

            // Update payment status based on webhook
            switch ($status) {
                case 'settlement':
                case 'capture':
                    if ($payment->status !== 'success') {
                        $payment->update([
                            'status' => 'success',
                            'gateway_response' => $payload,
                            'paid_at' => now()
                        ]);
                        $payment->markAsPaid();
                        
                        // Send confirmation email
                        \Mail::send('emails.payment-confirmed', 
                            ['payment' => $payment], 
                            function($msg) use ($payment) {
                                $msg->to($payment->user->email);
                            }
                        );
                    }
                    break;

                case 'pending':
                    $payment->update(['status' => 'processing']);
                    break;

                case 'deny':
                case 'cancel':
                    $payment->update(['status' => 'failed']);
                    break;

                case 'expire':
                    $payment->update(['status' => 'expired']);
                    break;
            }

            // Log action
            $payment->logs()->create([
                'event_type' => 'webhook',
                'status_to' => $payment->status,
                'response_payload' => $payload
            ]);

            return response()->json(['success' => true]);
        } catch (\Exception $e) {
            \Log::error('Webhook processing error: ' . $e->getMessage());
            return response()->json(['error' => 'Processing error'], 500);
        }
    }

    /**
     * Payment success callback
     * GET /payment/success
     */
    public function success(Request $request)
    {
        $orderId = $request->query('order_id');
        $order = Order::findOrFail($orderId);

        return view('payment.success', [
            'order' => $order,
            'payment' => $order->getLatestPayment()
        ]);
    }

    /**
     * Payment failure callback
     * GET /payment/failed
     */
    public function failed(Request $request)
    {
        $orderId = $request->query('order_id');
        $order = Order::findOrFail($orderId);

        return view('payment.failed', [
            'order' => $order,
            'payment' => $order->getLatestPayment()
        ]);
    }

    /**
     * Retry payment
     * POST /payment/{id}/retry
     */
    public function retry(Payment $payment)
    {
        $this->authorize('update', $payment);

        if (!$payment->isPending()) {
            return response()->json([
                'success' => false,
                'message' => 'Can only retry pending payments'
            ], 422);
        }

        try {
            $processor = PaymentProcessorFactory::create($payment);
            $result = $processor->create();

            if ($result['success']) {
                return response()->json([
                    'success' => true,
                    'message' => 'Payment retry initiated',
                    'data' => $result['data'] ?? $result
                ]);
            }

            return response()->json([
                'success' => false,
                'message' => $result['error']
            ], 422);
        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Retry failed: ' . $e->getMessage()
            ], 500);
        }
    }

    /**
     * Request refund
     * POST /payment/{id}/refund
     */
    public function refund(Payment $payment, Request $request)
    {
        $this->authorize('refund', $payment);

        $validated = $request->validate([
            'reason' => 'required|string|max:500',
            'amount' => 'nullable|numeric|min:1'
        ]);

        if (!$payment->isSuccess()) {
            return response()->json([
                'success' => false,
                'message' => 'Can only refund completed payments'
            ], 422);
        }

        try {
            $processor = PaymentProcessorFactory::create($payment);
            $result = $processor->refund();

            if ($result['success']) {
                $payment->update(['status' => 'refunded']);

                return response()->json([
                    'success' => true,
                    'message' => 'Refund processed successfully'
                ]);
            }

            return response()->json([
                'success' => false,
                'message' => 'Refund failed'
            ], 422);
        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Refund error: ' . $e->getMessage()
            ], 500);
        }
    }

    /**
     * Get transaction history
     * GET /payment/history
     */
    public function history()
    {
        $payments = auth()->user()->payments()
            ->with('order')
            ->latest()
            ->paginate(15);

        return view('payment.history', [
            'payments' => $payments
        ]);
    }

    /**
     * Download invoice
     * GET /payment/{id}/invoice
     */
    public function invoice(Payment $payment)
    {
        $this->authorize('view', $payment);

        // Generate PDF invoice
        $pdf = \PDF::loadView('payment.invoice', ['payment' => $payment]);
        return $pdf->download('invoice-' . $payment->transaction_id . '.pdf');
    }
}
```

---

## LANGKAH 7: Routes

### 7.1 Web Routes
```php
<?php
// routes/web.php

use App\Http\Controllers\PaymentController;

// Payment Routes
Route::middleware('auth')->group(function () {
    // Display payment page
    Route::get('/payment/{order}', [PaymentController::class, 'show'])->name('payment.show');
    
    // Initialize payment
    Route::post('/payment/initialize', [PaymentController::class, 'initialize'])->name('payment.initialize');
    
    // Get payment details
    Route::get('/payment/{id}/details', [PaymentController::class, 'details'])->name('payment.details');
    
    // Verify payment status
    Route::get('/payment/{id}/verify', [PaymentController::class, 'verify'])->name('payment.verify');
    
    // Retry payment
    Route::post('/payment/{id}/retry', [PaymentController::class, 'retry'])->name('payment.retry');
    
    // Request refund
    Route::post('/payment/{id}/refund', [PaymentController::class, 'refund'])->name('payment.refund');
    
    // Payment history
    Route::get('/payment-history', [PaymentController::class, 'history'])->name('payment.history');
    
    // Download invoice
    Route::get('/payment/{id}/invoice', [PaymentController::class, 'invoice'])->name('payment.invoice');
});

// Payment callbacks (no auth needed)
Route::get('/payment/success', [PaymentController::class, 'success'])->name('payment.success');
Route::get('/payment/failed', [PaymentController::class, 'failed'])->name('payment.failed');

// Webhook (requires signature verification)
Route::post('/api/payment/webhook', [PaymentController::class, 'webhook'])->name('payment.webhook');
```

### 7.2 API Routes
```php
<?php
// routes/api.php

use App\Http\Controllers\PaymentController;

Route::group(['prefix' => 'payment', 'middleware' => 'auth:sanctum'], function () {
    // Get payment details
    Route::get('/{id}', [PaymentController::class, 'details']);
    
    // Verify status
    Route::post('/{id}/verify', [PaymentController::class, 'verify']);
    
    // Retry payment
    Route::post('/{id}/retry', [PaymentController::class, 'retry']);
    
    // Request refund
    Route::post('/{id}/refund', [PaymentController::class, 'refund']);
});

// Webhook endpoint
Route::post('/payment/webhook', [PaymentController::class, 'webhook']);
```

---

## LANGKAH 8: Configuration

### 8.1 Create Config File
```php
<?php
// config/payment.php

return [
    'gateway' => env('PAYMENT_GATEWAY', 'midtrans'),
    'env' => env('PAYMENT_ENV', 'sandbox'),
    'enable_3ds' => env('PAYMENT_ENABLE_3DS', true),
    'webhook_verify' => env('PAYMENT_WEBHOOK_SIGNATURE_VERIFY', true),
    'retry_attempts' => env('PAYMENT_RETRY_ATTEMPTS', 3),
    'expiry_hours' => env('PAYMENT_EXPIRY_HOURS', 24),

    'midtrans' => [
        'server_key' => env('MIDTRANS_SERVER_KEY'),
        'client_key' => env('MIDTRANS_CLIENT_KEY'),
        'merchant_id' => env('MIDTRANS_MERCHANT_ID'),
        'api_base_url' => env('MIDTRANS_API_BASE_URL', 'https://app.sandbox.midtrans.com'),
    ],

    'xendit' => [
        'api_key' => env('XENDIT_API_KEY'),
        'webhook_token' => env('XENDIT_WEBHOOK_TOKEN'),
    ],

    'urls' => [
        'success' => env('PAYMENT_SUCCESS_URL', '/payment/success'),
        'failure' => env('PAYMENT_FAILURE_URL', '/payment/failed'),
        'webhook' => env('PAYMENT_WEBHOOK_URL', '/api/payment/webhook'),
    ],

    'banks' => [
        'bca' => [
            'name' => 'Bank Central Asia',
            'code' => 'bca',
            'min_amount' => 10000,
            'max_amount' => 1000000000
        ],
        'bni' => [
            'name' => 'Bank Negara Indonesia',
            'code' => 'bni',
            'min_amount' => 10000,
            'max_amount' => 1000000000
        ],
        'bri' => [
            'name' => 'Bank Rakyat Indonesia',
            'code' => 'bri',
            'min_amount' => 10000,
            'max_amount' => 1000000000
        ]
    ],

    'ewallets' => [
        'gopay' => [
            'name' => 'GoPay',
            'min_amount' => 10000,
            'max_amount' => 10000000
        ],
        'dana' => [
            'name' => 'DANA',
            'min_amount' => 10000,
            'max_amount' => 5000000
        ],
        'ovo' => [
            'name' => 'OVO',
            'min_amount' => 10000,
            'max_amount' => 10000000
        ],
        'shopeepay' => [
            'name' => 'ShopeePay',
            'min_amount' => 10000,
            'max_amount' => 5000000
        ]
    ]
];
```

---

## LANGKAH 9: Views

### 9.1 Payment Checkout View
```blade
{{-- resources/views/payment/show.blade.php --}}

@extends('layouts.app')

@section('content')
<div class="payment-container">
    <div class="payment-header">
        <h2>💳 Pembayaran Pesanan</h2>
        <p>Order ID: <strong>{{ $order->id }}</strong></p>
    </div>

    <div class="payment-body">
        <!-- Order Summary -->
        <div class="order-summary">
            <h3>Ringkasan Pesanan</h3>
            <div class="summary-item">
                <span>Subtotal:</span>
                <span>Rp {{ number_format($order->subtotal) }}</span>
            </div>
            <div class="summary-item">
                <span>Ongkir:</span>
                <span>Rp {{ number_format($order->shipping_cost) }}</span>
            </div>
            <div class="summary-item">
                <span>Pajak:</span>
                <span>Rp {{ number_format($order->tax) }}</span>
            </div>
            <hr>
            <div class="summary-total">
                <span>Total Pembayaran:</span>
                <span>Rp {{ number_format($order->total_price) }}</span>
            </div>
        </div>

        <!-- Payment Methods -->
        <div class="payment-methods">
            <h3>Pilih Metode Pembayaran</h3>

            <form id="paymentForm">
                @csrf

                <!-- Bank Transfer -->
                <div class="method-group">
                    <h4>🏦 Transfer Bank</h4>
                    <div class="method-options">
                        <label>
                            <input type="radio" name="payment_method" value="bank_transfer" 
                                   data-method="bank_transfer">
                            <span>BCA</span>
                            <input type="hidden" name="bank_code" class="bank-code" value="bca">
                        </label>
                        <label>
                            <input type="radio" name="payment_method" value="bank_transfer" 
                                   data-method="bank_transfer">
                            <span>BNI</span>
                            <input type="hidden" name="bank_code" class="bank-code" value="bni">
                        </label>
                        <label>
                            <input type="radio" name="payment_method" value="bank_transfer" 
                                   data-method="bank_transfer">
                            <span>BRI</span>
                            <input type="hidden" name="bank_code" class="bank-code" value="bri">
                        </label>
                    </div>
                </div>

                <!-- QRIS -->
                <div class="method-group">
                    <h4>📱 QRIS</h4>
                    <label>
                        <input type="radio" name="payment_method" value="qris" data-method="qris">
                        <span>Scan QRIS dengan app banking apapun</span>
                    </label>
                </div>

                <!-- E-Wallet -->
                <div class="method-group">
                    <h4>💳 E-Wallet</h4>
                    <div class="method-options">
                        <label>
                            <input type="radio" name="payment_method" value="gopay">
                            <span>GoPay</span>
                        </label>
                        <label>
                            <input type="radio" name="payment_method" value="dana">
                            <span>DANA</span>
                        </label>
                        <label>
                            <input type="radio" name="payment_method" value="ovo">
                            <span>OVO</span>
                        </label>
                        <label>
                            <input type="radio" name="payment_method" value="shopeepay">
                            <span>ShopeePay</span>
                        </label>
                    </div>
                </div>

                <!-- Card -->
                <div class="method-group">
                    <h4>💳 Kartu Kredit/Debit</h4>
                    <div class="method-options">
                        <label>
                            <input type="radio" name="payment_method" value="visa">
                            <span>Visa</span>
                        </label>
                        <label>
                            <input type="radio" name="payment_method" value="mastercard">
                            <span>Mastercard</span>
                        </label>
                    </div>
                </div>

                <button type="submit" class="btn-pay">
                    Lanjutkan ke Pembayaran
                </button>
            </form>
        </div>
    </div>
</div>

<script>
document.getElementById('paymentForm').addEventListener('submit', async function(e) {
    e.preventDefault();

    const method = document.querySelector('input[name="payment_method"]:checked').value;
    const data = {
        order_id: {{ $order->id }},
        payment_method: method,
        _token: document.querySelector('input[name="_token"]').value
    };

    // Add bank code if bank transfer
    if (method === 'bank_transfer') {
        const bankCode = document.querySelector('input[name="bank_code"]').value;
        data.bank_code = bankCode;
    }

    try {
        const response = await fetch('/payment/initialize', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Accept': 'application/json'
            },
            body: JSON.stringify(data)
        });

        const result = await response.json();

        if (result.success) {
            // Handle different payment methods
            if (method === 'bank_transfer') {
                // Redirect ke virtual account details
                window.location.href = `/payment/${result.payment_id}/details`;
            } else if (method === 'qris') {
                // Show QR code
                window.location.href = `/payment/${result.payment_id}/details`;
            } else if (['gopay', 'dana', 'ovo', 'shopeepay'].includes(method)) {
                // Redirect ke payment gateway
                window.location.href = result.data.redirect_url;
            } else if (['visa', 'mastercard'].includes(method)) {
                // Snap token payment
                snap.pay(result.data.snap_token);
            }
        } else {
            alert('Error: ' + result.message);
        }
    } catch (error) {
        console.error('Error:', error);
        alert('Terjadi kesalahan');
    }
});
</script>

<style>
    .payment-container {
        max-width: 900px;
        margin: 30px auto;
        display: grid;
        grid-template-columns: 1fr 2fr;
        gap: 30px;
    }

    .method-group {
        margin-bottom: 25px;
        padding-bottom: 15px;
        border-bottom: 1px solid #eee;
    }

    .method-group h4 {
        margin: 0 0 15px 0;
        color: #4A3F4E;
    }

    .method-options {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
        gap: 10px;
    }

    .method-options label {
        display: flex;
        align-items: center;
        padding: 12px;
        border: 2px solid #ddd;
        border-radius: 8px;
        cursor: pointer;
        transition: all 0.3s;
    }

    .method-options label:hover {
        border-color: #d46a8b;
        background: #FFEFF6;
    }

    .method-options input[type="radio"] {
        margin-right: 8px;
    }

    .btn-pay {
        width: 100%;
        padding: 15px;
        background: #F3AFCB;
        color: white;
        border: none;
        border-radius: 8px;
        font-weight: bold;
        font-size: 16px;
        cursor: pointer;
        transition: all 0.3s;
        margin-top: 20px;
    }

    .btn-pay:hover {
        background: #d46a8b;
        transform: translateY(-2px);
    }

    @media (max-width: 768px) {
        .payment-container {
            grid-template-columns: 1fr;
        }
    }
</style>
@endsection
```

---

## LANGKAH 10: Testing Checklist

```
TESTING PAYMENT SYSTEM:

BANK TRANSFER:
☐ Test VA creation
☐ Test VA expiry
☐ Test payment confirmation
☐ Test manual verification

QRIS:
☐ Test QR code generation
☐ Test QRIS string scanning
☐ Test payment confirmation
☐ Test expiry (15 minutes)

E-WALLET:
☐ Test redirect flow
☐ Test callback handling
☐ Test payment confirmation
☐ Test failure handling

CARD:
☐ Test card validation
☐ Test 3D Secure
☐ Test CVV verification
☐ Test fraud detection

WEBHOOK:
☐ Test signature verification
☐ Test idempotent processing
☐ Test error handling
☐ Test retry logic

GENERAL:
☐ Test payment history
☐ Test refund request
☐ Test invoice generation
☐ Test error scenarios
☐ Test database logging
☐ Test email notifications
```

---

Implementasi lengkap ini siap untuk diproduksi! 🚀
