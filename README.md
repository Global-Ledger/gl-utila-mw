# Utila ↔ Global Ledger Approval Middleware

This middleware connects Utila transaction approvals with Global Ledger (GL) risk scoring.

For each outgoing transaction that reaches `AWAITING_APPROVAL`, the middleware:

1. Fetches full transaction details from Utila.
2. Extracts destination address (and token contract when relevant).
3. Requests GL risk score.
4. Sends `APPROVE` or `DENY` vote back to Utila based on `RISK_THRESHOLD`.

The service is stateless and runs as a single container behind an HTTPS webhook endpoint.

---

## 1) Package contents

You receive:

- `utila-gl-mw.obf.js` — obfuscated middleware script (Node.js ESM).
- `.env.example` — configuration template.
- `docker-compose.yml` — reference deployment manifest.

All customization is done via environment variables.

---

## 2) Prerequisites

### 2.1 Utila setup

- Step 1: Create a Service Account in Utila Console (`Settings` → `Service Accounts`).
- Step 2: Save the service account email (example: `gl-...@vault-<vaultId>.utilaserviceaccount.io`).
- Step 3: Generate RSA key pair (4096 bits):

```bash
openssl genpkey -algorithm RSA -out utila_private.pem -pkeyopt rsa_keygen_bits:4096
openssl rsa -in utila_private.pem -pubout -out utila_public.pem
```

- Step 4: Upload `utila_public.pem` to the Service Account in Utila.
- Step 5: Configure transaction policy to require Service Account approval (`Settings` → `Policies` → `Transaction Policies`).
- Step 6: Configure webhook (`Settings` → `Webhooks`) with URL:

```text
https://<your-public-domain>/webhook
```

- Step 7: Enable webhook event `TRANSACTION_STATE_UPDATED` (optional: `TRANSACTION_CREATED`, `TEST`).

### 2.2 Global Ledger setup

Obtain a valid `GL_API_KEY` from Global Ledger.

---

## 3) Configuration

Create `.env` from `.env.example`.

### Environment variables

| Variable | Required | Example | Description |
| --- | --- | --- | --- |
| `UTILA_EMAIL` | Yes | `gl-...@vault-...utilaserviceaccount.io` | Utila Service Account email |
| `UTILA_PRIVATE_KEY_PATH` | Yes | `/run/secrets/utila_private.pem` | Path to RSA private key |
| `GL_API_KEY` | Yes | `...` | Global Ledger API key |
| `RISK_THRESHOLD` | Yes | `60` | If `score >= threshold` -> `DENY`, else `APPROVE` |
| `PORT` | Yes | `3000` | Web server port |
| `JWT_SKEW_SEC` | No | `120` | JWT issue-time skew (helps with clock drift) |
| `JWT_TTL_SEC` | No | `3600` | JWT TTL (must be `<= 3600`) |
| `MAX_TX_ATTEMPTS` | No | `12` | Retries while waiting for transfers in Utila transaction |
| `SCAN_PAGE_SIZE` | No | `100` | Startup scan page size |
| `VAULT_ID` | No | `12a5...` | Limit startup scan to one vault |
| `DEFAULT_DECISION_UNSUPPORTED` | No | `APPROVE` | Fallback decision for unmapped networks (`APPROVE` or `DENY`) |

Behavior:

- On startup, middleware scans pending `AWAITING_APPROVAL` transactions.
- Middleware only votes on outgoing `AWAITING_APPROVAL` transactions.
- Middleware does not sign transactions; it only sends approval votes.

---

## 4) Network mapping (`transaction.network`)

Middleware reads Utila `transaction.network` from transaction details and uses it to select the GL chain endpoint.

Format:

```text
networks/{network_id}
```

Examples:

- `networks/bitcoin-mainnet`
- `networks/ethereum-mainnet`
- `networks/ethereum-testnet-sepolia`

If network mapping is missing, middleware applies `DEFAULT_DECISION_UNSUPPORTED`.

Operational recommendation:

- Validate real `transaction.network` values in your environment before go-live.
- Use `DEFAULT_DECISION_UNSUPPORTED=DENY` for fail-closed posture.

References:

- `https://docs.utila.io/reference/transactions_gettransaction`
- `https://docs.utila.io/reference/transactions_listtransactions`

Note: if GL returns HTTP `404` with `totalFunds` fields, middleware treats it as a valid scoring response.

---

## 5) Deployment (Docker Compose)

Place these files in one directory:

- `docker-compose.yml`
- `.env`
- `utila-gl-mw.obf.js`
- `utila_private.pem`

Start:

```bash
docker compose up -d
docker compose logs -f
```

Webhook endpoint inside host:

- `http://localhost:<PORT>/webhook`

Expose it publicly via HTTPS through reverse proxy or load balancer.

If you use hardened security profile, replace disk-based key/API-key handling with runtime secrets:

- inject `GL_API_KEY` from Vault/secret manager at runtime;
- mount `utila_private.pem` from secrets backend (instead of storing it as a regular file on disk).

---

## 6) Reverse proxy requirements

Ensure:

- `POST https://<domain>/webhook` forwards to `http://127.0.0.1:<PORT>/webhook`.
- Request body is not modified.
- Timeout is sufficient (for example, `30s`).

---

## 7) Security profiles

The key pair in this guide is a service-account signing key pair for Utila JWT auth (not SSH).

- `utila_private.pem` is secret.
- `utila_public.pem` is not secret, but key integrity must be controlled.

### 7.1 Profile A: baseline

Suitable for small teams and non-critical environments:

- Keep non-sensitive settings in `.env`.
- `GL_API_KEY` may be stored in `.env` if policy allows.
- Keep `utila_private.pem` on disk with strict permissions (`0400`/`0600`).
- Never commit `.env` or key files to git.

### 7.2 Profile B: hardened (recommended for production)

Suitable for regulated or high-security environments:

- Keep `.env` non-sensitive only.
- Inject `GL_API_KEY` at runtime from Vault or equivalent secret manager.
- Mount `utila_private.pem` as runtime secret (Docker/Kubernetes secret, tmpfs, Vault agent sidecar).
- Restrict inbound traffic to known sources and enforce HTTPS.
- Use least-privilege access and define key/API-key rotation schedule.
- Prefer fail-closed default: `DEFAULT_DECISION_UNSUPPORTED=DENY`.

Example (runtime injection for `GL_API_KEY`):

```yaml
services:
  utila-gl-mw:
    image: your-registry/utila-gl-mw:latest
    env_file:
      - .env
    environment:
      GL_API_KEY: ${GL_API_KEY}
```

---

## 8) Troubleshooting

Auth issues:

- Verify `UTILA_EMAIL` matches the Service Account email in Utila.
- Verify uploaded public key matches your `utila_private.pem`.
- Verify `JWT_TTL_SEC <= 3600`.
- If you see `token used before issued`, increase `JWT_SKEW_SEC`.

No webhooks:

- Verify Utila webhook URL points to your public HTTPS endpoint.
- Verify reverse proxy routes `/webhook` correctly.
- Verify event `TRANSACTION_STATE_UPDATED` is enabled.

Votes not applied:

- Verify policy requires Service Account approval.
- Verify middleware logs contain successful call to `:vote`.

Unexpected decision on rare/new network:

- Check `transaction.network` via Utila `GetTransaction`.
- If mapping is missing, `DEFAULT_DECISION_UNSUPPORTED` is used.

---

## 9) Support

When contacting Global Ledger support, provide:

- Middleware logs around the failing transaction.
- Utila transaction ID and vault ID.
- Timestamp and environment (`prod`/`stage`).
