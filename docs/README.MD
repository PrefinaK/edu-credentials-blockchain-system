
# EduCredentials — Education Credentials Starter Kit

> Decentralised academic credential issuance and verification on Cardano.
> Milestone 16 — Zonke / Tobb Technologies Programme.

---

## Overview

EduCredentials allows educational institutions to issue tamper-proof diplomas,
certificates, and micro-credentials. A cryptographic hash of each credential is
anchored on the Cardano blockchain as a native token. Anyone can verify a
credential instantly by submitting the hash — no login, no phone calls, no delays.

The application runs in **database-verified mode** by default. This means all
credential issuance, verification, and revocation works through MySQL without
requiring a live Cardano node. When `PLUTUS_POLICY_ID` and
`BLOCKFROST_PROJECT_ID` are configured, the same codebase enables full
on-chain anchoring with zero code changes.

---

## Three roles

| Role | Who | What they can do |
|---|---|---|
| **Issuer** | University / training provider | Issue, view, and revoke credentials |
| **Student** | Credential holder | Claim via link, view details, share verification URL |
| **Verifier** | Employer / third party | Verify any credential by hash or QR code — no login needed |

---

## Technology stack

| Layer | Technology | Purpose |
|---|---|---|
| On-chain | Haskell / Plutus V2 | Minting policy — enforces issuer authorisation and credential integrity |
| Chain queries | Blockfrost API | Query Cardano without running a full node |
| Backend | Node.js 18 + Express | REST API, business logic, hash generation |
| Database | MySQL 8 | Off-chain private data — all PII stored here only |
| Frontend | HTML + CSS + JavaScript | Three-role dashboards — no framework required |
| Auth | JWT HS256 + bcrypt | Stateless issuer authentication |
| Hosting | Tobb Technologies | Node.js + MySQL environment |

---

## Project structure

```
edu-credentials/
├── contracts/
│   ├── CredentialPolicy.hs      # Plutus V2 minting policy (Haskell)
│   ├── CompilePolicy.hs         # Serialises policy to CBOR for deployment
│   └── edu-credentials.cabal   # Haskell build configuration
├── backend/
│   ├── src/
│   │   ├── app.js               # Express server entry point
│   │   ├── config/database.js   # MySQL connection pool
│   │   ├── routes/
│   │   │   ├── auth.js          # Login, profile, first-run setup
│   │   │   ├── credentials.js   # Issue, list, revoke, student claim
│   │   │   └── verify.js        # Public verification endpoint
│   │   ├── middleware/auth.js   # JWT authentication guard
│   │   └── utils/
│   │       ├── hash.js          # SHA-256 credential hash generation
│   │       └── cardano.js       # Blockfrost chain query helpers
│   ├── schema.sql               # MySQL schema — run once to initialise
│   ├── package.json
│   └── .env.example             # Copy to .env and fill in your values
├── frontend/
│   ├── index.html               # Login page (Issuer + Student tabs)
│   ├── issuer.html              # Issuer dashboard
│   ├── student.html             # Student credential portal
│   ├── verifier.html            # Public verifier — no login required
│   └── assets/
│       ├── style.css            # Design system — CSS custom properties
│       └── app.js               # Shared auth, API client, UI helpers
├── docs/
│   ├── SPEC.md                  # Customer-facing specification
│   ├── DESIGN.md                # Architecture and design decisions
│   └── USER_MANUAL.md           # End-user guide for all three roles
├── tests/
│   └── backend.test.js          # Unit tests — run with npm test
└── README.md
```

---

## Local setup

### Prerequisites

- Node.js 18 or higher
- MySQL 8 or higher
- Git

### Step 1 — Clone your fork

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/coxygen-cardano-haskell-industry-starter-kits.git
cd coxygen-cardano-haskell-industry-starter-kits
```

### Step 2 — Install dependencies

```bash
cd backend
npm install
```

### Step 3 — Configure environment

```bash
cp .env.example .env
```

Open `.env` and set these values:

```env
PORT=3000
NODE_ENV=development

# Database
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=edu_credentials
DB_USER=root
DB_PASS=your_mysql_password

# Authentication
JWT_SECRET=any_64_character_random_string_here
JWT_EXPIRES_IN=8h

# Cardano (leave empty for database-verified mode)
BLOCKFROST_PROJECT_ID=previewXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
CARDANO_NETWORK=preview
PLUTUS_POLICY_ID=

