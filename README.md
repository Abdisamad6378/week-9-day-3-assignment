# Week 9 Day 3: M-Pesa Callback Handling

After an STK Push, the customer enters their PIN (or cancels) and Safaricom sends the result to your **callback URL**. This project handles that callback properly:

- **Callback handler**: answers Safaricom with 200 right away, then reads the nested payload and marks the payment `completed`, `cancelled` or `failed` in Postgres (through Prisma)
- **Idempotency**: the same callback arriving twice (Safaricom retries) only updates the database once
- **Status query**: `POST /api/mpesa/query` answers from the database if the payment is already settled, and otherwise asks Daraja's STK Push Query API
- **Callback simulator**: scripts to fake a pending payment and a Safaricom callback, so you can test without a real payment or ngrok
- **20 tests** against the real database, with Daraja faked

It builds on Day 1 (cached OAuth token) and Day 2 (phone formatting, STK Push). Payments are now stored in Postgres instead of an in-memory array.

## Getting started

You need Node.js 18+, PostgreSQL and Daraja sandbox keys.

```bash
git clone https://github.com/Abdisamad6378/week-9-day-3-assignment.git
cd week-9-day-3-assignment
npm install                 # also runs prisma generate
cp .env.example .env        # fill in your keys and DATABASE_URL
npx prisma migrate dev      # creates the database and the Payment table
npm run dev                 # or: npm start
npm test
```

`.env`:

```
MPESA_CONSUMER_KEY=your_consumer_key_here
MPESA_CONSUMER_SECRET=your_consumer_secret_here
MPESA_SHORTCODE=174379
MPESA_PASSKEY=bfb279f9aa9bdbcf158e97dd71a467cd2e0c893059b10f78e6b72ada1ed2c919
MPESA_ENV=sandbox
PORT=3000
MPESA_CALLBACK_URL=https://your-ngrok-url.ngrok-free.app/api/mpesa/callback
DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/week9day3"
```

This uses Prisma 7, which connects to Postgres through the `@prisma/adapter-pg` driver adapter (see `db.js`), and keeps the database URL in `prisma.config.js` instead of `schema.prisma`.

## Project structure

```
week-9-day-3-assignment/
├── prisma/
│   ├── schema.prisma            Payment model
│   └── migrations/              init migration
├── prisma.config.js             Prisma 7 config (schema path, DATABASE_URL)
├── db.js                        Shared Prisma client
├── config.js                    .env validation, sandbox/production URL
├── phone.js                     Kenyan phone formatting (Day 2)
├── mpesa.js                     OAuth token, password, initiateSTKPush, querySTKPushStatus
├── payments.js                  getCallbackValue, processCallback, idempotent updates
├── server.js                    Express routes
├── scripts/
│   ├── create-test-payment.js   Makes a pending payment to test against
│   └── simulate-callback.js     Sends a fake Safaricom callback
└── test.js                      20 tests
```

## The full payment flow

```
1. POST /api/mpesa/stkpush            → Payment saved as "pending"
2. Customer enters PIN / cancels
3. Safaricom → POST /api/mpesa/callback
       ├─ reply 200 { ResultCode: 0, ResultDesc: "Accepted" }   (immediately)
       ├─ find payment by CheckoutRequestID
       │     not found        → warn, stop
       │     not "pending"    → duplicate, stop
       └─ ResultCode 0        → "completed" + receipt, date, phone, amount
          ResultCode 1032     → "cancelled"
          anything else       → "failed"
4. POST /api/mpesa/query               → status from DB, or from Daraja if still pending
```

## Task 1: Callback handler

`POST /api/mpesa/callback` in `server.js`, with the logic in `payments.js`.

**Respond first, process second.** The route sends `{ ResultCode: 0, ResultDesc: "Accepted" }` before touching the database. If Safaricom doesn't get a 200 quickly, it sends the callback again, so a slow database or a crash must not stop the 200 from going out. Any error after that is only logged.

