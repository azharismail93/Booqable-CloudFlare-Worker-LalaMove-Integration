# BubbaRent Logistics Automation

Cloudflare Worker that automates BubbaRent delivery logistics by connecting **Booqable**, **OneMap**, and **Lalamove**.

The Worker receives a Booqable order webhook, extracts the customer's delivery information, converts the Singapore postcode into geographic coordinates using OneMap, creates a Lalamove quotation, and then creates the corresponding Lalamove delivery order.

## Architecture

```text
Booqable
    │
    │ order.reserved webhook
    ▼
Cloudflare Worker
    │
    ├── Extract booking information
    │
    ├── Extract Singapore postcode
    │
    ▼
OneMap API
    │
    │ latitude / longitude
    ▼
Cloudflare Worker
    │
    ├── Calculate Lalamove schedule time
    │
    ├── Create Lalamove quotation
    │
    ▼
Lalamove API
    │
    │ quotationId + stop IDs
    ▼
Cloudflare Worker
    │
    └── Create Lalamove order
```

## Features

- Receives Booqable `order.reserved` webhooks
- Extracts:
  - `starts_at`
  - `delivery_address`
- Automatically extracts the Singapore 6-digit postcode
- Uses OneMap to geocode the postcode
- Calculates the Lalamove scheduled pickup time
- Creates a Lalamove quotation
- Uses the stop IDs returned by Lalamove
- Creates the Lalamove order automatically
- Uses HMAC-SHA256 authentication for Lalamove
- Runs entirely on Cloudflare Workers
- No VPS or dedicated server required
- API credentials are stored using Cloudflare Worker secrets

## Requirements

You will need:

- A Booqable account with webhook support
- A OneMap account/API credentials
- A Lalamove API account
- A Cloudflare account
- A Cloudflare Worker

## Environment Variables

The following values should be configured as **Cloudflare Worker secrets or environment variables**.

| Variable | Description |
|---|---|
| `ONEMAP_EMAIL` | OneMap account email |
| `ONEMAP_PASSWORD` | OneMap account password |
| `LALAMOVE_API_KEY` | Lalamove API key |
| `LALAMOVE_API_SECRET` | Lalamove API secret |
| `BUBBARENT_PICKUP_LAT` | BubbaRent pickup latitude |
| `BUBBARENT_PICKUP_LNG` | BubbaRent pickup longitude |
| `BUBBARENT_PICKUP_ADDRESS` | BubbaRent pickup address |

**Do not commit these values to GitHub.**

Example:

```text
ONEMAP_EMAIL=<your-email>
ONEMAP_PASSWORD=<your-password>
LALAMOVE_API_KEY=<your-api-key>
LALAMOVE_API_SECRET=<your-api-secret>

BUBBARENT_PICKUP_LAT=<pickup-latitude>
BUBBARENT_PICKUP_LNG=<pickup-longitude>
BUBBARENT_PICKUP_ADDRESS=<pickup-address>
```

Use placeholder values only in documentation.

## Booqable Webhook

The Worker is designed to process:

```text
order.reserved
```

Other Booqable events are ignored.

The Worker reads:

```javascript
webhook.data.starts_at
webhook.data.delivery_address
```

A delivery address such as:

```text
111 Singapore Central
11-1234
Singapore 123456
```

will result in:

```text
123456
```

being extracted as the postcode.

## OneMap

The Singapore postcode is sent to OneMap to obtain geographic coordinates.

The Worker:

1. Authenticates with OneMap
2. Obtains an access token
3. Searches the postcode
4. Extracts the first matching result
5. Retrieves latitude and longitude

Example:

```text
Postcode: 123556

Latitude: 55.xxxxx
Longitude: 66.xxxxx
```

## Lalamove Integration

The Worker uses the Lalamove API in two stages.

### 1. Create quotation

A request is sent to:

```text
POST /v3/quotations
```

The request contains:

- Scheduled pickup time
- Service type
- Language
- Pickup location
- Customer delivery location
- Item information

The pickup location is configured using the BubbaRent environment variables.

The customer location comes from OneMap.

### 2. Create order

After receiving a successful quotation, the Worker extracts:

```text
quotationId
```

and the `stopId` values returned by Lalamove.

The returned stop IDs should be used directly rather than attempting to calculate them from the quotation ID.

