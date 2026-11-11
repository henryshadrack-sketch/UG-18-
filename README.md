# UG18

UG18 is a responsive, adult-only dating and social-gathering web app with a
React frontend and an Express API backed by SQLite.

## Run locally

Requirements: Node.js 22.13 or newer.

```powershell
npm install
npm run dev
```

Open the Vite URL printed in the terminal (normally `http://localhost:5173`).
The frontend proxies `/api` requests to the local API at `http://localhost:4100`.
Run `npm run build` for a type-checked production frontend build; set
`$env:NODE_ENV = "production"` in PowerShell and run `npm start` to serve the
built app and API from the same process. The API and frontend run together
during development; keep the terminal open while using the app.

SQLite data is stored in `data/ug18.sqlite` by default. Set `UG18_DATA_DIR` to
the path of a persistent mounted directory on the host; the database and event
photos are both stored there. Pending events and their submitted payment
references are saved privately while they await verification. Accounts are
stored as soon as registration succeeds; email verification is not currently
required. The browser does not store passwords. User contact details and
password hashes are not exposed in public profiles. Passwords are hashed with
Node's built-in scrypt, sessions use random opaque tokens stored as hashes, and
the session cookie is HttpOnly and SameSite=Lax. Production deployments must
use HTTPS so the session cookie is Secure.

## Current functionality

- Adult-only (18+) registration and login with email or international phone,
  country, city, bio, and interests.
- Public member discovery, private direct messages, gatherings and RSVPs,
  premium-plan information, and member feedback.
- Event hosts can submit a photo, category, location, date/time, public contact,
  and description. New listings remain private until the UGX 34,000 publishing
  fee and any optional promotion amount are verified. Public event listings
  appear on the home page and are ordered by verified amount paid, highest first.
- The transparent UG18 logo in `public/ug18-logo.png` and an animated welcome
  intro that respects reduced-motion preferences.
- A mobile-installable PWA with a small static offline shell. Authenticated API
  responses are never cached by the service worker.
- A “Get the app” install guide for the PWA. There is no Android APK yet; the
  app is currently a web app that can be installed from supported browsers.
- Mobile and desktop layouts with Airbnb stays first and hookups (18+) last,
  alongside dating, house parties, trips, pool parties, bonfires, brunch,
  hiking, game nights, and live music. “Airbnb stays” is a community event
  category, not an Airbnb integration or partnership.

## Payments

Event publishing currently supports **manual payment and operator verification**,
not automatic provider checkout. Premium membership payments are not enabled.
The event payment screen shows these destinations and instructions:

- MTN Mobile Money merchant code: `39610456`. In MTN MoMo, choose the merchant
  payment option, enter the code and exact displayed total, and verify the
  merchant name before authorizing.
- Airtel Money merchant code: `6516828`. In Airtel Money, choose Pay Merchant /
  Make Payment, enter the code and exact total, and verify the merchant name.
- PayPal recipient: `henrykizza@gmail.com`.
- Payoneer recipient: `henrykizza@gmail.com`.

