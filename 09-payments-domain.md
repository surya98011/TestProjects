# 09 — Payments Domain (Payment Lifecycle, PCI-DSS, Tokenization, 3DS, Reconciliation)

> **Priority**: 🔴 Critical  
> **Estimated Study Time**: 2 days  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: Payment Lifecycle

### 🔥 Q1. Explain the complete payment lifecycle from initiation to settlement.

**Answer:**

```
Customer                Merchant              Payment Gateway         Acquiring Bank        Issuing Bank
   │                      │                        │                      │                    │
   │ 1. Enter card        │                        │                      │                    │
   │    details           │                        │                      │                    │
   │─────────────────────→│                        │                      │                    │
   │                      │ 2. Create payment      │                      │                    │
   │                      │    (tokenized card)    │                      │                    │
   │                      │───────────────────────→│                      │                    │
   │                      │                        │ 3. Authorization     │                    │
   │                      │                        │    request           │                    │
   │                      │                        │───────────────────→│                    │
   │                      │                        │                      │ 4. Auth request    │
   │                      │                        │                      │───────────────────→│
   │                      │                        │                      │                    │
   │                      │                        │                      │ 5. Check balance,  │
   │                      │                        │                      │    fraud, limits   │
   │                      │                        │                      │                    │
   │                      │                        │                      │ 6. Auth response   │
   │                      │                        │                      │←───────────────────│
   │                      │                        │ 7. Auth response     │                    │
   │                      │                        │←───────────────────│                    │
   │                      │ 8. Auth result         │                      │                    │
   │                      │←───────────────────────│                      │                    │
   │ 9. Payment           │                        │                      │                    │
   │    confirmed         │                        │                      │                    │
   │←─────────────────────│                        │                      │                    │
   │                      │                        │                      │                    │
   │                      │ 10. Capture            │                      │                    │
   │                      │     (end of day batch) │                      │                    │
   │                      │───────────────────────→│                      │                    │
   │                      │                        │ 11. Settlement       │                    │
   │                      │                        │     (T+1 or T+2)    │                    │
   │                      │                        │───────────────────→│                    │
   │                      │                        │                      │ 12. Fund transfer  │
   │                      │                        │                      │───────────────────→│
   │                      │ 13. Settlement         │                      │                    │
   │                      │     confirmation       │                      │                    │
   │                      │←───────────────────────│                      │                    │
```

**Payment States:**

| State | Description | Transitions |
|-------|-------------|-------------|
| **INITIATED** | Payment request created | → AUTHORIZED, → FAILED |
| **AUTHORIZED** | Bank approved, funds held | → CAPTURED, → VOIDED |
| **CAPTURED** | Funds captured from customer | → SETTLED, → REFUND_INITIATED |
| **SETTLED** | Funds transferred to merchant | → REFUND_INITIATED |
| **VOIDED** | Authorization cancelled (before capture) | Terminal |
| **FAILED** | Payment declined or error | Terminal |
| **REFUND_INITIATED** | Refund requested | → REFUNDED |
| **REFUNDED** | Funds returned to customer | Terminal |

```java
// Payment State Machine
public enum PaymentStatus {
    INITIATED, AUTHORIZED, CAPTURED, SETTLED, VOIDED, FAILED, REFUND_INITIATED, REFUNDED;
    
    private static final Map<PaymentStatus, Set<PaymentStatus>> VALID_TRANSITIONS = Map.of(
        INITIATED, Set.of(AUTHORIZED, FAILED),
        AUTHORIZED, Set.of(CAPTURED, VOIDED, FAILED),
        CAPTURED, Set.of(SETTLED, REFUND_INITIATED),
        SETTLED, Set.of(REFUND_INITIATED),
        REFUND_INITIATED, Set.of(REFUNDED, FAILED)
    );
    
    public boolean canTransitionTo(PaymentStatus target) {
        return VALID_TRANSITIONS.getOrDefault(this, Set.of()).contains(target);
    }
}

@Service
public class PaymentStateMachine {
    
    @Transactional
    public Payment transition(String txnId, PaymentStatus newStatus) {
        Payment payment = paymentRepository.findByTxnIdForUpdate(txnId)
            .orElseThrow(() -> new PaymentNotFoundException(txnId));
        
        if (!payment.getStatus().canTransitionTo(newStatus)) {
            throw new InvalidStateTransitionException(
                String.format("Cannot transition from %s to %s", payment.getStatus(), newStatus));
        }
        
        PaymentStatus oldStatus = payment.getStatus();
        payment.setStatus(newStatus);
        payment.setUpdatedAt(Instant.now());
        
        // Publish state change event
        eventPublisher.publish(new PaymentStatusChanged(txnId, oldStatus, newStatus));
        
        return paymentRepository.save(payment);
    }
}
```

