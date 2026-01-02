# Payment System Implementation Guide
## Panduan Step-by-Step Implementasi di Laravel

---

## LANGKAH 1: Setup Environment & Dependencies

### 1.1 Update .env file
```env
# app/config
APP_NAME="Aybi Supermarket"
APP_ENV=sandbox
APP_DEBUG=true
APP_URL=http://localhost

# Payment Gateway Configuration
PAYMENT_GATEWAY=midtrans
PAYMENT_ENV=sandbox

# Midtrans Keys (dari https://dashboard.midtrans.com)
MIDTRANS_MERCHANT_ID=M12345
MIDTRANS_SERVER_KEY=SB-Mid-server-xxxxxxxxxxxx
MIDTRANS_CLIENT_KEY=SB-Mid-client-xxxxxxxxxxxx
MIDTRANS_API_BASE_URL=https://app.sandbox.midtrans.com

# Xendit Keys (Alternative)
XENDIT_API_KEY=xnd_development_xxxxxxxxxxxx
XENDIT_WEBHOOK_TOKEN=xnd_whsec_xxxxxxxxxxxx

# Payment Webhook
PAYMENT_WEBHOOK_URL=${APP_URL}/api/payment/webhook
PAYMENT_SUCCESS_URL=${APP_URL}/payment/success
PAYMENT_FAILURE_URL=${APP_URL}/payment/failed

# Security
PAYMENT_WEBHOOK_SIGNATURE_VERIFY=true
PAYMENT_ENABLE_3DS=true
PAYMENT_RETRY_ATTEMPTS=3
```

### 1.2 Install Required Packages
```bash
composer require midtrans/midtrans-php
composer require guzzlehttp/guzzle
composer require spatie/laravel-permission  # Untuk authorization
composer require laravel/sanctum  # Untuk API authentication
```

---

## LANGKAH 2: Database Setup

### 2.1 Create Migrations
```bash
php artisan make:migration create_payments_table
php artisan make:migration create_payment_logs_table
php artisan make:migration create_payment_bank_accounts_table
```

### 2.2 payments table migration
```php
// database/migrations/xxxx_create_payments_table.php

Schema::create('payments', function (Blueprint $table) {
    $table->id();
    $table->string('transaction_id')->unique();
    $table->foreignId('order_id')->constrained();
    $table->foreignId('user_id')->constrained();
    
    // Payment Details
    $table->bigInteger('amount'); // in rupiah
    $table->string('currency')->default('IDR');
    $table->string('payment_method'); // bank_transfer, qris, gopay, etc
    $table->string('payment_provider')->default('midtrans');
    
    // Bank Transfer Specific
    $table->string('bank_code')->nullable();
    $table->string('bank_name')->nullable();
    $table->string('account_number')->nullable();
    $table->string('account_holder')->nullable();
    
    // QRIS Specific
    $table->longText('qris_string')->nullable();
    $table->string('merchant_id')->nullable();
    
    // E-Wallet Specific
    $table->string('wallet_type')->nullable();
    $table->string('phone_number')->nullable();
    
    // Card Specific
    $table->string('card_token')->nullable();
    $table->string('card_last_four')->nullable();
    $table->string('card_brand')->nullable();
    
    // Status & Tracking
    $table->enum('status', ['pending', 'processing', 'success', 'failed', 'expired', 'cancelled'])->default('pending');
    $table->json('gateway_response')->nullable();
    
    // Reference Numbers
    $table->string('reference_id')->nullable();
    $table->string('merchant_reference')->nullable();
    
    // Timing
    $table->timestamp('paid_at')->nullable();
    $table->timestamp('expired_at')->nullable();
    
    // Additional
    $table->text('description')->nullable();
    $table->json('metadata')->nullable();
    
    $table->timestamps();
    
    // Indexes
    $table->index('transaction_id');
    $table->index('order_id');
    $table->index('user_id');
    $table->index('status');
    $table->index('created_at');
});
```

