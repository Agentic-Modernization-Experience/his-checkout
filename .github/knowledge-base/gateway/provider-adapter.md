# Provider Adapter Interface — Payment Gateway Abstraction

**Phase**: 3 — Gateway & API Contracts
**Current Provider**: Stripe
**Design Goal**: Enable provider replacement without changing the internal domain contract

---

## 1. Motivation

All payment provider specifics (Stripe API calls, Stripe error types, Stripe event names) must be isolated behind this adapter interface. The rest of the system interacts only with the domain-level adapter contract described in this document.

This ensures that:
- Replacing Stripe with another provider requires changes only inside the adapter layer.
- Domain services, webhooks handlers, and API controllers are never coupled to Stripe SDK types.
- A mock adapter can be used in tests without network calls.

---

## 2. Core Adapter Interface

```typescript
interface PaymentGatewayAdapter {
  /**
   * Creates a payment intent for the given order.
   * Must be idempotent with respect to idempotencyKey.
   */
  createPaymentIntent(params: CreatePaymentIntentParams): Promise<PaymentIntentResult>;

  /**
   * Confirms a previously created payment intent with a payment method.
   * Must be idempotent with respect to idempotencyKey.
   */
  confirmPaymentIntent(params: ConfirmPaymentIntentParams): Promise<PaymentIntentResult>;

  /**
   * Retrieves the current state of a payment intent directly from the provider.
   * Used for reconciliation and polling.
   */
  retrievePaymentIntent(providerIntentId: string): Promise<PaymentIntentResult>;

  /**
   * Cancels a payment intent. Safe to call on already-cancelled intents.
   */
  cancelPaymentIntent(providerIntentId: string, idempotencyKey: string): Promise<PaymentIntentResult>;

  /**
   * Issues a full or partial refund for a completed charge.
   */
  refundCharge(params: RefundParams): Promise<RefundResult>;

  /**
   * Parses and validates an inbound webhook payload, returning a normalised event.
   * Throws ProviderWebhookVerificationError if the signature is invalid.
   */
  parseWebhookEvent(rawBody: Buffer, signatureHeader: string): ProviderWebhookEvent;
}
```

---

## 3. Domain Types

### 3.1 CreatePaymentIntentParams

```typescript
interface CreatePaymentIntentParams {
  orderId: string;          // Internal order ID; stored as provider metadata
  amount: number;           // Smallest currency unit (centavos)
  currency: string;         // ISO 4217 (e.g., "BRL")
  paymentMethodTypes: PaymentMethodType[];  // ["credit_card", "pix", "boleto"]
  idempotencyKey: string;   // Format: `pi-create-{orderId}`
  metadata?: Record<string, string>;        // Additional tracing fields
}
```

### 3.2 ConfirmPaymentIntentParams

```typescript
interface ConfirmPaymentIntentParams {
  providerIntentId: string;          // Provider's payment intent ID
  paymentMethodId?: string;          // Required for credit card (pm_xxx in Stripe)
  paymentMethodType: PaymentMethodType;
  returnUrl?: string;                // Required for redirect-based flows (3DS)
  idempotencyKey: string;            // Format: `pi-confirm-{orderId}-{attemptNumber}`
}
```

### 3.3 PaymentIntentResult

```typescript
interface PaymentIntentResult {
  providerIntentId: string;          // Provider's opaque intent ID
  internalStatus: PaymentStatus;     // Mapped internal status (see api-contracts.md §4.3 — Canonical Status Lifecycle)
  nextAction?: PaymentNextAction;    // Present when client action is required
  providerRawStatus: string;         // Raw provider status; for logging only; never forwarded to clients
  amount: number;
  currency: string;
  createdAt: Date;
  updatedAt: Date;
}
```

### 3.4 PaymentNextAction

```typescript
type PaymentNextAction =
  | { type: 'redirect_to_url'; url: string; returnUrl: string }
  | { type: 'pix'; qrCode: string; qrCodeImageUrl: string; expiresAt: Date }
  | { type: 'boleto'; barcode: string; pdfUrl: string; expiresAt: Date };
```

### 3.5 RefundParams

```typescript
interface RefundParams {
  providerIntentId: string;
  amountCentavos?: number;   // Omit for full refund
  reason?: 'duplicate' | 'fraudulent' | 'requested_by_customer';
  idempotencyKey: string;    // Format: `refund-{orderId}-{reason}`
}

interface RefundResult {
  providerRefundId: string;
  status: 'succeeded' | 'pending' | 'failed';
  amountCentavos: number;
  createdAt: Date;
}
```

### 3.6 ProviderWebhookEvent

```typescript
interface ProviderWebhookEvent {
  eventId: string;           // Provider's unique event ID (for deduplication)
  eventType: InternalWebhookEventType;
  providerRawEventType: string;  // Provider-specific event string; for logging only
  orderId: string | null;    // Extracted from provider metadata; null if not present
  providerIntentId: string;
  occurredAt: Date;
  payload: WebhookEventPayload;
}

type InternalWebhookEventType =
  | 'payment_intent.succeeded'
  | 'payment_intent.failed'
  | 'payment_intent.requires_action'
  | 'payment_intent.cancelled'
  | 'charge.refunded'
  | 'charge.dispute.created'
  | 'unknown';  // Unrecognised events; must be accepted and ignored

type WebhookEventPayload =
  | { type: 'payment_intent.succeeded'; internalStatus: 'confirmed' }
  | { type: 'payment_intent.failed'; declineCode: string | null; internalStatus: 'failed' }
  | { type: 'payment_intent.requires_action'; internalStatus: 'requires_action' }
  | { type: 'payment_intent.cancelled'; internalStatus: 'cancelled' }
  | { type: 'charge.refunded'; refundedAmount: number; isFullRefund: boolean }
  | { type: 'charge.dispute.created'; disputeId: string }
  | { type: 'unknown' };
```