**Auth vs Capture:**
- **Authorization**: Bank checks if customer has funds and places a hold. No money moves.
- **Capture**: Actually moves the money. Can be immediate (auth+capture) or delayed.
- **Why separate?** Hotels authorize at check-in, capture at check-out (amount may change). E-commerce authorizes at order, captures at shipment.

---

### 🔥 Q2. What is the difference between Auth+Capture, Auth-only, and Sale?

**Answer:**

| Flow | Steps | Use Case |
|------|-------|----------|
| **Sale (Auth+Capture)** | Single step: authorize and capture together | E-commerce (immediate delivery), digital goods |
| **Auth-only** | Authorize first, capture later | Hotels, car rentals, pre-orders |
| **Pre-auth** | Authorize a higher amount, capture actual | Gas stations, restaurants (tip) |

```java
// Auth + Capture (Sale) — Most common for e-commerce
public PaymentResult processSale(PaymentRequest request) {
    GatewayResponse response = gateway.authAndCapture(
        request.getCardToken(),
        request.getAmount(),
        request.getCurrency()
    );
    // Money moves immediately
    return toResult(response);
}

// Auth-only — Hotel booking
public PaymentResult authorizeOnly(PaymentRequest request) {
    GatewayResponse response = gateway.authorize(
        request.getCardToken(),
        request.getAmount(),
        request.getCurrency()
    );
    // Funds are held but not captured
    // Auth expires in 7-30 days depending on card network
    return toResult(response);
}

// Capture later — When guest checks out
public PaymentResult capturePayment(String authorizationId, BigDecimal finalAmount) {
    // finalAmount can be <= authorized amount
    GatewayResponse response = gateway.capture(authorizationId, finalAmount);
    return toResult(response);
}

// Void — Cancel authorization before capture
public PaymentResult voidAuthorization(String authorizationId) {
    GatewayResponse response = gateway.voidAuth(authorizationId);
    // Releases the hold on customer's funds
    return toResult(response);
}
```

---

## Section 2: PCI-DSS

### 🔥 Q3. What is PCI-DSS? How does tokenization work?

**Answer:**

**PCI-DSS** (Payment Card Industry Data Security Standard) is a set of security standards for organizations that handle credit card data.

**PCI-DSS Compliance Levels:**

| Level | Criteria | Validation |
|-------|----------|-----------|
| Level 1 | > 6M transactions/year | Annual on-site audit (QSA) |
| Level 2 | 1M - 6M transactions/year | Annual SAQ + quarterly scan |
| Level 3 | 20K - 1M e-commerce transactions/year | Annual SAQ + quarterly scan |
| Level 4 | < 20K e-commerce transactions/year | Annual SAQ |

**Key PCI-DSS Requirements:**

| # | Requirement | Implementation |
|---|-------------|---------------|
| 1 | Install and maintain firewall | Network segmentation, WAF |
| 2 | Don't use vendor defaults | Change all default passwords |
| 3 | Protect stored cardholder data | Encryption at rest (AES-256) |
| 4 | Encrypt transmission | TLS 1.2+ for all card data in transit |
| 5 | Use antivirus | Endpoint protection |
| 6 | Develop secure systems | Secure SDLC, code reviews, OWASP |
| 7 | Restrict access (need-to-know) | RBAC, least privilege |
| 8 | Unique IDs for access | MFA, strong passwords |
| 9 | Restrict physical access | Data center security |
| 10 | Track and monitor access | Logging, audit trails |
| 11 | Regular security testing | Penetration testing, vulnerability scans |
| 12 | Maintain security policy | Documented policies, training |

**Tokenization:**

Tokenization replaces sensitive card data with a non-sensitive token. The actual card data is stored in a secure **token vault**.