# Application
SETUP_KEY=choose_a_secret_key_for_first_run
CORS_ORIGINS=http://127.0.0.1:5500
QR_BASE_URL=http://127.0.0.1:5500/frontend/verifier.html
```

### Step 4 — Create the database

**Linux / Mac:**
```bash
mysql -u root -p < schema.sql
```

**Windows PowerShell:**
```powershell
Get-Content schema.sql | mysql -u root -P 3306 -p
```

### Step 5 — Create the first admin account

Start the server first, then in a second terminal:

```bash
curl -X POST http://localhost:3000/api/auth/setup \
  -H "Content-Type: application/json" \
  -d '{
    "setup_key": "your_setup_key",
    "email": "admin@university.edu",
    "password": "StrongPassword123",
    "institution_name": "My University",
    "wallet_address": "addr_test1q..."
  }'
```

### Step 6 — Start the server

```bash
npm run dev
```

Expected output:
```
[App] EduCredentials API running on port 3000
[DB] MySQL connected successfully
```

### Step 7 — Open the frontend

In VS Code, right-click `frontend/index.html` and select **Open with Live Server**.

Navigate to: `http://127.0.0.1:5500/frontend/index.html`

Login with the credentials you created in Step 5.

---

## Running tests

```bash
cd backend
npm test
```

Tests cover credential hash generation, hash verification, claim token
uniqueness, API auth guards, and the public verification endpoint.

---

## Running without a deployed Plutus policy

When `PLUTUS_POLICY_ID` is empty, the system runs in **database-verified mode**:

- All credential issuance, verification, and revocation work through MySQL
- The verifier shows "Database verified (chain pending)" instead of "Confirmed on Cardano"
- No Cardano wallet or testnet ADA is required

**This is the expected mode for local development and starter kit evaluation.**

To enable full on-chain verification, see the Plutus section below.

---

## Plutus smart contract

The minting policy (`contracts/CredentialPolicy.hs`) enforces four on-chain rules:

1. Only the authorised issuer PubKeyHash can mint tokens
2. Token name must be exactly 28 bytes (the credential hash)
3. Exactly one token is minted per credential transaction
4. Revocation (burning) requires the issuer's signature

### Compiling the contract

> Requires GHC + Cabal with Plutus libraries.
> Recommended environment: [Demeter.run](https://demeter.run) (browser-based, no local setup).

```bash
cd contracts
cabal build
cabal run compile-policy -- <your-issuer-pubkeyhash-hex>
# Outputs: policy.plutus (CBOR file)
# Prints:  PolicyId to use as PLUTUS_POLICY_ID
```

Copy the printed PolicyId into your `.env` as `PLUTUS_POLICY_ID`.

Get free testnet ADA from the [Cardano faucet](https://docs.cardano.org/cardano-testnets/tools/faucet/).

---

## Deploying to Tobb Technologies

1. Create a MySQL database in the Tobb Technologies control panel
2. Set all environment variables from `.env.example` in the Tobb settings panel
3. Upload the `backend/` folder to your Tobb Node.js application root
4. Run `npm install --production` on the server
5. Initialise the database with `schema.sql`
6. Tobb starts the app using `npm start`
7. Upload `frontend/` to your Tobb static files directory
8. Update `API_BASE` in `frontend/assets/app.js` to your Tobb backend domain

---

## Privacy design

| Data | Location | Reason |
|---|---|---|
| Student name, email, grade | MySQL only | PII — never stored on-chain |
| Institution details | MySQL only | Operational metadata |
| Credential hash (28 bytes) | MySQL + Cardano UTxO | Tamper-proof proof of issuance |
| Issuer wallet address | Cardano transaction | Proves who authorised the issuance |
| Verification audit logs | MySQL (IP address only) | Audit trail without PII |

No personally identifiable information ever appears on the public Cardano
blockchain. The credential hash is a one-way SHA-256 digest — it cannot be
reversed to recover the original student data.

---

## API endpoints

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | /api/auth/login | None | Login — returns JWT |
| GET | /api/auth/me | JWT | Current user and institution |
| POST | /api/auth/setup | Setup key | First-run admin creation |
| GET | /api/credentials | JWT | List credentials with filters |
| POST | /api/credentials | JWT | Issue a new credential |
| GET | /api/credentials/:id | JWT | Single credential detail |
| PATCH | /api/credentials/:id/revoke | JWT | Revoke a credential |
| GET | /api/credentials/claim/:token | None | Student claim endpoint |
| GET | /api/verify?hash=... | None | Public credential verification |
| GET | /health | None | Server health check |

---

## EBU Plutus certification

**EBU Plutus certified graduate: No**

---

## Licence

Apache 2.0 — see LICENSE.