Hosts submit their transaction reference from **My event listings**. An
authorized operator must confirm the provider, reference, and exact UGX amount
in the provider's merchant account before the event is published. Higher
verified promotion payments rank first; the base publishing fee is UGX 34,000.
These are manual instructions, not a payment-provider integration. Check
currency conversion, fees, merchant account ownership, and local legal
requirements before accepting payments. Never ask a user for their PIN or
password. Keep API secrets and admin tokens server-side.
The MTN logo is downloaded from the [official MTN Uganda website](https://www.mtn.co.ug/momo/).
The Airtel, PayPal, and Payoneer symbol marks are local SVG copies sourced from
Simple Icons (CC0): [Airtel](https://cdn.simpleicons.org/airtel),
[PayPal](https://cdn.simpleicons.org/paypal), and
[Payoneer](https://cdn.simpleicons.org/payoneer). Logos are bundled locally
and cached by the PWA for offline use. They are brand identifiers, not signs
that a provider has approved or integrated UG18.

## Event publishing and payment confirmation

Creating an event saves a private listing and its photo; visitors only see it
after payment is verified. The optional promotion amount is capped at UGX
1,000,000 and is added to the UGX 34,000 publishing fee. The application stores
the host's submitted transaction reference for an operator to check.

Configure a strong `PAYMENTS_ADMIN_TOKEN` on the server. Operators can list
submitted references with `GET /api/admin/events/pending-payments`, supplying
the configured token in the `Authorization` header. After checking the transaction in the provider's
merchant portal, an authorized operator can call
`POST /api/admin/events/{eventId}/confirm-payment` with the same authorization
header and these JSON fields:

```json
{
  "provider": "mtn",
  "transactionId": "provider-verified-reference",
  "amountUgx": 44000
}
```

The example total assumes a UGX 10,000 promotion; use that event's actual total.
The provider and transaction reference must match the host's submission.
Transaction references can only be used once. `PAYMENTS_ADMIN_TOKEN` must stay
in server-side environment configuration, never in the browser or source
control. Before automating payments, integrate provider-authenticated APIs and
signed webhooks and test refunds and failure cases.

For local operator verification, set the token before starting the API:

```powershell
$env:PAYMENTS_ADMIN_TOKEN = node -p "require('node:crypto').randomBytes(32).toString('base64url')"
npm run dev
```

After checking a submitted reference in the merchant account, use the same
token to list the queue and confirm a verified event:

```powershell
$headers = @{ Authorization = "Bearer $env:PAYMENTS_ADMIN_TOKEN" }
Invoke-RestMethod -Uri "http://localhost:4100/api/admin/events/pending-payments" -Headers $headers
$body = @{ provider = "mtn"; transactionId = "the-host-submitted-reference"; amountUgx = 44000 } | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:4100/api/admin/events/EVENT-ID/confirm-payment" -Method Post -Headers $headers -ContentType "application/json" -Body $body
```
## Private account spreadsheet

The sidebar’s **Accounts spreadsheet** view is protected by a separate server
admin token. It displays account contact details in a paginated table and
downloads all accounts as `ug18-accounts.csv`, which opens in Excel. Passwords,
password hashes, and session tokens are excluded. Because the export contains
personal data, keep it private and delete downloaded copies when they are no
longer needed.

Set `UG18_ADMIN_TOKEN` in the same terminal before starting the server. Generate
a strong token in PowerShell and then start the app:

```powershell
$env:UG18_ADMIN_TOKEN = node -p "require('node:crypto').randomBytes(32).toString('base64url')"
$env:PAYMENTS_ADMIN_TOKEN = node -p "require('node:crypto').randomBytes(32).toString('base64url')"
Write-Output "Copy this generated token into the Accounts spreadsheet screen: $env:UG18_ADMIN_TOKEN"
npm run dev
```

In the app, open **Accounts spreadsheet** and enter that token. It stays only
in page memory and is not saved in browser storage. Treat the copied token as
a secret. The CSV endpoint is also available to an authorized operator:

```powershell
$headers = @{ Authorization = "Bearer $env:UG18_ADMIN_TOKEN" }
Invoke-WebRequest -Uri "http://localhost:4100/api/admin/accounts.csv" -Headers $headers -OutFile ".\ug18-accounts.csv"
```

The admin API returns an explicit unavailable response until the server token
is configured. Use HTTPS and a secret manager for any deployed environment.

Light and dark modes can be switched from the header and are saved on the
device. Event listings are grouped under their selected category after
publishing, and the event photo/contact information is visible to viewers once
the verified event is public.

## Before a public launch

Configure HTTPS, backups, account recovery, email/phone verification, moderation
and report/block workflows, retention rules, and a privacy policy and terms.
This starter does not provide production payment processing, identity
verification, or end-to-end encrypted messaging.

## Publishing publicly

This is a full-stack Node.js app, not a static site: the Express API handles
accounts and the SQLite database. Deploy it to a Node host that supports a
**persistent disk/volume**; static-only hosting and ephemeral serverless
instances will lose account data and event photos. SQLite is intended for one
app instance with the database and photos on the same persistent volume.

1. Create a **private** GitHub repository and publish this project from VS
   Code's Source Control view. The existing `.gitignore` excludes `data/`,
   `node_modules/`, build output, and `.env` files; do not force-add account
   databases, uploaded event photos, or secrets.
2. In Render, choose **New → Blueprint**, connect the private repository, and
   select this project's `render.yaml`. The blueprint configures Node, build
   and start commands, `/api/health`, a persistent disk mounted at `/var/data`,
   `UG18_DATA_DIR`, and generated admin secrets. Render's persistent disk
   requires a paid web-service plan; keep a single instance while using SQLite.
3. Wait for the first deploy to finish. Render assigns a free HTTPS
   `*.onrender.com` address. Share that address to make the site publicly
   reachable; no domain purchase is required to start.
4. The blueprint generates `UG18_ADMIN_TOKEN` and `PAYMENTS_ADMIN_TOKEN` as
   server secrets. Optional payment destination overrides are
   `MTN_MERCHANT_CODE`, `AIRTEL_MERCHANT_CODE`, `PAYPAL_RECIPIENT_EMAIL`, and
   `PAYONEER_RECIPIENT_EMAIL`. Do not add tokens, merchant PINs, or provider
   API secrets to the frontend or Git.
5. Test registration, login, account export, event photo persistence, payment
   reference submission, manual verification, and event publication on the
   deployed URL. Set up encrypted, off-host backups and test restoring them
   before inviting the public.

Without a persistent volume, SQLite accounts and event photos may disappear
when the host restarts or redeploys. Review privacy, adult-content, consumer
protection, and payment rules for the regions where the site will operate
before opening registrations or accepting money.