```
Customer Card: 4111-1111-1111-1111
                    │
                    ▼
┌──────────────────────────────────┐
│         Token Vault               │
│  (PCI-DSS compliant, encrypted)  │
│                                  │
│  Token: tok_abc123def456         │
│  Card:  4111-1111-1111-1111      │
│  Exp:   12/26                    │
│  CVV:   Not stored (ever!)       │
└──────────────────────────────────┘
                    │
                    ▼
Your Application stores: tok_abc123def456
(Not PCI-DSS scope — no card data!)
```

```java
// Tokenization flow
@Service
public class TokenizationService {
    
    // Step 1: Client-side tokenization (card data never hits your server)
    // Frontend uses Stripe.js / Razorpay.js to tokenize
    // Your server only receives the token
    
    // Step 2: Use token for payment
    public PaymentResult chargeWithToken(String token, BigDecimal amount) {
        // Your server NEVER sees the actual card number
        return gateway.charge(token, amount);
    }
    
    // Step 3: Store token for recurring payments
    @Entity
    @Table(name = "saved_payment_methods")
    public class SavedPaymentMethod {
        @Id @GeneratedValue
        private Long id;
        
        private String userId;
        private String token;           // tok_abc123def456
        private String last4;           // "1111" (for display only)
        private String cardNetwork;     // "VISA"
        private String expiryMonth;     // "12"
        private String expiryYear;      // "2026"
        
        // ❌ NEVER store: full card number, CVV, magnetic stripe data
    }
}
```

**Reducing PCI Scope:**
- Use **client-side tokenization** (Stripe Elements, Razorpay.js) — card data never touches your server
- Use **hosted payment pages** — redirect to gateway's page
- Use **iframes** — gateway's form embedded in your page
- Result: Your application is **SAQ-A** (simplest compliance level)

---

## Section 3: 3D Secure (3DS)

### 💎 Q4. What is 3D Secure? How does 3DS2 work?

**Answer:**

3D Secure adds an **authentication layer** where the cardholder verifies their identity with the issuing bank. Shifts fraud liability from merchant to issuer.

**3DS1 vs 3DS2:**

| Feature | 3DS1 | 3DS2 |
|---------|------|------|
| User experience | Full-page redirect, password | Frictionless or challenge (OTP/biometric) |
| Data shared | Minimal | 150+ data points for risk assessment |
| Mobile support | Poor | Native mobile SDKs |
| Conversion impact | High drop-off (10-15%) | Low drop-off (frictionless for low-risk) |

**3DS2 Flow:**

```
Customer          Merchant           3DS Server        Issuer (ACS)
   │                 │                   │                  │
   │ 1. Pay          │                   │                  │
   │────────────────→│                   │                  │
   │                 │ 2. Auth Request   │                  │
   │                 │   + device data   │                  │
   │                 │──────────────────→│                  │
   │                 │                   │ 3. AReq          │
   │                 │                   │  (150+ data pts) │
   │                 │                   │─────────────────→│
   │                 │                   │                  │
   │                 │                   │ 4. Risk Analysis │
   │                 │                   │  (AI/ML scoring) │
   │                 │                   │                  │
   │                 │                   │ 5. ARes          │
   │                 │                   │←─────────────────│
   │                 │                   │                  │
   │                 │  ┌────────────────┴──────────────┐   │
   │                 │  │ Low Risk → Frictionless       │   │
   │                 │  │ (no customer interaction!)     │   │
   │                 │  │                                │   │
   │                 │  │ High Risk → Challenge          │   │
   │                 │  │ (OTP, biometric, push notif)  │   │
   │                 │  └────────────────┬──────────────┘   │
   │                 │                   │                  │
   │ 6. Challenge    │                   │                  │
   │    (if needed)  │                   │                  │
   │←────────────────│                   │                  │
   │                 │                   │                  │
   │ 7. OTP/Bio      │                   │                  │
   │────────────────→│──────────────────→│─────────────────→│
   │                 │                   │                  │
   │                 │ 8. Auth Result    │                  │
   │                 │←──────────────────│←─────────────────│
   │                 │                   │                  │
   │ 9. Payment      │                   │                  │
   │    confirmed    │                   │                  │
   │←────────────────│                   │                  │
```

**Liability Shift:**
- Without 3DS: Merchant bears fraud liability
- With 3DS: Issuer bears fraud liability (if they approved the authentication)
- This is why merchants want 3DS even though it adds friction

---

## Section 4: Reconciliation

### 🔥 Q5. How does payment reconciliation work? Design a reconciliation system.

**Answer:**