### 2.3 Run Migrations
```bash
php artisan migrate
```

---

## LANGKAH 3: Create Models & Relationships

### 3.1 Payment Model
```php
<?php
// app/Models/Payment.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Payment extends Model
{
    protected $fillable = [
        'transaction_id',
        'order_id',
        'user_id',
        'amount',
        'currency',
        'payment_method',
        'payment_provider',
        'bank_code',
        'bank_name',
        'account_number',
        'account_holder',
        'qris_string',
        'merchant_id',
        'wallet_type',
        'phone_number',
        'card_token',
        'card_last_four',
        'card_brand',
        'status',
        'gateway_response',
        'reference_id',
        'merchant_reference',
        'paid_at',
        'expired_at',
        'description',
        'metadata'
    ];

    protected $casts = [
        'gateway_response' => 'array',
        'metadata' => 'array',
        'paid_at' => 'datetime',
        'expired_at' => 'datetime',
    ];

    // Relationships
    public function order(): BelongsTo
    {
        return $this->belongsTo(Order::class);
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function logs(): HasMany
    {
        return $this->hasMany(PaymentLog::class);
    }

    // Scopes
    public function scopePending($query)
    {
        return $query->where('status', 'pending');
    }

    public function scopeSuccess($query)
    {
        return $query->where('status', 'success');
    }

    public function scopeFailed($query)
    {
        return $query->where('status', 'failed');
    }

    // Methods
    public function markAsPaid()
    {
        $this->update([
            'status' => 'success',
            'paid_at' => now()
        ]);
        $this->order->markAsPaid();
    }

    public function markAsFailed($message = null)
    {
        $this->update([
            'status' => 'failed',
            'description' => $message
        ]);
    }

    public function isPending(): bool
    {
        return $this->status === 'pending';
    }

    public function isSuccess(): bool
    {
        return $this->status === 'success';
    }

    public function generateReference(): string
    {
        return 'ORDER-' . $this->order_id . '-' . $this->id;
    }
}
```

### 3.2 Order Model (Update)
```php
<?php
// app/Models/Order.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Order extends Model
{
    // ... existing code ...

    public function payments(): HasMany
    {
        return $this->hasMany(Payment::class);
    }

    public function getLatestPayment()
    {
        return $this->payments()->latest()->first();
    }

    public function markAsPaid()
    {
        $this->update(['status' => 'paid']);
        // Trigger fulfillment, send email, etc
    }

    public function isPaid(): bool
    {
        return $this->status === 'paid';
    }
}
```

---

## LANGKAH 4: Create Payment Service Classes

### 4.1 Abstract Payment Processor
```php
<?php
// app/Services/Payment/PaymentProcessor.php

namespace App\Services\Payment;

use App\Models\Payment;

abstract class PaymentProcessor
{
    protected Payment $payment;
    
    public function __construct(Payment $payment)
    {
        $this->payment = $payment;
    }

    /**
     * Create payment in gateway
     */
    abstract public function create(): array;

    /**
     * Verify payment status
     */
    abstract public function verify(): array;

    /**
     * Cancel/Refund payment
     */
    abstract public function refund(): array;

    /**
     * Get payment details
     */
    abstract public function getDetails(): array;

    /**
     * Log payment action
     */
    protected function logAction($type, $statusFrom, $statusTo, $request, $response, $error = null)
    {
        \App\Models\PaymentLog::create([
            'payment_id' => $this->payment->id,
            'event_type' => $type,
            'status_from' => $statusFrom,
            'status_to' => $statusTo,
            'request_payload' => $request,
            'response_payload' => $response,
            'error_message' => $error,
            'ip_address' => request()->ip(),
            'user_agent' => request()->userAgent(),
        ]);
    }
}
```

