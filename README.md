# GradTrust — Blockchain-Verified Academic Credentials

GradTrust is a decentralized platform for **issuing, holding and instantly verifying academic credentials**
(diplomas, certificates) on Ethereum. Universities issue tamper-proof credentials on-chain, graduates
carry them in their wallet, and employers can verify one in seconds without contacting the registrar.

Built as a senior capstone project at Texas A&M University by a team of six.

## The problem

Degree verification today is slow, manual and easy to fake: diploma mills and forged transcripts are
common, and confirming a credential often means emailing a registrar and waiting days. GradTrust replaces
that with a cryptographic check against a public ledger.

## How it works

GradTrust has three roles, each with its own dashboard:

| Role         | What they do                                                                 |
| ------------ | ---------------------------------------------------------------------------- |
| **Issuer**   | An accredited institution that signs and publishes credentials on-chain      |
| **Holder**   | A graduate who views their credentials and shares them (including via an NFC-friendly profile link) |
| **Verifier** | An employer or third party who checks a credential's authenticity by its hash |

```
                ┌──────────────────────┐
  Admins ──────►│   IssuerRegistry     │  Merkle root of all trusted issuers
 (multi-sig)    └──────────┬───────────┘
                           │ verifyIssuer(proof)
                ┌──────────▼───────────┐
  Issuer ──────►│ CredentialVerification│  credentialHash → {issuer, holder, issuedAt, data}
                └──────────┬───────────┘
                           │ pullCredential / verifyCredential
          Holder ◄─────────┴─────────► Verifier
```

### Smart contracts (Solidity, Sepolia testnet)

- **`IssuerRegistry.sol`** stores a single **Merkle root** representing every trusted issuer. Checking
  whether an issuer is legitimate is an O(log n) Merkle-proof verification, so the issuer list never has
  to be stored on-chain.
- **`CredentialVerification.sol`** stores credentials keyed by their hash. The `onlyVerifiedIssuer`
  modifier rejects any `storeCredential` call that doesn't include a valid Merkle proof for the issuer.
  Holders can pull all their credentials, and anyone can call `verifyCredential(hash)`.

### Security design

- **Merkle-tree issuer allow-list.** Adding or removing an issuer only updates one 32-byte root on-chain.
- **Multi-signature root updates.** Changes to the issuer registry require signed approval from several
  admin wallets. Signatures are recovered server-side with `eth_account` and checked against the admin
  set before the root is updated.
- **Wallet-signed issuer registration.** Issuers prove control of their address by signing a message with
  added entropy.
- **JWT authentication** (header and cookie) with bcrypt-hashed, peppered passwords and role-based access
  to each dashboard.

## Tech stack

| Layer           | Technologies                                                        |
| --------------- | ------------------------------------------------------------------- |
| Smart contracts | Solidity 0.8.20, Hardhat, Ethereum Sepolia                          |
| Backend         | Python 3.12, Flask, Flask-JWT-Extended, Flask-SocketIO, web3.py, PostgreSQL (SQLAlchemy / psycopg2) |
| Frontend        | Next.js (App Router), TypeScript, Tailwind CSS, shadcn/ui, NextAuth |
| DevOps          | GitHub Actions (Hardhat CI), Docker, Procfile-based deployment      |

## Project structure

```
├── contracts/            # Solidity smart contracts
├── scripts/deploy.js     # Hardhat deployment script
├── test/                 # Hardhat tests (run in CI)
├── backend/
│   ├── app.py            # Flask app factory
│   ├── routes/           # admin, admin_multi, issuer, holder, verifier, common blueprints
│   ├── abis/             # Contract ABIs used by web3.py
│   └── models.py         # SQLAlchemy models
└── frontend/src/my-app/  # Next.js frontend (issuer, holder, verifier, admin dashboards)
```

## Getting started

### Prerequisites

- Python 3.12, Node.js 18+
- A PostgreSQL database
- A Sepolia RPC endpoint (e.g. Alchemy or Infura) and a funded deployer wallet

### 1. Configure environment

Create a `.env` file in the repository root:

```env
# Blockchain
API_URL=<sepolia-rpc-url>
PRIVATE_KEY=<deployer-wallet-private-key>
NETWORK=sepolia
ISSUER_REGISTRY_ADDRESS=<deployed-address>
CREDENTIAL_VERIFICATION_ADDRESS=<deployed-address>

# Backend
CONNECTION_STRING=postgresql://<user>:<password>@<host>/<db>
JWT_SECRET_KEY=<random-secret>
SECRET_KEY=<random-secret>
PEPPER=<random-secret>
FRONTEND_ORIGIN=http://localhost:3000
BACKEND_DEPLOYMENT=http://localhost:5000
FLASK_DEBUG=1
```

### 2. Deploy the contracts

```bash
npm install
npx hardhat compile
npx hardhat test
npx hardhat run scripts/deploy.js --network sepolia
```

Copy the printed contract addresses into `.env`.

### 3. Run the backend

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
flask --app backend.app run
```

### 4. Run the frontend

```bash
cd frontend/src/my-app
npm install
npm run dev
```

## Team

Gabriel Floreslovo · Segun Oluwole · Nathan Cornelius · Blake Dejohn · Joshua Wilson · Ezra Jeter

## License

See [LICENSE](LICENSE). This project is the original work of the GradTrust team and was submitted to
fulfill academic requirements at Texas A&M University.