Reconciliation ensures that **your records match the payment gateway's records** and the **bank's settlement records**.

**Types of Reconciliation:**

| Type | Compares | Frequency | Purpose |
|------|----------|-----------|---------|
| **Transaction Reconciliation** | Internal DB vs Gateway | Daily | Detect missing/extra transactions |
| **Settlement Reconciliation** | Gateway vs Bank statement | Daily (T+1) | Verify money received |
| **Financial Reconciliation** | Expected vs Actual amounts | Monthly | Accounting accuracy |

```
Internal DB                    Gateway Report              Bank Statement
┌──────────────┐              ┌──────────────┐            ┌──────────────┐
│ TXN-001: ₹100│              │ TXN-001: ₹100│            │ Settled: ₹97 │
│ TXN-002: ₹200│              │ TXN-002: ₹200│            │ (₹100 - ₹3   │
│ TXN-003: ₹150│              │ TXN-003: ₹150│            │  gateway fee) │
│ TXN-004: ₹300│ ←── Match ──→│ TXN-004: ₹300│            │              │
│              │              │ TXN-005: ₹50 │ ← Missing! │              │
│ TXN-006: ₹75 │ ← Extra!    │              │            │              │
└──────────────┘              └──────────────┘            └──────────────┘

Exceptions:
- TXN-005: In gateway but not in our DB → investigate
- TXN-006: In our DB but not in gateway → investigate
- Amount mismatch: ₹97 settled vs ₹100 expected → gateway fee
```

```java
// Reconciliation Service
@Service
@Slf4j
public class ReconciliationService {
    
    @Scheduled(cron = "0 0 6 * * *") // Run at 6 AM daily
    @Transactional
    public ReconciliationReport reconcileDaily() {
        LocalDate yesterday = LocalDate.now().minusDays(1);
        
        // 1. Fetch internal records
        Map<String, Payment> internalPayments = paymentRepository
            .findByDateRange(yesterday.atStartOfDay(), yesterday.plusDays(1).atStartOfDay())
            .stream()
            .collect(Collectors.toMap(Payment::getTxnId, Function.identity()));
        
        // 2. Fetch gateway settlement report
        List<GatewayTransaction> gatewayTxns = gatewayClient.getSettlementReport(yesterday);
        Map<String, GatewayTransaction> gatewayMap = gatewayTxns.stream()
            .collect(Collectors.toMap(GatewayTransaction::getTxnId, Function.identity()));
        
        // 3. Compare
        List<ReconciliationException> exceptions = new ArrayList<>();
        
        // Check internal records against gateway
        for (Map.Entry<String, Payment> entry : internalPayments.entrySet()) {
            String txnId = entry.getKey();
            Payment internal = entry.getValue();
            GatewayTransaction gateway = gatewayMap.get(txnId);
            
            if (gateway == null) {
                exceptions.add(new ReconciliationException(txnId, "MISSING_AT_GATEWAY",
                    "Transaction exists internally but not in gateway report"));
            } else if (internal.getAmount().compareTo(gateway.getAmount()) != 0) {
                exceptions.add(new ReconciliationException(txnId, "AMOUNT_MISMATCH",
                    String.format("Internal: %s, Gateway: %s", internal.getAmount(), gateway.getAmount())));
            } else if (!internal.getStatus().name().equals(gateway.getStatus())) {
                exceptions.add(new ReconciliationException(txnId, "STATUS_MISMATCH",
                    String.format("Internal: %s, Gateway: %s", internal.getStatus(), gateway.getStatus())));
            }
        }
        
        // Check gateway records not in internal
        for (String txnId : gatewayMap.keySet()) {
            if (!internalPayments.containsKey(txnId)) {
                exceptions.add(new ReconciliationException(txnId, "MISSING_INTERNALLY",
                    "Transaction in gateway but not in internal records"));
            }
        }
        
        // 4. Generate report
        ReconciliationReport report = ReconciliationReport.builder()
            .date(yesterday)
            .totalInternal(internalPayments.size())
            .totalGateway(gatewayTxns.size())
            .matched(internalPayments.size() - exceptions.size())
            .exceptions(exceptions)
            .build();
        
        reconciliationReportRepository.save(report);
        
        if (!exceptions.isEmpty()) {
            alertService.sendReconciliationAlert(report);
        }
        
        return report;
    }
}
```

---

## Section 5: Chargebacks