### 4.2 Bank Transfer Processor
```php
<?php
// app/Services/Payment/BankTransferProcessor.php

namespace App\Services\Payment;

class BankTransferProcessor extends PaymentProcessor
{
    public function create(): array
    {
        $bankCode = $this->payment->bank_code; // 'bca', 'bni', 'bri'
        
        $request = [
            'payment_type' => 'bank_transfer',
            'bank_transfer' => [
                'bank' => $bankCode,
                'required' => true
            ],
            'transaction_details' => [
                'order_id' => $this->payment->transaction_id,
                'gross_amount' => $this->payment->amount
            ],
            'customer_details' => [
                'first_name' => $this->payment->user->name,
                'email' => $this->payment->user->email,
                'phone' => $this->payment->user->phone ?? ''
            ],
            'expiry' => [
                'start_time' => now()->toDateTimeString(),
                'duration' => 86400, // 24 hours
                'unit' => 'seconds'
            ]
        ];

        try {
            $response = $this->callGatewayAPI('charge', $request);
            
            // Store virtual account details
            if (isset($response['va_numbers'][0])) {
                $vaData = $response['va_numbers'][0];
                $this->payment->update([
                    'account_number' => $vaData['va_number'],
                    'bank_code' => $vaData['bank'],
                    'gateway_response' => $response
                ]);
            }

            $this->logAction('create_va', 'pending', 'pending', $request, $response);

            return [
                'success' => true,
                'data' => $response
            ];
        } catch (\Exception $e) {
            $this->logAction('create_va', 'pending', 'failed', $request, [], $e->getMessage());
            
            return [
                'success' => false,
                'error' => $e->getMessage()
            ];
        }
    }

    public function verify(): array
    {
        // Panggil gateway API untuk check status
        $response = $this->callGatewayAPI('status', [
            'transaction_id' => $this->payment->transaction_id
        ]);

        return [
            'success' => true,
            'status' => $response['transaction_status']
        ];
    }

    public function refund(): array
    {
        // Implement refund logic
        return [
            'success' => true,
            'message' => 'Refund processed'
        ];
    }

    public function getDetails(): array
    {
        return [
            'account_number' => $this->payment->account_number,
            'bank_name' => $this->payment->bank_name,
            'amount' => $this->payment->amount,
            'merchant' => config('payment.merchant_name')
        ];
    }

    private function callGatewayAPI($endpoint, $data): array
    {
        // Implementation will use Midtrans SDK
        // See next section
    }
}
```

### 4.3 QRIS Processor
```php
<?php
// app/Services/Payment/QRISProcessor.php

namespace App\Services\Payment;

class QRISProcessor extends PaymentProcessor
{
    public function create(): array
    {
        $request = [
            'payment_type' => 'qris',
            'qris' => [
                'acquirer' => 'gopay' // or blank for auto
            ],
            'transaction_details' => [
                'order_id' => $this->payment->transaction_id,
                'gross_amount' => $this->payment->amount
            ],
            'customer_details' => [
                'first_name' => $this->payment->user->name,
                'email' => $this->payment->user->email,
            ],
            'expiry' => [
                'start_time' => now()->toDateTimeString(),
                'duration' => 900, // 15 minutes
                'unit' => 'seconds'
            ]
        ];

        try {
            $response = $this->callGatewayAPI('charge', $request);

            $this->payment->update([
                'qris_string' => $response['qr_string'] ?? null,
                'gateway_response' => $response
            ]);

            return [
                'success' => true,
                'qr_code' => $response['qr_string'],
                'qr_image_url' => $this->generateQRImage($response['qr_string'])
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage()
            ];
        }
    }

    public function verify(): array
    {
        $response = $this->callGatewayAPI('status', [
            'transaction_id' => $this->payment->transaction_id
        ]);

        return [
            'success' => true,
            'status' => $response['transaction_status']
        ];
    }

    public function refund(): array
    {
        return ['success' => true];
    }

    public function getDetails(): array
    {
        return [
            'qr_string' => $this->payment->qris_string,
            'amount' => $this->payment->amount,
            'expires_at' => $this->payment->expired_at
        ];
    }

    private function generateQRImage(string $qrString): string
    {
        // Use QR code library
        // https://github.com/chillerlan/php-qrcode
        $qrCode = new \chillerlan\QRCode\QRCode($qrString);
        return base64_encode($qrCode->render());
    }

    private function callGatewayAPI($endpoint, $data): array
    {
        // Implementation
    }
}
```

