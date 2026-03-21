---
layout: default
title: "API Idempotency Implementation Guide for Distributed Systems"
description: "A guide to implementing API idempotency. Learn how to design idempotent endpoints that safely handle retries, prevent duplicate operations, and build"
date: 2026-03-18
author: theluckystrike
permalink: /a11-api-idempotency-implementation/
categories: [guides]
tags: [remote-work-tools, api-design, distributed-systems, backend-development, reliability, best-practices, api]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}
# API Idempotency Implementation Guide for Distributed Systems

Idempotency is a fundamental concept in API design that ensures the same request can be executed multiple times without producing different results. When implementing distributed systems, network failures, timeouts, and client retries can cause the same operation to be processed accidentally multiple times. Without proper idempotency handling, this leads to duplicate records, double charges, inconsistent state, and frustrated users.

This guide walks you through implementing idempotent APIs that gracefully handle retries while maintaining data integrity.

## Understanding Idempotency

An idempotent operation produces the same result regardless of how many times it's executed. GET, PUT, and DELETE requests should be idempotent by design. POST and PATCH requests are typically non-idempotent but can be made idempotent with proper implementation.

### Idempotent vs Non-Idempotent Operations

```
GET /users/123        → Idempotent (retrieving doesn't change state)
PUT /users/123        → Idempotent (replacing with same data yields same result)
DELETE /users/123     → Idempotent (already deleted, subsequent deletes return 404)
POST /orders          → Non-idempotent (multiple calls = multiple orders)
PATCH /users/123      → Can be idempotent depending on implementation
```

## Idempotency Key Pattern

The most common approach to implementing idempotency uses client-generated unique keys. Here's how it works:

### Implementation Architecture

```javascript
// Client-side: Generate a unique idempotency key
const idempotencyKey = crypto.randomUUID();

// Send request with idempotency key in header
const response = await fetch('/api/orders', {
  method: 'POST',
  headers: {
    'Idempotency-Key': idempotencyKey,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ amount: 99.99, productId: 'widget-123' })
});
```

### Server-Side Implementation

```javascript
// Store idempotency keys with TTL (Time-To-Live)
const idempotencyStore = new Map();

// Middleware to check for duplicate requests
async function idempotencyMiddleware(req, res, next) {
  const key = req.headers['idempotency-key'];
  
  if (!key) {
    return res.status(400).json({ error: 'Idempotency-Key header required' });
  }
  
  // Check if we've seen this key before
  const existing = idempotencyStore.get(key);
  
  if (existing) {
    // Return cached response for duplicate request
    return res.status(existing.status).json(existing.body);
  }
  
  // Store pending status
  idempotencyStore.set(key, { status: 202, body: { status: 'processing' } });
  
  // Attach key to request for controller use
  req.idempotencyKey = key;
  next();
}

// Controller that handles the actual operation
async function createOrder(req, res) {
  const { amount, productId } = req.body;
  const key = req.idempotencyKey;
  
  try {
    // Process the order
    const order = await processOrder({ amount, productId });
    
    // Store successful response
    idempotencyStore.set(key, {
      status: 201,
      body: { orderId: order.id, status: 'completed' }
    });
    
    return res.status(201).json({ orderId: order.id, status: 'completed' });
    
  } catch (error) {
    // Clear the key on failure so client can retry
    idempotencyStore.delete(key);
    throw error;
  }
}
```

## Idempotency with Database Transactions

For operations that modify database state, use transactions combined with idempotency keys:

```javascript
async function createIdempotentOrder(db, idempotencyKey, orderData) {
  // Start transaction
  const client = await db.connect();
  
  try {
    await client.query('BEGIN');
    
    // Check if this idempotency key was already processed
    const existing = await client.query(
      'SELECT * FROM idempotent_requests WHERE key = $1',
      [idempotencyKey]
    );
    
    if (existing.rows.length > 0) {
      // Return cached result
      await client.query('COMMIT');
      return existing.rows[0].response;
    }
    
    // Create the order
    const order = await createOrder(client, orderData);
    
    // Store the idempotency key with response
    await client.query(
      'INSERT INTO idempotent_requests (key, request_hash, response) VALUES ($1, $2, $3)',
      [idempotencyKey, hash(orderData), order]
    );
    
    await client.query('COMMIT');
    return order;
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

## Idempotency for Payment Processing

Payment systems absolutely require idempotency. Here's a pattern:

```javascript
class PaymentIdempotencyService {
  constructor(stripe, redis) {
    this.stripe = stripe;
    this.redis = redis;
    this.TTL = 24 * 60 * 60; // 24 hours
  }
  