### 💎 Q6. What are chargebacks? How do you handle them?

**Answer:**

A **chargeback** is when a customer disputes a transaction with their bank, and the bank reverses the payment.

**Chargeback Flow:**
```
1. Customer disputes charge with issuing bank
2. Issuing bank initiates chargeback
3. Acquiring bank notifies merchant/gateway
4. Merchant has 7-30 days to respond with evidence
5. Bank reviews evidence and makes decision
6. If merchant loses → funds deducted + chargeback fee ($15-$100)
```

**Chargeback Reasons:**

| Code | Reason | Prevention |
|------|--------|-----------|
| **Fraud** | Unauthorized transaction | 3DS, fraud detection, AVS/CVV |
| **Not received** | Product/service not delivered | Tracking, delivery confirmation |
| **Not as described** | Product different from description | Accurate descriptions, photos |
| **Duplicate** | Charged twice | Idempotency keys |
| **Subscription** | Unwanted recurring charge | Clear cancellation process |

```java
// Chargeback handling
@Service
public class ChargebackService {
    
    @KafkaListener(topics = "chargeback-notifications")
    public void handleChargeback(ChargebackEvent event) {
        log.warn("Chargeback received: txnId={}, amount={}, reason={}", 
            event.getTxnId(), event.getAmount(), event.getReasonCode());
        
        // 1. Update payment status
        paymentStateMachine.transition(event.getTxnId(), PaymentStatus.CHARGEBACK);
        
        // 2. Block suspicious user if fraud
        if (event.getReasonCode().startsWith("FRAUD")) {
            fraudService.flagUser(event.getUserId());
        }
        
        // 3. Gather evidence for representment
        ChargebackEvidence evidence = ChargebackEvidence.builder()
            .txnId(event.getTxnId())
            .transactionReceipt(getTransactionReceipt(event.getTxnId()))
            .deliveryProof(getDeliveryProof(event.getOrderId()))
            .customerCommunication(getCustomerEmails(event.getUserId()))
            .ipAddress(getTransactionIP(event.getTxnId()))
            .deviceFingerprint(getDeviceInfo(event.getTxnId()))
            .threeDSResult(get3DSResult(event.getTxnId()))
            .build();
        
        chargebackRepository.save(evidence);
        
        // 4. Auto-respond if evidence is strong
        if (evidence.isStrongCase()) {
            gatewayClient.submitChargebackResponse(event.getChargebackId(), evidence);
        } else {
            // Queue for manual review
            chargebackQueue.add(event.getChargebackId());
            notifyOpsTeam(event);
        }
    }
}
```

**Chargeback Prevention Metrics:**
- **Chargeback Rate**: Must stay below 1% (Visa/Mastercard threshold)
- Above 1% → Monitoring program → Higher fees → Potential termination
- Target: < 0.5% chargeback rate

---

## Section 6: Retry Strategies in Payments

### 🔥 Q7. How do you implement retry strategies for payment processing?

**Answer:**

**Key Principle**: Not all payment failures are retryable!

| Error Type | Retryable? | Example | Action |
|-----------|-----------|---------|--------|
| **Network timeout** | ✅ Yes | Gateway didn't respond | Retry with backoff |
| **5xx Server error** | ✅ Yes | Gateway internal error | Retry with backoff |
| **Insufficient funds** | ❌ No | Card declined | Return failure |
| **Invalid card** | ❌ No | Expired card | Return failure |
| **Fraud detected** | ❌ No | Flagged by fraud system | Block and alert |
| **Rate limited** | ✅ Yes (after delay) | Too many requests | Retry after delay |

```java
@Service
public class PaymentRetryService {
    
    @Retryable(
        retryFor = {GatewayTimeoutException.class, GatewayServerException.class},
        noRetryFor = {CardDeclinedException.class, FraudException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000)
    )
    public GatewayResponse chargeWithRetry(PaymentRequest request) {
        return gateway.charge(request);
    }
    
    @Recover
    public GatewayResponse recoverFromRetry(Exception ex, PaymentRequest request) {
        log.error("All retries exhausted for txnId={}: {}", request.getTxnId(), ex.getMessage());
        
        // Check if payment actually went through (idempotent check)
        Optional<GatewayResponse> existing = gateway.getPaymentStatus(request.getIdempotencyKey());
        if (existing.isPresent()) {
            return existing.get(); // Payment succeeded on a previous attempt
        }
        
        // Mark as failed
        return GatewayResponse.failed("Payment failed after retries: " + ex.getMessage());
    }
}
```