### 4.4 E-Wallet Processor (GoPay, DANA, OVO, ShopeePay)
```php
<?php
// app/Services/Payment/EWalletProcessor.php

namespace App\Services\Payment;

class EWalletProcessor extends PaymentProcessor
{
    public function create(): array
    {
        $walletType = $this->payment->wallet_type; // 'gopay', 'dana', 'ovo', 'shopeepay'

        $request = [
            'payment_type' => 'digital_wallet',
            'digital_wallet' => [
                'type' => $walletType,
                'phone_number' => $this->payment->phone_number ?? ''
            ],
            'transaction_details' => [
                'order_id' => $this->payment->transaction_id,
                'gross_amount' => $this->payment->amount
            ],
            'customer_details' => [
                'first_name' => $this->payment->user->name,
                'email' => $this->payment->user->email,
                'phone' => $this->payment->phone_number ?? $this->payment->user->phone
            ],
            'callbacks' => [
                'finish' => config('app.url') . '/payment/callback',
                'error' => config('app.url') . '/payment/error',
                'notification' => config('app.url') . '/api/payment/webhook'
            ]
        ];

        try {
            $response = $this->callGatewayAPI('charge', $request);

            $this->payment->update([
                'reference_id' => $response['redirect_url'] ?? null,
                'gateway_response' => $response
            ]);

            return [
                'success' => true,
                'redirect_url' => $response['redirect_url'],
                'payment_url' => $response['actions']['redirect_url'] ?? null
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage()
            ];
        }
    }

    public function verify(): array
    {
        $response = $this->callGatewayAPI('status', [
            'transaction_id' => $this->payment->transaction_id
        ]);

        return [
            'success' => true,
            'status' => $response['transaction_status'],
            'amount_received' => $response['gross_amount'] ?? null
        ];
    }

    public function refund(): array
    {
        // E-wallet refund logic
        return ['success' => true];
    }

    public function getDetails(): array
    {
        return [
            'wallet_type' => $this->payment->wallet_type,
            'payment_url' => $this->payment->reference_id,
            'amount' => $this->payment->amount
        ];
    }

    private function callGatewayAPI($endpoint, $data): array
    {
        // Implementation
    }
}
```

### 4.5 Card Payment Processor
```php
<?php
// app/Services/Payment/CardProcessor.php

namespace App\Services\Payment;

class CardProcessor extends PaymentProcessor
{
    public function create(): array
    {
        $request = [
            'payment_type' => 'credit_card',
            'credit_card' => [
                'secure' => true // 3D Secure enabled
            ],
            'transaction_details' => [
                'order_id' => $this->payment->transaction_id,
                'gross_amount' => $this->payment->amount
            ],
            'customer_details' => [
                'first_name' => $this->payment->user->name,
                'email' => $this->payment->user->email,
            ],
            'callbacks' => [
                'finish' => config('app.url') . '/payment/callback'
            ]
        ];

        try {
            $response = $this->callGatewayAPI('charge', $request);

            $this->payment->update([
                'card_token' => $response['payment_method']['token_id'] ?? null,
                'gateway_response' => $response
            ]);

            return [
                'success' => true,
                'token' => $response['payment_method']['token_id'] ?? null,
                'redirect_url' => $response['redirect_url'] ?? null
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage()
            ];
        }
    }

    public function verify(): array
    {
        $response = $this->callGatewayAPI('status', [
            'transaction_id' => $this->payment->transaction_id
        ]);

        return [
            'success' => true,
            'status' => $response['transaction_status'],
            'fraud_status' => $response['fraud_status'] ?? 'accept'
        ];
    }

    public function refund(): array
    {
        // Card refund
        return ['success' => true];
    }

    public function getDetails(): array
    {
        return [
            'card_last_four' => $this->payment->card_last_four,
            'card_brand' => $this->payment->card_brand,
            'amount' => $this->payment->amount
        ];
    }

    private function callGatewayAPI($endpoint, $data): array
    {
        // Implementation
    }
}
```

