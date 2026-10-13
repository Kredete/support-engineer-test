# Support & Integration Engineer — Technical Exercise

Welcome, and thanks for taking the time.

This is a **live, screen-shared problem-solving session**. You will not be asked to write an application from scratch.
We want to see how you think: talk us through your reasoning out loud, say what you would check and why, and ask
questions whenever something is unclear. There is no single "right" answer to most of these.

All names, references, amounts and companies below are fictional.

---

## Part 1 — Triage: "I was debited but the money never arrived"

### 1.1 The customer's message

> Hi, I sent ₦250,000 to my mum about two hours ago. My balance went down but she has not received anything.
> This is urgent, she needs it for hospital bills. Please help. — Adaeze

### 1.2 The wallet transaction record

```json
{
  "reference": "TRX-8F3K29QZ",
  "userId": "64f1a2b3c4d5e6f708192a3b",
  "type": "DEBIT",
  "category": "BANK_TRANSFER",
  "amount": 250000,
  "fee": 50,
  "currency": "NGN",
  "status": "PENDING",
  "recipient": {
    "accountName": "NGOZI OKAFOR",
    "accountNumber": "01******89",
    "bankName": "Example Bank"
  },
  "metadata": {
    "provider": "payment_provider_a",
    "providerReference": "pp_tx_77120ab9"
  },
  "createdAt": "2026-10-13T10:02:11Z",
  "updatedAt": "2026-10-13T10:02:14Z"
}
```

### 1.3 The outbound API call log

```text
2026-10-13T10:02:12Z  POST https://api.payment-provider.example/v1/transfers
request:  { "reference": "TRX-8F3K29QZ", "amount": 25000000, "currency": "NGN",
            "destination": { "accountNumber": "0123456789", "bankCode": "999" } }
response: HTTP 200
          { "success": true,
            "data": { "id": "pp_tx_77120ab9", "reference": "TRX-8F3K29QZ", "status": "processing" } }
duration: 2140ms
```

### 1.4 The webhook log

```text
2026-10-13T10:41:57Z  POST /webhooks/payment-provider
headers:  x-signature: 9b1e0c...f42a
body:     { "event": "transfer.failed",
            "data": { "id": "pp_tx_77120ab9", "reference": "TRX-8F3K29QZ",
                      "status": "failed", "reason": "Beneficiary bank unavailable" } }
result:   401 — signature verification failed
```

### Questions

1. Walk us through what you think happened to this transfer, step by step.
2. What do you know for certain, and what is still unknown?
3. What would you check next, and in what order?
4. Can the customer be refunded right now? Why or why not?
5. What would you reply to Adaeze **right now**, before the investigation is finished?

---

## Part 2 — Integration thinking

Below is an excerpt from a (fictional) payout partner's API documentation.

> **POST /v1/transfers** — Initiate a transfer.
> `reference` (string, required) — your unique reference. Reusing a reference returns the original transfer.
> Returns `200` with `status` one of `processing`, `successful`, `failed`.
>
> **GET /v1/transfers/{reference}** — Fetch the current state of a transfer.
>
> **Webhooks** — Sent for `transfer.processing`, `transfer.successful`, `transfer.failed`.
> Each request carries an `x-signature` header: HMAC-SHA256 of the raw body using your webhook secret.
> Webhooks are retried up to 5 times with exponential backoff if you do not respond `2xx`.
>
> **Errors** — Some validation errors are returned as `200` with `"success": false` and an `error` object.
>
> **Authentication** — `Authorization: Bearer <secret_key>`. Sandbox and live use different keys and base URLs.

### Questions

1. Our `POST /v1/transfers` call **times out** after 30 seconds with no response. Do we retry? With the same
   reference or a new one? What could go wrong either way?
2. The same `transfer.successful` webhook arrives **twice**. What should our system do?
3. A `transfer.failed` webhook arrives **before** `transfer.processing`. What should our system do?
4. The partner returns `HTTP 200` with `"success": false`. How would you make sure that is never treated as a success?
5. Write the `curl` command you would use to check the current status of `TRX-8F3K29QZ`.
6. Going back to Part 1: the webhook failed signature verification. List the possible reasons.

---

## Part 3 — Data

Transactions are stored in a collection/table called `walletTransactions` with (at least) these fields:

| Field | Type |
|---|---|
| `reference` | string |
| `status` | `PENDING` \| `SUCCESSFUL` \| `FAILED` \| `REVERSED` |
| `category` | string |
| `amount` | number |
| `currency` | string |
| `metadata.provider` | string |
| `createdAt` | datetime |

