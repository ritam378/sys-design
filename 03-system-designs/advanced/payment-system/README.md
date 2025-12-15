# Design a Payment System

## Table of Contents
- [Problem Statement](#problem-statement)
- [Requirements](#requirements)
- [Payment Flow Overview](#payment-flow-overview)
- [High-Level Architecture](#high-level-architecture)
- [Core Components](#core-components)
- [Payment Methods](#payment-methods)
- [Security and Compliance](#security-and-compliance)
- [Fraud Detection](#fraud-detection)
- [Idempotency and Reliability](#idempotency-and-reliability)
- [Reconciliation and Ledger](#reconciliation-and-ledger)
- [Database Design](#database-design)
- [API Design](#api-design)
- [Implementation Examples](#implementation-examples)
- [Real-World Examples](#real-world-examples)
- [Interview Tips](#interview-tips)

---

## Problem Statement

Design a **payment processing system** like Stripe, PayPal, or Square that can:
- Process online payments securely
- Support multiple payment methods (cards, wallets, bank transfers)
- Handle high transaction volumes (millions per day)
- Ensure PCI DSS compliance
- Prevent fraud and double-spending
- Support refunds, chargebacks, and disputes
- Provide real-time payment status
- Support multi-currency transactions

**Similar to**: Stripe, PayPal, Square, Adyen

---

## Requirements

### Functional Requirements

1. **Payment Processing**:
   - Accept credit/debit cards, digital wallets, bank transfers
   - Authorization and capture flow
   - Immediate and delayed capture

2. **Payment Operations**:
   - Create payment intent
   - Confirm payment
   - Capture authorized payment
   - Refund transactions
   - Handle disputes and chargebacks

3. **Multi-party Payments**:
   - Marketplace payments (split between seller and platform)
   - Escrow payments
   - Recurring payments/subscriptions

4. **Compliance**:
   - PCI DSS Level 1 compliance
   - KYC (Know Your Customer)
   - AML (Anti-Money Laundering)

### Non-Functional Requirements

1. **Security**: Encryption, tokenization, PCI compliance
2. **Reliability**: 99.99% uptime, no double-charging
3. **Consistency**: Strong consistency for payments
4. **Performance**: <500ms payment processing
5. **Scalability**: Handle millions of transactions/day
6. **Auditability**: Complete audit trail for all transactions

---

## Payment Flow Overview

### Standard Payment Flow

```
1. Customer initiates payment
   ↓
2. Merchant creates Payment Intent (with amount, currency)
   ↓
3. Customer provides payment method (card details)
   ↓
4. Payment processor authorizes payment (hold funds)
   ↓
5. Merchant captures payment (settle funds)
   ↓
6. Funds transferred from customer to merchant
   ↓
7. Payment confirmed, order fulfilled
```

### Authorization vs Capture

```
AUTHORIZATION (Reserve funds)
├─ Check card validity
├─ Verify available balance
├─ Place hold on funds
└─ Return authorization code

CAPTURE (Settle funds)
├─ Request fund transfer
├─ Deduct from customer account
├─ Credit merchant account
└─ Complete transaction
```

---

## High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                         Customer                                │
│                    (Web/Mobile App)                            │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────────┐
│                      Merchant System                            │
│              (E-commerce, POS, etc.)                           │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ↓ Create Payment Intent
┌────────────────────────────────────────────────────────────────┐
│                   Payment Gateway API                           │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  • Authentication & Rate Limiting                        │ │
│  │  • Request Validation                                    │ │
│  │  • Idempotency Check                                     │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────┬───────────────────────────────────────────────┘
                 │
        ┌────────┴────────┐
        ↓                 ↓
┌──────────────┐   ┌──────────────┐
│   Fraud      │   │  Tokenization│
│  Detection   │   │   Service    │
└──────┬───────┘   └──────┬───────┘
       │                  │
       └────────┬─────────┘
                ↓
┌────────────────────────────────────────────────────────────────┐
│              Payment Processing Engine                          │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │Authorization│  │   Capture   │  │   Refund    │          │
│  │   Service   │  │   Service   │  │   Service   │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
└─────────┼─────────────────┼─────────────────┼─────────────────┘
          │                 │                 │
          └────────┬────────┴────────┬────────┘
                   ↓                 ↓
        ┌──────────────────┐  ┌──────────────────┐
        │  Card Networks   │  │   Banks (ACH,    │
        │ (Visa, Master)   │  │   Wire Transfer) │
        └──────────────────┘  └──────────────────┘
                   │
                   ↓
        ┌──────────────────────────────┐
        │   Ledger & Reconciliation    │
        │        Service               │
        └──────────────────────────────┘
                   │
                   ↓
        ┌──────────────────────────────┐
        │   Database (PostgreSQL)      │
        │   - Transactions             │
        │   - Payment Methods          │
        │   - Accounts                 │
        └──────────────────────────────┘
```

---

## Core Components

### 1. Payment Gateway API

```python
from flask import Flask, request, jsonify
import uuid
from datetime import datetime

app = Flask(__name__)

class PaymentGateway:
    def __init__(self):
        self.idempotency_keys = {}  # key -> response
        self.fraud_detector = FraudDetector()
        self.tokenizer = TokenizationService()
        self.processor = PaymentProcessor()

    def create_payment_intent(self, merchant_id, amount, currency, metadata):
        """
        Create payment intent (Step 1 of payment flow)

        Returns payment_intent_id that client uses to complete payment
        """
        payment_intent_id = str(uuid.uuid4())

        # Store payment intent
        intent = {
            'id': payment_intent_id,
            'merchant_id': merchant_id,
            'amount': amount,
            'currency': currency,
            'status': 'requires_payment_method',
            'metadata': metadata,
            'created_at': datetime.utcnow().isoformat()
        }

        db.save_payment_intent(intent)

        return {
            'payment_intent_id': payment_intent_id,
            'client_secret': self._generate_client_secret(payment_intent_id),
            'status': 'requires_payment_method'
        }

    def confirm_payment(self, payment_intent_id, payment_method, idempotency_key):
        """
        Confirm payment with payment method (card, wallet, etc.)

        Uses idempotency key to prevent duplicate payments
        """
        # Check idempotency
        if idempotency_key in self.idempotency_keys:
            return self.idempotency_keys[idempotency_key]

        # Get payment intent
        intent = db.get_payment_intent(payment_intent_id)

        if intent['status'] != 'requires_payment_method':
            raise ValueError(f"Invalid status: {intent['status']}")

        # Tokenize payment method (never store raw card details)
        token = self.tokenizer.tokenize(payment_method)

        # Fraud detection
        fraud_score = self.fraud_detector.assess_risk(
            amount=intent['amount'],
            merchant_id=intent['merchant_id'],
            payment_method=token,
            ip_address=request.remote_addr
        )

        if fraud_score > 0.8:  # High risk
            return self._handle_fraud(payment_intent_id, fraud_score)

        # Authorize payment
        auth_result = self.processor.authorize(
            amount=intent['amount'],
            currency=intent['currency'],
            payment_method_token=token
        )

        if auth_result['status'] == 'succeeded':
            # Update intent status
            db.update_payment_intent(payment_intent_id, {
                'status': 'requires_capture',
                'authorization_code': auth_result['auth_code'],
                'payment_method_token': token
            })

            response = {
                'status': 'requires_capture',
                'authorization_code': auth_result['auth_code']
            }
        else:
            response = {
                'status': 'failed',
                'error': auth_result['error']
            }

        # Store for idempotency
        self.idempotency_keys[idempotency_key] = response

        return response

    def capture_payment(self, payment_intent_id, amount=None):
        """
        Capture authorized payment (settle funds)

        amount: Optional, for partial capture
        """
        intent = db.get_payment_intent(payment_intent_id)

        if intent['status'] != 'requires_capture':
            raise ValueError("Payment not authorized")

        capture_amount = amount or intent['amount']

        # Capture funds
        capture_result = self.processor.capture(
            authorization_code=intent['authorization_code'],
            amount=capture_amount
        )

        if capture_result['status'] == 'succeeded':
            # Create transaction record
            transaction_id = str(uuid.uuid4())
            transaction = {
                'id': transaction_id,
                'payment_intent_id': payment_intent_id,
                'type': 'capture',
                'amount': capture_amount,
                'status': 'succeeded',
                'captured_at': datetime.utcnow().isoformat()
            }
            db.save_transaction(transaction)

            # Update intent
            db.update_payment_intent(payment_intent_id, {
                'status': 'succeeded',
                'captured_amount': capture_amount
            })

            # Record in ledger
            ledger.record_payment(
                from_account=intent['customer_id'],
                to_account=intent['merchant_id'],
                amount=capture_amount,
                transaction_id=transaction_id
            )

            return {'status': 'succeeded', 'transaction_id': transaction_id}
        else:
            return {'status': 'failed', 'error': capture_result['error']}

@app.route('/v1/payment_intents', methods=['POST'])
def create_payment_intent():
    """
    POST /v1/payment_intents
    {
      "amount": 1000,
      "currency": "usd",
      "merchant_id": "merch_123"
    }
    """
    data = request.json
    gateway = PaymentGateway()

    result = gateway.create_payment_intent(
        merchant_id=data['merchant_id'],
        amount=data['amount'],
        currency=data['currency'],
        metadata=data.get('metadata', {})
    )

    return jsonify(result), 200

@app.route('/v1/payment_intents/<intent_id>/confirm', methods=['POST'])
def confirm_payment(intent_id):
    """
    POST /v1/payment_intents/{intent_id}/confirm
    {
      "payment_method": {
        "type": "card",
        "card": {...}
      }
    }

    Headers:
      Idempotency-Key: unique_key_123
    """
    data = request.json
    idempotency_key = request.headers.get('Idempotency-Key')

    if not idempotency_key:
        return jsonify({'error': 'Idempotency-Key required'}), 400

    gateway = PaymentGateway()
    result = gateway.confirm_payment(
        payment_intent_id=intent_id,
        payment_method=data['payment_method'],
        idempotency_key=idempotency_key
    )

    return jsonify(result), 200

@app.route('/v1/payment_intents/<intent_id>/capture', methods=['POST'])
def capture_payment(intent_id):
    """
    POST /v1/payment_intents/{intent_id}/capture
    {
      "amount": 1000  # Optional, for partial capture
    }
    """
    data = request.json
    gateway = PaymentGateway()

    result = gateway.capture_payment(
        payment_intent_id=intent_id,
        amount=data.get('amount')
    )

    return jsonify(result), 200
```

### 2. Payment Processor

```python
class PaymentProcessor:
    """
    Handles communication with card networks and banks
    """

    def __init__(self):
        self.visa_gateway = VisaGateway()
        self.mastercard_gateway = MastercardGateway()
        self.ach_processor = ACHProcessor()

    def authorize(self, amount, currency, payment_method_token):
        """
        Authorize payment (hold funds)

        Returns authorization code if successful
        """
        payment_method = self._get_payment_method(payment_method_token)

        if payment_method['type'] == 'card':
            return self._authorize_card(amount, currency, payment_method)
        elif payment_method['type'] == 'bank_account':
            return self._authorize_ach(amount, currency, payment_method)
        else:
            raise ValueError(f"Unsupported payment method: {payment_method['type']}")

    def _authorize_card(self, amount, currency, card):
        """Authorize card payment through card network"""

        # Determine card network
        card_network = self._get_card_network(card['number'])

        if card_network == 'visa':
            gateway = self.visa_gateway
        elif card_network == 'mastercard':
            gateway = self.mastercard_gateway
        else:
            return {'status': 'failed', 'error': 'Unsupported card'}

        # Send authorization request
        auth_request = {
            'amount': amount,
            'currency': currency,
            'card_number': card['number'],
            'cvv': card['cvv'],
            'expiry': card['expiry'],
            'billing_address': card['billing_address']
        }

        response = gateway.authorize(auth_request)

        if response['approved']:
            return {
                'status': 'succeeded',
                'auth_code': response['authorization_code'],
                'processor_response': response
            }
        else:
            return {
                'status': 'failed',
                'error': response['decline_reason']
            }

    def capture(self, authorization_code, amount):
        """
        Capture authorized payment (settle funds)
        """
        # Look up authorization
        auth = db.get_authorization(authorization_code)

        if not auth:
            return {'status': 'failed', 'error': 'Authorization not found'}

        if amount > auth['authorized_amount']:
            return {'status': 'failed', 'error': 'Capture amount exceeds authorized amount'}

        # Send capture request to payment network
        gateway = self._get_gateway(auth['card_network'])

        capture_request = {
            'authorization_code': authorization_code,
            'amount': amount
        }

        response = gateway.capture(capture_request)

        if response['success']:
            return {
                'status': 'succeeded',
                'settlement_id': response['settlement_id']
            }
        else:
            return {
                'status': 'failed',
                'error': response['error']
            }

    def refund(self, transaction_id, amount):
        """
        Refund a captured payment
        """
        transaction = db.get_transaction(transaction_id)

        if transaction['status'] != 'succeeded':
            return {'status': 'failed', 'error': 'Transaction not captured'}

        if amount > transaction['amount']:
            return {'status': 'failed', 'error': 'Refund exceeds transaction amount'}

        # Send refund request
        gateway = self._get_gateway(transaction['card_network'])

        refund_request = {
            'original_transaction_id': transaction_id,
            'amount': amount
        }

        response = gateway.refund(refund_request)

        if response['success']:
            # Record refund transaction
            refund_transaction = {
                'id': str(uuid.uuid4()),
                'type': 'refund',
                'original_transaction_id': transaction_id,
                'amount': amount,
                'status': 'succeeded',
                'refunded_at': datetime.utcnow().isoformat()
            }
            db.save_transaction(refund_transaction)

            return {
                'status': 'succeeded',
                'refund_id': refund_transaction['id']
            }
        else:
            return {
                'status': 'failed',
                'error': response['error']
            }
```

---

## Payment Methods

### 1. Credit/Debit Cards

```python
class CardPayment:
    """Handle card payments"""

    def validate_card(self, card_number, cvv, expiry):
        """
        Validate card details
        - Luhn algorithm for card number
        - CVV format
        - Expiry date
        """
        if not self._luhn_check(card_number):
            return False, "Invalid card number"

        if not self._validate_cvv(cvv):
            return False, "Invalid CVV"

        if not self._validate_expiry(expiry):
            return False, "Card expired"

        return True, None

    def _luhn_check(self, card_number):
        """Luhn algorithm to validate card number"""
        def digits_of(n):
            return [int(d) for d in str(n)]

        digits = digits_of(card_number)
        odd_digits = digits[-1::-2]
        even_digits = digits[-2::-2]

        checksum = sum(odd_digits)
        for d in even_digits:
            checksum += sum(digits_of(d*2))

        return checksum % 10 == 0

    def get_card_brand(self, card_number):
        """Determine card brand from BIN (first 6 digits)"""
        bin_number = card_number[:6]

        if bin_number[0] == '4':
            return 'visa'
        elif bin_number[:2] in ['51', '52', '53', '54', '55']:
            return 'mastercard'
        elif bin_number[:2] in ['34', '37']:
            return 'amex'
        elif bin_number[:4] == '6011':
            return 'discover'
        else:
            return 'unknown'
```

### 2. Digital Wallets (Apple Pay, Google Pay)

```python
class WalletPayment:
    """Handle digital wallet payments"""

    def process_apple_pay(self, payment_token):
        """
        Process Apple Pay token
        Token is encrypted payment data
        """
        # Decrypt Apple Pay token
        decrypted = self._decrypt_apple_pay_token(payment_token)

        # Extract card details
        card_data = {
            'number': decrypted['applicationPrimaryAccountNumber'],
            'expiry': decrypted['applicationExpirationDate'],
            'cvv': decrypted['applicationCVV']
        }

        # Process as normal card payment
        return self.card_processor.process(card_data)

    def process_google_pay(self, payment_token):
        """Process Google Pay token"""
        # Similar to Apple Pay
        decrypted = self._decrypt_google_pay_token(payment_token)

        return self.card_processor.process(decrypted['card_data'])
```

### 3. Bank Transfers (ACH, Wire)

```python
class BankTransferPayment:
    """Handle bank transfer payments"""

    def initiate_ach_transfer(self, from_account, to_account, amount):
        """
        Initiate ACH transfer
        Takes 1-3 business days to settle
        """
        transfer_id = str(uuid.uuid4())

        transfer = {
            'id': transfer_id,
            'type': 'ach',
            'from_routing': from_account['routing_number'],
            'from_account': from_account['account_number'],
            'to_routing': to_account['routing_number'],
            'to_account': to_account['account_number'],
            'amount': amount,
            'status': 'pending',
            'initiated_at': datetime.utcnow().isoformat()
        }

        # Submit to ACH network
        ach_result = self.ach_processor.submit(transfer)

        if ach_result['accepted']:
            db.save_transfer(transfer)
            return {'status': 'pending', 'transfer_id': transfer_id}
        else:
            return {'status': 'failed', 'error': ach_result['error']}
```

---

## Security and Compliance

### 1. PCI DSS Compliance

```python
class PCICompliance:
    """
    PCI DSS (Payment Card Industry Data Security Standard)

    Requirements:
    1. Never store full card numbers in plain text
    2. Use tokenization
    3. Encrypt data in transit (TLS)
    4. Encrypt data at rest
    5. Maintain access logs
    6. Regular security audits
    """

    def __init__(self):
        self.encryption_key = self._load_encryption_key()

    def encrypt_sensitive_data(self, data):
        """Encrypt sensitive data at rest"""
        from cryptography.fernet import Fernet

        f = Fernet(self.encryption_key)
        encrypted = f.encrypt(data.encode())
        return encrypted

    def mask_card_number(self, card_number):
        """
        Mask card number for display
        Only show last 4 digits
        """
        return f"****-****-****-{card_number[-4:]}"

    def log_access(self, user_id, action, resource):
        """Log all access to payment data"""
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'user_id': user_id,
            'action': action,
            'resource': resource,
            'ip_address': request.remote_addr
        }
        audit_log.write(log_entry)
```

### 2. Tokenization

```python
class TokenizationService:
    """
    Replace sensitive card data with tokens
    Tokens are randomly generated, cannot be reverse-engineered
    """

    def tokenize(self, payment_method):
        """
        Convert payment method to token
        Store mapping in secure vault
        """
        token = self._generate_token()

        # Store in secure vault (encrypted)
        vault.store(token, {
            'type': payment_method['type'],
            'card_number': self._encrypt(payment_method['card']['number']),
            'expiry': payment_method['card']['expiry'],
            'billing_address': payment_method['billing_address'],
            'created_at': datetime.utcnow().isoformat()
        })

        return token

    def detokenize(self, token):
        """Retrieve original payment method from token"""
        encrypted_data = vault.retrieve(token)

        if not encrypted_data:
            raise ValueError("Invalid token")

        return {
            'type': encrypted_data['type'],
            'card_number': self._decrypt(encrypted_data['card_number']),
            'expiry': encrypted_data['expiry'],
            'billing_address': encrypted_data['billing_address']
        }

    def _generate_token(self):
        """Generate cryptographically secure random token"""
        import secrets
        return f"tok_{secrets.token_urlsafe(32)}"
```

---

## Fraud Detection

```python
class FraudDetector:
    """
    Detect fraudulent transactions using rules and ML
    """

    def assess_risk(self, amount, merchant_id, payment_method, ip_address):
        """
        Calculate fraud risk score (0-1)

        Higher score = higher risk
        """
        risk_score = 0.0

        # Rule-based checks
        risk_score += self._check_velocity(payment_method)
        risk_score += self._check_amount_anomaly(amount, merchant_id)
        risk_score += self._check_geolocation(ip_address, payment_method)
        risk_score += self._check_device_fingerprint()

        # ML-based prediction
        ml_score = self._ml_model_predict(amount, merchant_id, payment_method)
        risk_score += ml_score * 0.4

        return min(risk_score, 1.0)

    def _check_velocity(self, payment_method):
        """
        Check transaction velocity
        Many transactions in short time = suspicious
        """
        recent_txns = db.get_recent_transactions(
            payment_method_token=payment_method['token'],
            time_window=3600  # Last hour
        )

        if len(recent_txns) > 10:  # More than 10 transactions/hour
            return 0.3
        elif len(recent_txns) > 5:
            return 0.1

        return 0.0

    def _check_amount_anomaly(self, amount, merchant_id):
        """
        Check if amount is anomalous for this merchant
        """
        avg_amount = db.get_average_transaction_amount(merchant_id)

        if amount > avg_amount * 10:  # 10x average
            return 0.4
        elif amount > avg_amount * 5:
            return 0.2

        return 0.0

    def _check_geolocation(self, ip_address, payment_method):
        """
        Check if IP location matches card billing address
        """
        ip_location = geoip.get_location(ip_address)
        card_country = payment_method.get('billing_address', {}).get('country')

        if ip_location['country'] != card_country:
            return 0.3  # Different countries

        return 0.0

    def _ml_model_predict(self, amount, merchant_id, payment_method):
        """
        Use ML model to predict fraud probability

        Features:
        - Transaction amount
        - Merchant category
        - Time of day
        - Device fingerprint
        - Historical behavior
        """
        features = self._extract_features(amount, merchant_id, payment_method)

        # Load pre-trained model
        fraud_prob = ml_model.predict(features)

        return fraud_prob

# 3D Secure (Additional authentication)
class ThreeDSecure:
    """
    3D Secure 2.0 for additional cardholder authentication
    Used for high-risk transactions
    """

    def initiate_3ds(self, payment_intent_id, card):
        """
        Initiate 3D Secure authentication
        Redirects customer to bank for authentication
        """
        auth_request = {
            'payment_intent_id': payment_intent_id,
            'card_number': card['number'],
            'amount': payment_intent['amount'],
            'merchant_id': payment_intent['merchant_id']
        }

        # Call 3DS provider (e.g., Stripe Radar)
        response = three_ds_provider.create_authentication(auth_request)

        return {
            'status': 'requires_authentication',
            'redirect_url': response['authentication_url'],
            'session_id': response['session_id']
        }

    def complete_3ds(self, session_id):
        """
        Complete 3DS authentication after customer returns
        """
        result = three_ds_provider.get_authentication_result(session_id)

        if result['authenticated']:
            return {'status': 'authenticated', 'liability_shift': True}
        else:
            return {'status': 'failed', 'reason': result['failure_reason']}
```

---

## Idempotency and Reliability

```python
class IdempotencyManager:
    """
    Prevent duplicate payments using idempotency keys
    """

    def __init__(self):
        self.redis = redis.Redis()

    def check_idempotency(self, idempotency_key):
        """
        Check if request with this key was already processed
        """
        cached_response = self.redis.get(f"idempotency:{idempotency_key}")

        if cached_response:
            # Request already processed
            return json.loads(cached_response)

        return None

    def store_response(self, idempotency_key, response):
        """
        Store response for idempotency
        Keep for 24 hours
        """
        self.redis.setex(
            f"idempotency:{idempotency_key}",
            86400,  # 24 hours
            json.dumps(response)
        )

    def process_with_idempotency(self, idempotency_key, func, *args, **kwargs):
        """
        Execute function with idempotency guarantee
        """
        # Check if already processed
        cached = self.check_idempotency(idempotency_key)
        if cached:
            return cached

        # Use distributed lock to prevent concurrent execution
        lock = redis_lock.Lock(self.redis, f"lock:{idempotency_key}")

        if lock.acquire(blocking=True, timeout=10):
            try:
                # Double-check after acquiring lock
                cached = self.check_idempotency(idempotency_key)
                if cached:
                    return cached

                # Execute function
                result = func(*args, **kwargs)

                # Store result
                self.store_response(idempotency_key, result)

                return result
            finally:
                lock.release()
        else:
            raise TimeoutError("Could not acquire lock for idempotency")

# Usage
@app.route('/v1/charges', methods=['POST'])
def create_charge():
    idempotency_key = request.headers.get('Idempotency-Key')

    if not idempotency_key:
        return jsonify({'error': 'Idempotency-Key required'}), 400

    idempotency = IdempotencyManager()

    result = idempotency.process_with_idempotency(
        idempotency_key,
        process_charge,
        request.json
    )

    return jsonify(result), 200
```

### Retry Logic

```python
class PaymentRetry:
    """
    Retry failed payments with exponential backoff
    """

    def retry_with_backoff(self, func, max_retries=3):
        """
        Retry function with exponential backoff
        """
        import time

        for attempt in range(max_retries):
            try:
                return func()
            except (NetworkError, TemporaryError) as e:
                if attempt == max_retries - 1:
                    raise

                # Exponential backoff: 1s, 2s, 4s
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)

        raise Exception("Max retries exceeded")
```

---

## Reconciliation and Ledger

```python
class Ledger:
    """
    Double-entry accounting ledger for all financial transactions
    """

    def record_payment(self, from_account, to_account, amount, transaction_id):
        """
        Record payment using double-entry bookkeeping

        Debit customer account, Credit merchant account
        """
        entries = [
            {
                'entry_id': str(uuid.uuid4()),
                'transaction_id': transaction_id,
                'account_id': from_account,
                'type': 'debit',
                'amount': amount,
                'timestamp': datetime.utcnow().isoformat()
            },
            {
                'entry_id': str(uuid.uuid4()),
                'transaction_id': transaction_id,
                'account_id': to_account,
                'type': 'credit',
                'amount': amount,
                'timestamp': datetime.utcnow().isoformat()
            }
        ]

        # Atomic transaction - both entries or neither
        with db.transaction():
            for entry in entries:
                db.insert('ledger_entries', entry)

        return entries

    def get_balance(self, account_id):
        """
        Calculate account balance from ledger
        """
        credits = db.sum('ledger_entries',
                        where={'account_id': account_id, 'type': 'credit'})
        debits = db.sum('ledger_entries',
                       where={'account_id': account_id, 'type': 'debit'})

        return credits - debits

class ReconciliationService:
    """
    Reconcile payments with banks and card networks
    """

    def reconcile_daily_settlements(self, date):
        """
        Reconcile daily settlements

        Compare our records with bank statements
        """
        # Get our records
        our_settlements = db.get_settlements(date=date)

        # Get bank statement
        bank_settlements = bank_api.get_settlements(date=date)

        # Compare
        discrepancies = []

        for our_txn in our_settlements:
            bank_txn = bank_settlements.get(our_txn['id'])

            if not bank_txn:
                discrepancies.append({
                    'type': 'missing_in_bank',
                    'transaction_id': our_txn['id'],
                    'amount': our_txn['amount']
                })
            elif bank_txn['amount'] != our_txn['amount']:
                discrepancies.append({
                    'type': 'amount_mismatch',
                    'transaction_id': our_txn['id'],
                    'our_amount': our_txn['amount'],
                    'bank_amount': bank_txn['amount']
                })

        # Check for transactions in bank but not in our records
        for bank_id, bank_txn in bank_settlements.items():
            if bank_id not in [t['id'] for t in our_settlements]:
                discrepancies.append({
                    'type': 'missing_in_our_records',
                    'transaction_id': bank_id,
                    'amount': bank_txn['amount']
                })

        if discrepancies:
            self._create_reconciliation_report(date, discrepancies)

        return {
            'date': date,
            'total_transactions': len(our_settlements),
            'discrepancies': len(discrepancies),
            'status': 'failed' if discrepancies else 'success'
        }
```

---

## Database Design

```sql
-- Payment Intents
CREATE TABLE payment_intents (
    id VARCHAR(255) PRIMARY KEY,
    merchant_id VARCHAR(255) NOT NULL,
    customer_id VARCHAR(255),
    amount BIGINT NOT NULL,  -- Amount in cents
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(50) NOT NULL,  -- requires_payment_method, requires_capture, succeeded, etc.
    payment_method_token VARCHAR(255),
    authorization_code VARCHAR(255),
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_merchant (merchant_id),
    INDEX idx_status (status)
);

-- Transactions
CREATE TABLE transactions (
    id VARCHAR(255) PRIMARY KEY,
    payment_intent_id VARCHAR(255) REFERENCES payment_intents(id),
    type VARCHAR(50) NOT NULL,  -- authorization, capture, refund
    amount BIGINT NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(50) NOT NULL,
    processor_response JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_payment_intent (payment_intent_id),
    INDEX idx_created_at (created_at)
);

-- Ledger Entries (Double-entry bookkeeping)
CREATE TABLE ledger_entries (
    entry_id VARCHAR(255) PRIMARY KEY,
    transaction_id VARCHAR(255) REFERENCES transactions(id),
    account_id VARCHAR(255) NOT NULL,
    type VARCHAR(10) NOT NULL,  -- debit or credit
    amount BIGINT NOT NULL,
    currency VARCHAR(3) NOT NULL,
    timestamp TIMESTAMP DEFAULT NOW(),
    INDEX idx_account (account_id),
    INDEX idx_transaction (transaction_id)
);

-- Payment Methods (Tokenized)
CREATE TABLE payment_methods (
    token VARCHAR(255) PRIMARY KEY,
    customer_id VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,  -- card, bank_account
    card_last4 VARCHAR(4),
    card_brand VARCHAR(50),
    card_exp_month INT,
    card_exp_year INT,
    billing_address JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_customer (customer_id)
);

-- Merchant Accounts
CREATE TABLE merchants (
    id VARCHAR(255) PRIMARY KEY,
    business_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    bank_account_token VARCHAR(255),
    fee_percentage DECIMAL(5,2) DEFAULT 2.9,  -- Platform fee
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Refunds
CREATE TABLE refunds (
    id VARCHAR(255) PRIMARY KEY,
    transaction_id VARCHAR(255) REFERENCES transactions(id),
    amount BIGINT NOT NULL,
    reason VARCHAR(255),
    status VARCHAR(50) NOT NULL,
    refunded_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_transaction (transaction_id)
);

-- Disputes/Chargebacks
CREATE TABLE disputes (
    id VARCHAR(255) PRIMARY KEY,
    transaction_id VARCHAR(255) REFERENCES transactions(id),
    reason VARCHAR(255) NOT NULL,
    amount BIGINT NOT NULL,
    status VARCHAR(50) NOT NULL,  -- pending, won, lost
    evidence JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    resolved_at TIMESTAMP,
    INDEX idx_transaction (transaction_id),
    INDEX idx_status (status)
);
```

---

## API Design

### RESTful API Endpoints

```
# Payment Intents
POST   /v1/payment_intents
GET    /v1/payment_intents/:id
POST   /v1/payment_intents/:id/confirm
POST   /v1/payment_intents/:id/capture
POST   /v1/payment_intents/:id/cancel

# Refunds
POST   /v1/refunds
GET    /v1/refunds/:id
GET    /v1/refunds?payment_intent=:id

# Payment Methods
POST   /v1/payment_methods
GET    /v1/payment_methods/:id
POST   /v1/payment_methods/:id/attach
POST   /v1/payment_methods/:id/detach

# Customers
POST   /v1/customers
GET    /v1/customers/:id
POST   /v1/customers/:id
DELETE /v1/customers/:id

# Webhooks (Events)
POST   /v1/webhook_endpoints
GET    /v1/events/:id
```

---

## Real-World Examples

### Stripe
- **Architecture**: Microservices on AWS
- **Processing**: 300K+ requests/second
- **Features**: Payment intents, automatic retries, 3D Secure
- **Compliance**: PCI DSS Level 1, SOC 2

### PayPal
- **Scale**: Processes $1+ trillion annually
- **Features**: Buyer/seller protection, dispute resolution
- **Global**: 200+ markets, 100+ currencies

### Square
- **Focus**: In-person + online payments
- **Hardware**: Point-of-sale terminals
- **Features**: Instant transfers, invoicing

---

## Interview Tips

### Common Questions

**Q: How do you prevent double-charging?**
- Idempotency keys for API requests
- Database constraints (unique transaction IDs)
- Distributed locks for critical sections
- Two-phase commit for settlements

**Q: How do you handle payment failures?**
- Automatic retries with exponential backoff
- Webhook notifications to merchant
- Customer notification (email/SMS)
- Grace period for subscription payments

**Q: How do you detect fraud?**
- Rule-based checks (velocity, amount, geolocation)
- ML models (anomaly detection)
- 3D Secure for high-risk transactions
- Device fingerprinting
- Behavior analysis

**Q: How do you ensure consistency?**
- ACID transactions for critical operations
- Eventual consistency for non-critical (analytics)
- Distributed transactions (2PC, Saga pattern)
- Ledger with double-entry bookkeeping

**Q: How do you scale?**
- Horizontal scaling of API servers
- Database sharding by merchant_id
- Async processing with message queues
- CDN for static content
- Rate limiting per merchant

### Key Takeaways

1. **Never store raw card data** - Use tokenization
2. **Idempotency** - Critical for preventing duplicate charges
3. **PCI DSS compliance** - Mandatory for card processing
4. **Ledger system** - Double-entry bookkeeping for reconciliation
5. **Fraud detection** - Multi-layered approach (rules + ML)
6. **Eventual consistency** - For non-critical operations
7. **Webhooks** - Notify merchants of payment events

This payment system design demonstrates financial transaction processing, security, compliance, and scalability - critical for fintech interviews!