---

## LANGKAH 5: Gateway Integration Service

### 5.1 Midtrans Service
```php
<?php
// app/Services/Payment/Gateways/MidtransGateway.php

namespace App\Services\Payment\Gateways;

use Midtrans\Config;
use Midtrans\Snap;
use Midtrans\Transaction;

class MidtransGateway
{
    public function __construct()
    {
        // Configure Midtrans
        Config::$serverKey = config('payment.midtrans.server_key');
        Config::$clientKey = config('payment.midtrans.client_key');
        Config::$isProduction = config('payment.env') === 'production';
        Config::$isSanitized = true;
        Config::$is3ds = config('payment.enable_3ds', true);
    }

    /**
     * Create payment snap redirect
     */
    public function createSnapToken(array $params): string
    {
        try {
            $snapToken = Snap::getSnapToken($params);
            return $snapToken;
        } catch (\Exception $e) {
            \Log::error('Midtrans Snap Token Error: ' . $e->getMessage());
            throw $e;
        }
    }

    /**
     * Direct charge (bank transfer, QRIS, etc)
     */
    public function charge(array $params): array
    {
        try {
            $response = Snap::transactions()->charge($params);
            return $response;
        } catch (\Exception $e) {
            \Log::error('Midtrans Charge Error: ' . $e->getMessage());
            throw $e;
        }
    }

    /**
     * Get transaction status
     */
    public function getStatus(string $transactionId): array
    {
        try {
            $status = Transaction::status($transactionId);
            return (array) $status;
        } catch (\Exception $e) {
            \Log::error('Midtrans Status Error: ' . $e->getMessage());
            throw $e;
        }
    }

    /**
     * Verify webhook signature
     */
    public function verifySignature($request): bool
    {
        $serverKey = config('payment.midtrans.server_key');
        $orderId = $request['order_id'];
        $statusCode = $request['status_code'];
        $grossAmount = $request['gross_amount'];
        $signatureKey = $request['signature_key'];

        $signature = hash('sha512', $orderId . $statusCode . $grossAmount . $serverKey);
        
        return $signature === $signatureKey;
    }

    /**
     * Refund transaction
     */
    public function refund(string $transactionId, $amount = null): array
    {
        try {
            $refund = Transaction::refund($transactionId, $amount);
            return (array) $refund;
        } catch (\Exception $e) {
            \Log::error('Midtrans Refund Error: ' . $e->getMessage());
            throw $e;
        }
    }
}
```

### 5.2 Payment Factory
```php
<?php
// app/Services/Payment/PaymentProcessorFactory.php

namespace App\Services\Payment;

use App\Models\Payment;

class PaymentProcessorFactory
{
    public static function create(Payment $payment): PaymentProcessor
    {
        return match($payment->payment_method) {
            'bank_transfer' => new BankTransferProcessor($payment),
            'qris' => new QRISProcessor($payment),
            'gopay', 'dana', 'ovo', 'shopeepay' => new EWalletProcessor($payment),
            'visa', 'mastercard' => new CardProcessor($payment),
            default => throw new \InvalidArgumentException("Unknown payment method: {$payment->payment_method}")
        };
    }
}
```

---

## LANGKAH 6: Controllers & Routes

Lanjut di file implementasi berikutnya...