### 3.7 Enumerations

```typescript
type PaymentMethodType = 'credit_card' | 'pix' | 'boleto';

type PaymentStatus =
  | 'pending'
  | 'processing'
  | 'requires_action'
  | 'requires_capture'
  | 'confirmed'
  | 'failed'
  | 'cancelled'
  | 'refunded'
  | 'partially_refunded';
```

---

## 4. Error Contract

The adapter must throw only domain-level errors. It must never propagate raw SDK exceptions outside the adapter layer.

```typescript
class ProviderError extends Error {
  constructor(
    public readonly code: ProviderErrorCode,
    public readonly message: string,
    public readonly retryable: boolean,
    public readonly providerRawCode?: string,  // Original provider code; for internal logs only
  ) { super(message); }
}

type ProviderErrorCode =
  | 'payment_declined'
  | 'insufficient_funds'
  | 'card_expired'
  | 'do_not_honor'
  | 'authentication_required'
  | 'fraud_blocked'
  | 'invalid_request'           // Programming error — do not retry
  | 'authentication_failure'    // Wrong API key — alert; do not retry
  | 'gateway_timeout'           // SDK timeout; retryable
  | 'gateway_error'             // Provider 5xx; retryable
  | 'rate_limited'              // Provider rate limit; retryable with backoff
  | 'idempotency_conflict'      // Same key, different payload; do not retry
  | 'webhook_verification_failed';  // Invalid signature; reject
```

The `retryable` flag on `ProviderError` must be honoured by the calling service to decide whether to re-queue or escalate.

---

## 5. Stripe Adapter Implementation Contract

The Stripe adapter (`StripePaymentGatewayAdapter`) implements `PaymentGatewayAdapter` and is the only file permitted to import from the Stripe SDK.

### Mapping Responsibilities

| Adapter Method | Stripe SDK Call | Notes |
|---|---|---|
| `createPaymentIntent` | `stripe.paymentIntents.create()` | Passes `idempotencyKey` via SDK option |
| `confirmPaymentIntent` | `stripe.paymentIntents.confirm()` | Sets `return_url` for redirect flows |
| `retrievePaymentIntent` | `stripe.paymentIntents.retrieve()` | No idempotency key needed |
| `cancelPaymentIntent` | `stripe.paymentIntents.cancel()` | Idempotency key prevents duplicate cancel |
| `refundCharge` | `stripe.refunds.create()` | `payment_intent` field used to reference the charge |
| `parseWebhookEvent` | `stripe.webhooks.constructEvent()` | Throws `ProviderWebhookVerificationError` on failure |

### Stripe Status → Internal Status Mapping

Defined in `api-contracts.md §4.3 — Canonical Status Lifecycle`. The adapter must apply this mapping before returning `PaymentIntentResult.internalStatus`.

### Stripe Error → ProviderError Mapping

| Stripe Exception | ProviderErrorCode | retryable |
|---|---|---|
| `StripeCardError` | Mapped via decline code (see error-envelope.md §2.3) | false |
| `StripeInvalidRequestError` | `invalid_request` | false |
| `StripeAuthenticationError` | `authentication_failure` | false |
| `StripeRateLimitError` | `rate_limited` | true |
| `StripeAPIError` (5xx) | `gateway_error` | true |
| `StripeConnectionError` | `gateway_timeout` | true |
| `StripeIdempotencyError` | `idempotency_conflict` | false |
| Signature verification failure | `webhook_verification_failed` | false |

---

## 6. Mock Adapter for Testing

A `MockPaymentGatewayAdapter` must implement `PaymentGatewayAdapter` and support:

```typescript
interface MockPaymentGatewayAdapter extends PaymentGatewayAdapter {
  /** Pre-configure what the next createPaymentIntent call will return */
  setNextCreateResult(result: PaymentIntentResult | ProviderError): void;

  /** Pre-configure what the next confirmPaymentIntent call will return */
  setNextConfirmResult(result: PaymentIntentResult | ProviderError): void;

  /** Simulate an inbound webhook event */
  simulateWebhookEvent(event: ProviderWebhookEvent): void;

  /** Returns all calls recorded for assertions */
  getRecordedCalls(): AdapterCallRecord[];

  /** Reset all configured results and recorded calls */
  reset(): void;
}
```

The mock adapter must never make real network calls and must be injected via dependency injection in all unit and integration tests.

---

## 7. Fallback Behaviour for Ambiguous Provider States

When the provider returns an ambiguous or unknown state:

1. Log the raw provider status with full correlation fields.
2. Map to the most conservative internal status (`processing` if unclear).
3. Schedule a reconciliation poll (see `retry-policy.md §7`).
4. Never use `unknown` as a final terminal status — always resolve via polling or webhook.

Unknown `InternalWebhookEventType` values must result in a `200` response to the provider (to stop retries) and a log entry with `severity: warning`. No state mutation occurs.