  async processPayment(idempotencyKey, customerId, amount, currency) {
    // Check Redis cache first
    const cached = await this.redis.get(`idempotent:${idempotencyKey}`);
    if (cached) {
      return JSON.parse(cached);
    }
    
    try {
      // Attempt the payment
      const payment = await this.stripe.paymentIntents.create({
        amount,
        currency,
        customer: customerId,
        idempotency_key: idempotencyKey
      });
      
      // Cache successful response
      await this.redis.setex(
        `idempotent:${idempotencyKey}`,
        this.TTL,
        JSON.stringify(payment)
      );
      
      return payment;
      
    } catch (error) {
      // Check if it's a duplicate (Stripe returns 409)
      if (error.type === 'IdempotencyError') {
        // Retrieve the original payment intent
        const originalKey = error.originalRequest?.idempotency_key;
        if (originalKey) {
          return await this.stripe.paymentIntents.retrieve(
            error.paymentIntent?.id
          );
        }
      }
      throw error;
    }
  }
}
```

## Handling Idempotency Key Collisions

Prevent intentional key reuse attacks with request hashing:

```javascript
function createRequestHash(method, path, body) {
  const payload = `${method}:${path}:${JSON.stringify(body)}`;
  return crypto.createHash('sha256').update(payload).digest('hex');
}

async function validateIdempotencyKey(req, res, next) {
  const key = req.headers['idempotency-key'];
  const hash = createRequestHash(req.method, req.originalUrl, req.body);
  
  const existing = idempotencyStore.get(key);
  
  if (existing && existing.hash !== hash) {
    // Same key but different request - potential attack
    return res.status(409).json({
      error: 'Idempotency key already used with different request'
    });
  }
  
  next();
}
```

## Best Practices

1. Use appropriate TTL: Store idempotency keys long enough to handle delayed retries (typically 24-48 hours for payments, shorter for other operations).

2. Return proper status codes: Use 201 for successful creation, 200 for successful updates, and return the original response for duplicates.

3. Document idempotent endpoints: Clearly indicate which endpoints support idempotency and how to use the idempotency key.

4. Implement at the API gateway level: Consider implementing idempotency handling at the gateway to protect all backend services.

5. Handle partial failures: If a request succeeds but the response fails to deliver, the client retries - ensure your system handles this gracefully.

## Testing Idempotency

Write tests that verify duplicate requests return the same response:

```javascript
describe('Idempotent Order Creation', () => {
  it('should return same order for duplicate requests', async () => {
    const idempotencyKey = 'test-key-' + Date.now();
    const orderData = { amount: 50, productId: 'test-123' };
    
    // First request
    const response1 = await request(app)
      .post('/api/orders')
      .set('Idempotency-Key', idempotencyKey)
      .send(orderData);
    
    // Duplicate request
    const response2 = await request(app)
      .post('/api/orders')
      .set('Idempotency-Key', idempotencyKey)
      .send(orderData);
    
    expect(response1.status).toEqual(response2.status);
    expect(response1.body.orderId).toEqual(response2.body.orderId);
  });
  
  it('should create separate orders for different idempotency keys', async () => {
    const orderData = { amount: 50, productId: 'test-123' };
    
    const response1 = await request(app)
      .post('/api/orders')
      .set('Idempotency-Key', 'key-1')
      .send(orderData);
    
    const response2 = await request(app)
      .post('/api/orders')
      .set('Idempotency-Key', 'key-2')
      .send(orderData);
    
    expect(response1.body.orderId).not.toEqual(response2.body.orderId);
  });
});
```




## Idempotency in Distributed Systems and Microservices

In microservice architectures, a single user-facing operation often triggers multiple internal service calls. Idempotency must be implemented at each service boundary, not just at the entry point.

Consider a payment flow that calls three internal services: billing, inventory, and notifications. If the payment service receives a duplicate request, all three downstream services should also receive the same idempotency key to prevent double-billing, double-inventory-deduction, and duplicate notification emails.

Pass idempotency keys through service boundaries explicitly:

```javascript
// payment-service/handlers/processPayment.js
async function processPayment(idempotencyKey, paymentData) {
  // Each downstream call uses the same idempotency key
  const billingResult = await billingService.charge({
    idempotencyKey: `billing:${idempotencyKey}`,
    ...paymentData
  });

  const inventoryResult = await inventoryService.reserve({
    idempotencyKey: `inventory:${idempotencyKey}`,
    ...paymentData
  });

  const notificationResult = await notificationService.send({
    idempotencyKey: `notification:${idempotencyKey}`,
    email: paymentData.customerEmail,
    template: 'payment_confirmed'
  });

  return { billing: billingResult, inventory: inventoryResult };
}
```

Prefix the key at each service layer (`billing:`, `inventory:`) to prevent key collisions across service namespaces. Each service independently checks and stores the prefixed key in its own idempotency store.


## Choosing Your Idempotency Storage Backend

The in-memory Map used in the examples above works for single-server deployments but breaks in horizontally scaled systems. Production implementations need a shared storage backend:

**Redis** is the standard choice for idempotency key storage. It supports atomic operations, built-in TTL, and handles high throughput:

```javascript
const redis = require('redis');
const client = redis.createClient({ url: process.env.REDIS_URL });