### Questions

1. Write a query (MongoDB or SQL — your choice) that returns every transaction that has been `PENDING` for **more than
   30 minutes**, grouped by provider, with the count and total amount per provider.
2. This collection holds 40 million rows. What would you check before running your query in production?

---

## Part 4 — Communication

### Questions

1. Write the reply you would send to Adaeze (from Part 1) once you have confirmed the transfer failed and a refund is
   on its way.
2. Write the message you would post in the engineering channel to escalate the webhook problem you found in Part 1.

---

## Part 5 — Reconciliation

The table below compares the payout partner's settlement report for one day with our own transaction records.

| # | Our reference | Our status | Our amount (₦) | Partner status | Partner amount (₦) |
|---|---|---|---|---|---|
| 1 | TRX-A1001 | SUCCESSFUL | 50,000 | successful | 50,000 |
| 2 | TRX-A1002 | SUCCESSFUL | 120,000 | successful | 120,000 |
| 3 | TRX-A1003 | FAILED | 75,000 | successful | 75,000 |
| 4 | TRX-A1004 | SUCCESSFUL | 10,000 | successful | 10,000 |
| 5 | TRX-A1005 | SUCCESSFUL | 30,000 | successful | 30,000 |
| 6 | TRX-A1005 | — | — | successful | 30,000 |
| 7 | TRX-A1006 | PENDING | 200,000 | failed | 200,000 |
| 8 | TRX-A1007 | SUCCESSFUL | 15,000 | successful | 15,000 |
| 9 | TRX-A1008 | SUCCESSFUL | 45,000 | successful | 44,950 |
| 10 | TRX-A1009 | SUCCESSFUL | 5,000 | successful | 5,000 |
| 11 | — | — | — | successful | 18,000 |
| 12 | TRX-A1010 | REVERSED | 60,000 | failed | 60,000 |

### Questions

1. Which rows do not reconcile?
2. For each one, what do you think happened, and what would you do about it?
3. Which of these would you treat as most urgent, and why?

---

## Part 6 — Partner onboarding (role-play)

One of us will play an engineer at a business customer who is integrating with our API. They will describe problems
such as:

- "Your webhooks never reach us."
- "We get `401 Unauthorized` on every call."
- "Everything worked in sandbox, but nothing works in live."

### Questions

1. Ask the questions you would ask to diagnose each problem.
2. Tell them what you would like them to try or send you.

---

## Part 7 — Reading unfamiliar code

Here is a webhook handler. You do not need to run it.

```typescript
@Post('webhooks/payment-provider')
async handleWebhook(@Body() body: any, @Res() res: Response) {
  const { event, data } = body;

  const trx = await this.transactions.findOne({ reference: data.reference });

  if (!trx) {
    return res.status(404).send('not found');
  }

  if (event === 'transfer.successful') {
    await this.transactions.updateOne(
      { reference: data.reference },
      { $set: { status: 'SUCCESSFUL' } },
    );
  }

  if (event === 'transfer.failed') {
    await this.transactions.updateOne(
      { reference: data.reference },
      { $set: { status: 'FAILED' } },
    );
    await this.wallets.refund(trx.userId, trx.amount + trx.fee);
  }

  return res.status(200).send('ok');
}
```

### Questions

1. What problems do you see in this handler?
2. For each problem, describe a real situation in which it would cause harm.
3. Which one would you raise first, and why?

---

## Part 8 — Prioritisation

These five tickets arrive at the same time:

1. A customer cannot log in after changing phone.
2. A high-value customer's ₦5,000,000 transfer has been stuck for three hours.
3. Forty customers report that card top-ups are failing in the last 20 minutes.
4. A business partner reports that our webhooks are arriving 15 minutes late.
5. A customer reports a typo in our "transfer successful" email.

### Questions

1. In what order would you handle them, and why?
2. Who, if anyone, would you pull in for each one?

---

## Part 9 — Account access and compliance

> "Your system blocked my account for no reason. I need to withdraw my money today, I'm travelling tomorrow."

The account shows it was restricted by an automated identity-verification check two days ago.

### Questions

1. What would you do, and what would you **not** do?
2. What would you tell the customer?

---

## Part 10 — Your experience

1. Tell us about the worst production incident you have handled. What did you do in the first 10 minutes?
2. Tell us about a time you pushed back on an engineer or an external partner. How did it go?
3. How do you decide when to escalate and when to keep digging yourself?

---

Thank you! Please share any questions you have for us at the end.