What Safaricom sends for a successful payment:

```json
{
  "Body": {
    "stkCallback": {
      "MerchantRequestID": "29115-34620561-1",
      "CheckoutRequestID": "ws_CO_TEST_001",
      "ResultCode": 0,
      "ResultDesc": "The service request is processed successfully.",
      "CallbackMetadata": {
        "Item": [
          { "Name": "Amount", "Value": 100 },
          { "Name": "MpesaReceiptNumber", "Value": "SHK4G7Y8DR" },
          { "Name": "TransactionDate", "Value": 20260504143055 },
          { "Name": "PhoneNumber", "Value": 254712345678 }
        ]
      }
    }
  }
}
```

The details sit in a list of `{ Name, Value }` pairs rather than plain fields, so `getCallbackValue(metadata, name)` searches the list and returns the value, or `null` if it isn't there. Failed and cancelled callbacks have no `CallbackMetadata` at all, and the helper handles that too.

`TransactionDate` and `PhoneNumber` come as numbers but are stored as strings, which matches the schema. The phone number is a 12-digit number, and you never do maths on it.

| ResultCode | Status | Example ResultDesc |
|------------|--------|--------------------|
| `0` | `completed` | The service request is processed successfully. |
| `1032` | `cancelled` | Request cancelled by user. |
| anything else (`1`, `1037`, `2001`, ...) | `failed` | The balance is insufficient / DS timeout user cannot be reached / wrong PIN |

Console output:

```
Payment ws_CO_TEST_001 completed. Receipt: SHK4G7Y8DR
Payment ws_CO_TEST_002 cancelled: Request cancelled by user.
```

## Task 2: Idempotency

Before updating, `processCallback()`:

1. Looks up the payment by `checkoutRequestId`. If there is none, it logs `Callback for unknown CheckoutRequestID: ...` and stops. It never creates a payment from a callback.
2. If the status is not `pending`, it logs `Duplicate callback for ws_CO_TEST_001. Already completed. Skipping.` and stops.
3. Otherwise it updates.

The update itself also checks the status:

```js
prisma.payment.updateMany({
  where: { checkoutRequestId, status: 'pending' },
  data,
});
```

Steps 1-2 alone leave a small gap. If two copies of a callback arrive at the same moment, both can read `pending` before either one writes. Putting `status: 'pending'` in the `WHERE` makes the database do the check and the write in one step, so only one of them changes a row (`count === 1`). The other gets `count === 0` and is treated as a duplicate. There's a test that fires two callbacks at the same time to prove it.

This also means a late or retried callback can never change a finished payment. A `cancelled` callback arriving after `completed` is skipped.

Try it with the assignment's curl test:

```bash
npm run test-payment                # creates pending ws_CO_TEST_001 (or resets it)
# then send the success payload from Task 1 twice:
curl -X POST http://localhost:3000/api/mpesa/callback -H "Content-Type: application/json" -d '{ ...payload... }'
curl -X POST http://localhost:3000/api/mpesa/callback -H "Content-Type: application/json" -d '{ ...payload... }'
```

```
Payment ws_CO_TEST_001 completed. Receipt: SHK4G7Y8DR
Duplicate callback for ws_CO_TEST_001. Already completed. Skipping.
```

Or use the simulator, which sends the same payload:

```bash
npm run simulate -- ws_CO_TEST_001 success
npm run simulate -- ws_CO_TEST_001 success     # skipped
npm run test-payment -- ws_CO_TEST_002
npm run simulate -- ws_CO_TEST_002 cancelled
```

## Task 3: `POST /api/mpesa/query`

```bash
curl -X POST http://localhost:3000/api/mpesa/query \
  -H "Content-Type: application/json" \
  -d '{"checkoutRequestId": "ws_CO_TEST_001"}'
```