class RedisIdempotencyStore {
  constructor(ttlSeconds = 86400) {  // 24 hours default
    this.ttlSeconds = ttlSeconds;
  }

  async get(key) {
    const value = await client.get(`idempotent:${key}`);
    return value ? JSON.parse(value) : null;
  }

  async setIfAbsent(key, value) {
    // NX option: only set if key doesn't exist (atomic)
    const result = await client.set(
      `idempotent:${key}`,
      JSON.stringify(value),
      { NX: true, EX: this.ttlSeconds }
    );
    return result === 'OK'; // true = key was set (new), false = key existed
  }
}
```

The `NX` (Not eXists) option makes the Redis SET atomic — it either sets the key if absent and returns `OK`, or returns `null` if the key already existed. This eliminates the race condition where two concurrent duplicate requests both see the key as absent and both create the resource.

**PostgreSQL** works when you are already using it and want to avoid adding Redis:

```sql
-- idempotent_requests table
CREATE TABLE idempotent_requests (
  key VARCHAR(255) PRIMARY KEY,
  request_hash VARCHAR(64) NOT NULL,
  response JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL
);

-- Automatic cleanup of expired keys
CREATE INDEX idx_idempotent_expires_at ON idempotent_requests (expires_at);
```

Use PostgreSQL's `INSERT... ON CONFLICT DO NOTHING` for atomic upsert behavior similar to Redis NX.


## Idempotency Key Generation on the Client

Client-side key generation strategies significantly affect your system's safety properties. Poorly generated keys cause either unintended duplicates (too short, possible collision) or unnecessary uniqueness (new key per retry, defeating the purpose).

The best strategy: generate the key when the user initiates an action, not when the request is sent. Store the key in memory for the duration of the operation. Only generate a new key if the user explicitly starts a new transaction:

```javascript
class PaymentForm {
  constructor() {
    // Key generated once when form is loaded
    this.idempotencyKey = crypto.randomUUID();
  }

  async submitPayment(paymentData) {
    // All retries use the same key
    for (let attempt = 0; attempt < 3; attempt++) {
      try {
        return await apiClient.post('/payments', paymentData, {
          headers: { 'Idempotency-Key': this.idempotencyKey }
        });
      } catch (error) {
        if (!isRetryable(error) || attempt === 2) throw error;
        await sleep(Math.pow(2, attempt) * 1000);
      }
    }
  }

  resetForm() {
    // Only regenerate key when user explicitly cancels and starts over
    this.idempotencyKey = crypto.randomUUID();
  }
}
```

This pattern ensures that button-spam and network retries all use the same idempotency key, while explicit user actions (clicking "cancel" and starting over) generate a fresh key.

## Related Reading

- [Best Remote Work Tools in 2026](/best-remote-work-tools-2026/)
- [Remote Work Productivity Guide](/remote-work-productivity-guide/)
- [Remote Work Tools Hub](/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
