# Utila ↔ Global Ledger Approval Middleware

This middleware integrates **Utila Transaction Approval** workflow with **Global Ledger (GL) risk scoring**.

When Utila creates an outgoing transaction that reaches `AWAITING_APPROVAL`, the middleware:
1) fetches full transaction details from Utila;
2) extracts destination address (+ token contract when it is a token transfer);
3) requests the GL risk score;
4) votes **APPROVE** or **DENY** back to Utila based on the configured threshold.

The service is stateless and can run as a single container behind HTTPS with a public webhook endpoint.

---

## 1) What you will receive from Global Ledger

You will receive two files:

- `utila-gl-mw.obf.js` — **obfuscated** middleware script (Node.js ESM).
- `.env.example` — configuration template.

> You do NOT need to modify the script. All customization is done via environment variables.

---

## 2) Prerequisites

### 2.1 Utila (required)

Configure the following in Utila Console:

#### A) Create a Service Account
- Utila Console → Settings → Service Accounts → Create
- Save the **service account email** (example):  
  `gl-...@vault-<vaultId>.utilaserviceaccount.io`

#### B) Create an RSA private key (4096 bits)
Generate locally:

```bash
openssl genpkey -algorithm RSA -out utila_private.pem -pkeyopt rsa_keygen_bits:4096
openssl rsa -in utila_private.pem -pubout -out utila_public.pem
```

Upload `utila_public.pem` (public key) into the Service Account settings in Utila.

Keep `utila_private.pem` secure — it is used to sign JWTs for Utila API.

#### C) Configure Transaction Policy
- Utila Console → Settings → Policies → Transaction Policies
- Create a rule to require approval by your Service Account
- This ensures transactions enter `AWAITING_APPROVAL` and webhook events are emitted.

#### D) Configure Webhook
- Utila Console → Settings → Webhooks → Add webhook
- URL:

```
https://<your-public-domain>/webhook
```

- Enable events:
  - `TRANSACTION_STATE_UPDATED`
  - (optional) `TRANSACTION_CREATED`
  - (optional) `TEST`

### 2.2 Global Ledger (required)

Obtain a valid **GL API key** from Global Ledger.

---

## 3) Configuration

Create `.env` from the provided `.env.example`.

### 3.1 Environment variables

| Variable | Required | Example | Description |
|---|---:|---|---|
| `UTILA_EMAIL` | ✅ | `gl-...@vault-...utilaserviceaccount.io` | Utila Service Account email |
| `UTILA_PRIVATE_KEY_PATH` | ✅ | `/run/secrets/utila_private.pem` | Path to RSA private key |
| `GL_API_KEY` | ✅ | `...` | Global Ledger API key |
| `RISK_THRESHOLD` | ✅ | `60` | If `score >= threshold` → `DENY`, else `APPROVE` |
| `PORT` | ✅ | `3000` | Web server port |
| `JWT_SKEW_SEC` | optional | `120` | JWT iat skew (prevents “token used before issued”) |
| `JWT_TTL_SEC` | optional | `3600` | JWT TTL (must be <= 3600) |
| `MAX_TX_ATTEMPTS` | optional | `12` | Retry count while waiting for transfers in Utila tx |
| `SCAN_PAGE_SIZE` | optional | `100` | Startup scan page size |
| `VAULT_ID` | optional | `12a5...` | If set, scan only this vault |
| `DEFAULT_DECISION_UNSUPPORTED` | optional | `APPROVE` | APPROVE or DENY if network mapping is unsupported |

### 3.2 Important behavior notes

- On startup the service scans Utila for **pending transactions** in `AWAITING_APPROVAL` and processes them.
- The service **only votes** on `AWAITING_APPROVAL` outgoing transactions.
- The service does NOT sign transactions. It only votes APPROVE/DENY.

---

## 4) Supported networks & scoring modes

Global Ledger supports two scoring modes:
- **Advanced** mode: `https://<chain>.glprotocol.com/...` (token is always `supported`)
- **Essential** mode: `https://common.glprotocol.com/essential-api-<chain>/...` (token contract is included only if it exists)

Network mapping is based on Utila `transaction.network` values.

> If GL responds with HTTP 404 but returns `totalFunds` fields, the middleware treats it as a valid response and uses the score.

---

## 5) Run with Docker Compose (recommended)

### 5.1 Files required on the server

Place these files in one directory:
- `docker-compose.yml`
- `.env`
- `utila-gl-mw.obf.js`
- `utila_private.pem` (your RSA private key)

### 5.2 Start

```bash
docker compose up -d
docker compose logs -f
```

The service listens on:
- `http://localhost:<PORT>/webhook` (host port → container port)

> You must expose it publicly via HTTPS using your reverse proxy / load balancer (recommended).

---

## 6) Reverse proxy (HTTPS) requirements

Utila webhook must call an HTTPS endpoint. Use one of:
- Nginx / Caddy on the same host
- Cloud load balancer / ingress

Ensure:
- `POST https://<domain>/webhook` forwards to `http://127.0.0.1:<PORT>/webhook`
- Request body is not modified
- Timeouts are reasonable (e.g. 30s)

---

## 7) Troubleshooting

### Auth test fails
- Check `UTILA_EMAIL` matches the service account email in Utila.
- Ensure the public key uploaded to Utila matches your `utila_private.pem`.
- Ensure `JWT_TTL_SEC <= 3600`.
- If you see `token used before issued`, increase `JWT_SKEW_SEC` (e.g. 120 → 300).

### No webhooks received
- Verify webhook URL in Utila points to your public HTTPS endpoint.
- Check reverse proxy routing to `/webhook`.
- Verify enabled events include `TRANSACTION_STATE_UPDATED`.

### Votes not applied
- Confirm the transaction policy requires approval from the service account.
- Ensure middleware logs show successful call to `:vote`.

---

## 8) Security notes (must read)

- Store `utila_private.pem` securely (Docker secret / filesystem permissions).
- Do not commit `.env` into git.
- Run behind HTTPS.
- Restrict inbound access to only Utila IPs if possible.
- Rotate GL API keys and Utila service keys per your internal policy.

---

## 9) Support

If you need help, contact Global Ledger support and provide:
- middleware logs around the failed transaction
- Utila transaction id + vault id
- timestamp and environment (prod/stage)