The order is then submitted to:

```text
POST /v3/orders
```

This creates the actual Lalamove delivery order.

## Schedule Time

Booqable provides the rental start time through:

```text
starts_at
```

The Worker currently schedules the Lalamove delivery **4 hours before the Booqable start time**.

For example:

```text
Booqable:
2026-10-22T12:00:00.000000+00:00

Lalamove:
2026-10-22T08:00:00.000Z
```

This offset can be changed in the Worker code if the logistics process changes.

## Lalamove Authentication

Lalamove requests are authenticated using HMAC-SHA256.

The signature is generated from:

```text
<TIMESTAMP>
<HTTP METHOD>
<PATH>

<BODY>
```

with the Lalamove API secret.

The resulting authorization header follows the Lalamove API format:

```text
hmac <API_KEY>:<TIMESTAMP>:<SIGNATURE>
```

The API secret is never stored in the source code.

## Cloudflare Worker Setup

### 1. Create the Worker

Create a new Cloudflare Worker.

Copy the Worker source code into the Worker editor or deploy it using Wrangler.

### 2. Configure secrets

Using Wrangler:

```bash
wrangler secret put ONEMAP_EMAIL
wrangler secret put ONEMAP_PASSWORD
wrangler secret put LALAMOVE_API_KEY
wrangler secret put LALAMOVE_API_SECRET
```

Configure the BubbaRent pickup variables as Worker environment variables or secrets according to your deployment configuration.

### 3. Deploy

```bash
wrangler deploy
```

Cloudflare will provide a Worker URL similar to:

```text
https://your-worker.your-subdomain.workers.dev
```

### 4. Configure Booqable

Configure a Booqable webhook pointing to the Worker URL.

The Worker expects:

```text
POST
```

requests containing the Booqable webhook JSON payload.

## Testing

A test webhook can be sent to the Worker using:

```bash
curl -X POST "https://your-worker.example.workers.dev" \
  -H "Content-Type: application/json" \
  -d '{
    "event": "order.reserved",
    "data": {
      "starts_at": "2026-10-22T12:00:00.000000+00:00",
      "delivery_address": "111 Singapore Central\n11-123\nSingapore 123456"
    }
  }'
```

Replace the Worker URL with your actual endpoint.

## Error Handling

The Worker returns an error response when:

- The request method is not `POST`
- The webhook contains invalid JSON
- The Booqable event is missing
- `starts_at` is missing
- `delivery_address` is missing
- A Singapore postcode cannot be extracted
- OneMap authentication fails
- OneMap cannot find coordinates
- Lalamove quotation creation fails
- Lalamove does not return a quotation ID
- Lalamove order creation fails

HTTP status codes are used to indicate the general type of failure.

## Security

### Never commit secrets

Do not place any of the following directly in the source code:

```text
ONEMAP_EMAIL
ONEMAP_PASSWORD
LALAMOVE_API_KEY
LALAMOVE_API_SECRET
```

Use Cloudflare Worker secrets instead.

### Recommended `.gitignore`

```gitignore
.dev.vars
.wrangler/
node_modules/
.env
.env.*
```

If using Wrangler locally, `.dev.vars` can contain development secrets and should **never be committed**.

## Project Structure

A minimal project can look like:

```text
bubbarent-logistics/
│
├── src/
│   └── index.js
│
├── .gitignore
├── README.md
├── package.json
└── wrangler.toml
```

## API Flow

```text
1. Booqable
   └── order.reserved

2. Cloudflare Worker
   └── Extract starts_at and delivery_address

3. Postcode extraction
   └── Extract 6-digit Singapore postcode

4. OneMap
   └── Postcode → latitude / longitude

5. Cloudflare Worker
   └── Build Lalamove quotation request

6. Lalamove
   └── POST /v3/quotations

7. Lalamove
   └── Return quotationId + stop IDs

8. Cloudflare Worker
   └── Build order request using returned IDs

9. Lalamove
   └── POST /v3/orders

10. Delivery order created
```

## Disclaimer

This project is intended for BubbaRent's internal logistics automation.

API availability, authentication requirements, request formats, pricing, and other behavior are controlled by the respective third-party services and may change over time.

Always refer to the current API documentation for:

- Booqable
- OneMap
- Lalamove
- Cloudflare Workers