1. No `checkoutRequestId` in the body → 400.
2. Not in the database → **404** `{ "success": false, "error": "Payment not found" }`.
3. Already `completed` / `failed` / `cancelled` → return it straight from the database (`"source": "database"`). Daraja is not called.
4. Still `pending` → call `POST {baseUrl}/mpesa/stkpushquery/v1/query` with `BusinessShortCode`, `Password`, `Timestamp` and `CheckoutRequestID`. Then map `ResultCode` `"0"` → completed, `"1032"` → cancelled, anything else → failed, save it, and return it (`"source": "daraja"`).
5. If the query fails (Daraja unreachable, timeout, rate limit, or Daraja's own "The transaction is being processed" while the customer hasn't answered yet), return the payment as `pending` with `"message": "Payment is still being processed. Try again in a few seconds."`

Completed:

```json
{
  "success": true,
  "data": {
    "checkoutRequestId": "ws_CO_TEST_001",
    "status": "completed",
    "resultCode": 0,
    "resultDesc": "The service request is processed successfully.",
    "mpesaReceiptNumber": "SHK4G7Y8DR",
    "amount": 100,
    "phoneNumber": "254712345678",
    "source": "database"
  }
}
```

Still pending:

```json
{
  "success": true,
  "data": {
    "checkoutRequestId": "ws_CO_...",
    "status": "pending",
    "resultCode": null,
    "resultDesc": null,
    "mpesaReceiptNumber": null,
    "amount": 1,
    "phoneNumber": "254712345678",
    "message": "Payment is still being processed. Try again in a few seconds."
  }
}
```

The query result is saved with the same `status: 'pending'` rule as the callback, so if the real callback arrives while we're waiting on Daraja, the callback's data (which includes the receipt number) is kept. The query API doesn't return a receipt number, which is one reason the callback is the main source of truth and the query is the backup.

## Other endpoints

| Endpoint | What it does |
|----------|--------------|
| `GET /` | Lists the endpoints |
| `GET /api/health` | Config and database check |
| `POST /api/mpesa/stkpush` | Day 2 STK Push, now saving to Postgres |
| `GET /api/payments` | All payments, newest first |
| `GET /api/payments/:checkoutRequestId` | One payment |

## Testing with a real callback (ngrok)

Safaricom can't reach `localhost`, so you need a public https URL:

```bash
ngrok http 3000
```

Copy the `https://....ngrok-free.app` URL into `.env` as `MPESA_CALLBACK_URL=https://....ngrok-free.app/api/mpesa/callback`, restart the server, and send an STK Push:

```bash
curl -X POST http://localhost:3000/api/mpesa/stkpush \
  -H "Content-Type: application/json" \
  -d '{"phoneNumber": "07XXXXXXXX", "amount": 1}'
```

Accept or cancel on the phone, and the callback shows up in the server log a few seconds later. The ngrok dashboard at http://localhost:4040 shows the raw request Safaricom sent.

## Automated tests

`npm test` runs 20 tests with `node:test`. They use the real Postgres database, but only with IDs starting with `ws_CO_UNITTEST_`, which are deleted before and after. Daraja is faked by replacing `axios.get` and `axios.post`, and our own API is called with `fetch`.

- Helpers: `getCallbackValue()` found and missing, `statusFromResultCode()` with numbers and strings
- Task 1: immediate 200 even for a bad body; completed with all four metadata fields saved; cancelled (1032); failed; through the real endpoint
- Task 2: same callback twice, where the second is skipped, the receipt is unchanged and `updatedAt` is unchanged; a late cancel can't overwrite completed; two callbacks at once, where exactly one wins; twice through the endpoint; unknown ID creates nothing
- Task 3: 404, 400, settled payment answered without calling Daraja, the query payload and password, 1032 → cancelled, 0 → completed, 1037 → failed, network error → pending with message, Daraja "being processed" → pending

```
# tests 20
# pass 20
# fail 0
```

## Tech used

Node.js, Express 5, Prisma 7 + PostgreSQL, axios, dotenv, cors, nodemon, ngrok.