**Exponential Backoff with Jitter:**
```java
public Duration calculateBackoff(int attempt) {
    long baseDelay = 1000; // 1 second
    long maxDelay = 30000; // 30 seconds
    
    // Exponential: 1s, 2s, 4s, 8s, 16s, 30s (capped)
    long exponentialDelay = Math.min(baseDelay * (long) Math.pow(2, attempt), maxDelay);
    
    // Add jitter to prevent thundering herd
    long jitter = ThreadLocalRandom.current().nextLong(0, exponentialDelay / 2);
    
    return Duration.ofMillis(exponentialDelay + jitter);
}
```

**⚠️ Critical: Always check payment status before retrying!**
```java
// The gateway might have processed the payment but the response was lost
// Retrying without checking = DOUBLE CHARGE!

public GatewayResponse safeRetry(PaymentRequest request) {
    // Step 1: Check if previous attempt succeeded
    Optional<GatewayResponse> existing = gateway.getByIdempotencyKey(request.getIdempotencyKey());
    if (existing.isPresent()) {
        return existing.get(); // Already processed!
    }
    
    // Step 2: Safe to retry
    return gateway.charge(request);
}
```

---

## Section 7: Payment Methods (India-specific)

### 💎 Q8. Explain UPI, Net Banking, and Wallet payment flows in India.

**Answer:**

**UPI (Unified Payments Interface):**
```
Customer App          PSP (PhonePe/GPay)      NPCI (UPI Switch)      Customer's Bank
     │                      │                       │                      │
     │ 1. Enter UPI ID      │                       │                      │
     │    or scan QR         │                       │                      │
     │─────────────────────→│                       │                      │
     │                      │ 2. Collect request     │                      │
     │                      │───────────────────────→│                      │
     │                      │                       │ 3. Route to bank     │
     │                      │                       │─────────────────────→│
     │                      │                       │                      │
     │ 4. Enter UPI PIN     │                       │                      │
     │─────────────────────→│───────────────────────→│─────────────────────→│
     │                      │                       │                      │
     │                      │                       │ 5. Debit account     │
     │                      │                       │←─────────────────────│
     │                      │ 6. Success response    │                      │
     │                      │←───────────────────────│                      │
     │ 7. Payment confirmed │                       │                      │
     │←─────────────────────│                       │                      │
```

**Key UPI Facts:**
- Real-time settlement (instant)
- Transaction limit: ₹1 lakh per transaction (₹2 lakh for some categories)
- Zero MDR (Merchant Discount Rate) for most transactions
- UPI handles 10+ billion transactions/month in India

**Payment Method Comparison (India):**

| Method | Settlement | Limit | MDR | Refund Time |
|--------|-----------|-------|-----|-------------|
| **UPI** | Instant | ₹1-2 lakh | 0% | Instant |
| **Credit Card** | T+2 to T+3 | Card limit | 1.5-2.5% | 5-7 days |
| **Debit Card** | T+2 | Account balance | 0.4-0.9% | 5-7 days |
| **Net Banking** | T+1 to T+2 | Bank limit | ₹5-15 flat | 5-7 days |
| **Wallet** | Instant | ₹10,000 (KYC: ₹2 lakh) | 1-2% | Instant |

---

## Quick Revision Checklist

- [ ] Payment Lifecycle: Initiated → Authorized → Captured → Settled
- [ ] Auth vs Capture: Auth holds funds, Capture moves funds
- [ ] Sale = Auth + Capture in one step
- [ ] PCI-DSS: 12 requirements, 4 compliance levels
- [ ] Tokenization: Replace card data with token, vault stores actual data
- [ ] Reduce PCI scope: Client-side tokenization, hosted payment pages
- [ ] 3DS2: Frictionless (low risk) or Challenge (high risk), liability shift
- [ ] Reconciliation: Internal vs Gateway vs Bank, daily automated
- [ ] Chargebacks: Customer disputes, evidence-based response, < 1% rate
- [ ] Retry Strategy: Only retry network/server errors, not business declines
- [ ] Always check payment status before retry (prevent double charge)
- [ ] Idempotency: Client-generated key, cache response, handle concurrent requests
- [ ] UPI: Instant settlement, zero MDR, 10B+ txns/month in India
