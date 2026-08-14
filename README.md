# HenBridge — Card

[![Built on Stellar](https://img.shields.io/badge/Built%20on-Stellar-blue?logo=stellar)](https://stellar.org)
[![Soroban](https://img.shields.io/badge/Trust%20layer-Soroban-purple)](https://soroban.stellar.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](#license)
[![Status: Pre-alpha](https://img.shields.io/badge/status-pre--alpha-orange)](#where-were-headed)

**The bridge between a patient who can't speak and the responder who has to act.**

HenBridge Card puts the five or six facts that actually change emergency treatment — blood group, genotype, allergies, current medications, chronic conditions — into a QR code the patient carries. It opens with no login, keeps working with no signal, and carries a verification a first responder can trust without phoning a clinic.

> **Where things stand:** Pre-alpha · Stellar **testnet** · Live at [henbridge-web.vercel.app](https://henbridge-web.vercel.app) · not audited · **not a medical device** (see [Disclaimer](#disclaimer)).

This repository — **`henbridge_frontend`** — is the web app: the public emergency page, the authenticated profile editor, and QR generation. The on-chain trust layer, the community-health-worker service, and the design docs live in sibling repos ([the HenBridge org](#the-henbridge-org)).

---

## The gap we close

Picture a clinic in Nigeria at 2 a.m. A patient arrives unconscious after a road accident. The people treating them don't know the blood group, don't know the sickle-cell status, don't know which drugs will trigger a reaction — and there is no one to ask. The paper card, if it ever existed, is at another facility three referrals ago.

That gap is where people die of *avoidable* mistakes. Four things make it stubborn:

- **Nothing portable survives the journey** — paper doesn't outlast a move, a referral, or a bad day.
- **A claim isn't proof** — even a conscious patient stating "I'm O-negative, allergic to penicillin" gives a responder nothing checkable under time pressure.
- **The last mile has no economics** — community health workers (CHWs) are best placed to register and verify people, but nobody pays them to.
- **The obvious fix is a honeypot** — a central database of everyone's health records is exactly what patients and regulators are right to fear.

HenBridge threads that needle: minimal data, patient-controlled, encrypted off-chain — with trust and payment handled on Stellar so no health data ever leaves the private store.

## Who gets what

| If you are… | HenBridge gives you… |
| --- | --- |
| a patient or parent | a free card — on a phone or printed — that speaks the critical facts when you can't |
| a responder or clinician | a QR you scan with no account, showing only the decision-relevant subset, marked *verified* when it has been |
| a community health worker | a paid reason to register and verify people in the field, settled in USDC on Stellar |

## How a card actually works

1. **Create.** A patient signs up and fills a field-by-field editor — identity, blood group / genotype, allergies, medications, chronic conditions, up to three emergency contacts, an optional photo. It lands encrypted in Supabase, gated by Row-Level Security.
2. **Publish.** The patient chooses what appears on a minimal, read-only public page and gets a QR that points to it (`/card/[id]`).
3. **Scan.** A responder opens that QR — no login — and sees only the emergency subset.
4. **Trust.** If an allowlisted health worker has attested the record on-chain, the page shows a **verified** badge the responder can rely on. The check reads a Soroban attestation; the health data itself was never on the chain.
5. **Survive a dead zone.** A card the responder has opened once stays readable offline, with a visible "as of `<time>`" staleness banner so nobody mistakes a cached copy for a live one.

## The emergency data set

Deliberately small — the public page carries only what changes treatment in the first minutes:

- Name, age, photo
- **Blood group and genotype**
- Drug allergies
- Current medications (especially anticoagulants, insulin, anti-epileptics)
- Chronic conditions / implants
- Emergency contact(s)
- Spoken language

Full history, documents, and notes never leave the authenticated side.

## Trust and money, handled on Stellar

Two jobs a plain web app can't do — and the reason Stellar is load-bearing here, not decorative:

| Capability | Mechanism | Guarantee |
| --- | --- | --- |
| **Verification** | A Soroban attestation: `hash(record) + attester identity + timestamp` | A responder can prove a real, allowlisted worker checked *this exact record* — without the record ever being exposed |
| **Incentives** | USDC on Stellar, one micro-payment per verified registration | Sub-cent, cross-border settlement makes last-mile outreach pay |
| **Funding** | Grants and donations flow on-chain into the payout pool | Every dollar in maps to a countable number of verified cards out |

> **The one invariant that governs everything:** no personal health data ever touches the blockchain. It lives encrypted and access-controlled in Supabase; Stellar holds only hashes, attestations, and payments. That line is what keeps HenBridge both private and regulator-compatible.

### Where this repo meets the chain

The wiring is already here, contract or no contract:

- `lib/attestation/recordHash.ts` derives the deterministic hash a contract call would use.
- `lib/stellar/attestation.ts` exposes `getAttestation(recordHash)` with the real Soroban signature. When `ATTESTATION_CONTRACT_ID` is set it runs a read-only `simulateTransaction` against the deployed registry via `@stellar/stellar-sdk`; unset, it serves an in-memory mock. An unattested hash comes back `null` (shown as *not verified*).

So the public page and the `/api/attestation/[recordHash]` route behave identically before and after the registry goes live — only the env var changes.

## Offline, without lying about freshness

The card matters most exactly where there's no network, so a service worker (`public/sw.js`) makes a previously-seen card render offline:

- **Only what was actually opened** is cached — the rendered HTML of each `/card/[id]` a device visited *while online*. Nothing is prefetched.
- **Network-first, cache on success.** Each good navigation stores the HTML plus a timestamp. Only `2xx` is cached — a `404` from an unknown id is never stored, so a stale "not found" can't be served later.
- **Staleness is shown, not hidden.** A served-from-cache page carries a sticky banner: *Showing cached data as of `<time>`…* — inline-styled so it appears before any stylesheet loads.
- **Scoped tight.** The worker registers for the whole origin but its fetch handler only touches `/card/*` navigations and their stylesheets; auth, API, and everything else pass straight through. JS chunks are deliberately not cached, so an offline card can't silently re-hydrate and drop the banner.

Helpers in `public/offline-cache-helpers.js` are unit-tested; end-to-end behaviour is exercised by the manual protocol in [Testing](#testing).

## Stack

| Layer | Choice |
| --- | --- |
| App | Next.js 16 (App Router), React, TypeScript — on Vercel |
| Data & auth | Supabase — Postgres, Auth, Storage, Row-Level Security |
| Chain | `@stellar/stellar-sdk` (Soroban RPC, read-only attestation lookups) |
| Validation | `zod`, shared by env config and the profile form |
| QR | `qrcode` |
| Tests | Vitest + React Testing Library (unit/component), Supabase integration, Playwright e2e |
| Observability | structured JSON logging with field redaction, Sentry |

Requires Node.js 24+.

## Run it locally

```bash
npm install
npx supabase start          # prints API_URL, ANON_KEY, SERVICE_ROLE_KEY
cp .env.example .env.local   # paste the three values above
npm run dev
```

`supabase db reset` applies the migrations and seeds one demo patient
(`demo@henbridge.test` / `henbridge-demo-password`, card id
`11111111-1111-1111-1111-111111111111`). Then:

- `/signup` — create a card
- `/profile` — edit it, grab the QR and public link
- `/card/11111111-1111-1111-1111-111111111111` — the seeded patient's public page

## Testing

```bash
npm test                  # unit + component (Vitest, jsdom)
npm run test:integration  # RLS + RPC against a real local Supabase (needs `supabase start`)
npm run test:e2e          # Playwright: signup → save → QR → public view
npm run lint && npm run typecheck && npm run build   # what CI runs on every PR
```

Covered: public page leaks nothing beyond the chosen subset · RLS enforces patient-only read/write (plus a table GRANT RLS alone won't give) · QR output is valid and input-dependent · verified / not-yet-verified rendering · the attestation route's hash validation and stable shape · the `get_emergency_card` RPC (valid/unknown id, anon-callable, no column leakage) · offline banner + timestamp helpers.

**Offline is browser-only.** jsdom can't run a service worker, so verify by hand against a production build (registration is skipped in dev):

1. Open `/card/1111…1111` online — confirm `sw.js` is *activated* and the `henbridge-cards-v2` cache holds the URL.
2. Go offline, reload — the card renders with the amber "cached as of…" banner.
3. Offline, open an id you've never visited — you get "No cached card available", never a guess.
4. Online, open a bad id → `404`; offline reload → still `404` (errors aren't cached).
5. Back online, reload → banner gone, live data.

## Environment

`.env.example` is the full list; the cross-repo keys that matter:

| Key | Purpose |
| --- | --- |
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` | off-chain store; browser-safe, RLS-scoped |
| `SUPABASE_SERVICE_ROLE_KEY` | server-only, bypasses RLS — never shipped to the browser |
| `STELLAR_NETWORK_PASSPHRASE` | must match the network the contracts run on |
| `SOROBAN_RPC_URL` | Soroban RPC endpoint (testnet first) |
| `ATTESTATION_CONTRACT_ID` | deployed registry id; **unset ⇒ mock**, so the badge still renders locally |
| `STELLAR_HORIZON_URL` / `STELLAR_USDC_ISSUER` | Horizon endpoint and the exact accepted-USDC issuer |
| `CHW_INCENTIVE_POOL_ADDRESS` | payout source account the indexer watches |
| `PAYOUT_INDEXER_START_LEDGER` / `…_START_PAYMENT_CURSOR` | backfill origins used only until durable cursors exist |
| `PAYOUT_INDEXER_CRON_SECRET` | bearer secret for the scheduled `POST /api/internal/payout-indexer` |
| `NEXT_PUBLIC_SENTRY_DSN` / `SENTRY_AUTH_TOKEN` | error tracking; the token is optional (missing ⇒ source-map upload disabled, not a failed build) |

The payout mirror is refreshed by authenticated scheduled calls; see [`docs/chw-payout-indexer.md`](docs/chw-payout-indexer.md).

## Project layout

```
henbridge_frontend/
├── app/
│   ├── page.tsx                          landing page
│   ├── (public)/card/[id]/               public read-only emergency page (the QR target)
│   ├── (auth)/{signup,signin,signout}/   auth flows
│   ├── (auth)/profile/                   authenticated profile editor + QR/link
│   └── api/
│       ├── attestation/[recordHash]/     read-only attestation lookup
│       └── internal/payout-indexer/      scheduled CHW payout mirror refresh
├── lib/
│   ├── env.ts            zod-validated config      supabase/   client/server + DB types
│   ├── validation/       profile zod schema        qr/         QR generation
│   ├── attestation/      record-hash canonicalization + types
│   ├── stellar/          Soroban attestation lookup + payout indexer
│   └── logging/          structured logger with recursive field redaction
├── supabase/
│   ├── migrations/       profiles + RLS, get_emergency_card RPC, avatars bucket, payouts
│   └── seed.sql          demo patient fixture
├── proxy.ts              session refresh + route protection (Next 16's "middleware")
└── tests/                unit · integration (RLS/RPC) · e2e (Playwright)
```

## Observability rules (non-negotiable)

1. **Never log patient health data** — no field from the emergency model, no credentials, ever.
2. **Redaction is central.** `lib/logging/logger.ts` recursively and case-insensitively scrubs `name`, `age`, `date_of_birth`, `blood_group`, `genotype`, `allergies`, `medications`, `chronic_conditions`, `emergency_contacts`, `phone`, `relationship`, `language`, `photo_url`, `email`, `password`.
3. **One logging path.** Server code uses `logInfo` / `logError` from `@/lib/logging/logger` so everything is queryable JSON and reaches Sentry.

## Where we're headed

- **M0 — Public card (testnet):** profile editor, QR-reachable public page, offline service worker, full test coverage + CI. *Done, bar the Vercel/testnet deploy.*
- **M1 — Attestation:** registry deployed in `henbridge_contract`; this app already calls real `get_attestation` when `ATTESTATION_CONTRACT_ID` is set. Remaining: the contract-side verify path and the deploy.
- **M2 — Incentives:** wire USDC payout to attestation events. The durable payout indexer (Soroban + Horizon ingestion, crash-safe cursors, out-of-order reconciliation) already ships — see [`docs/chw-payout-indexer.md`](docs/chw-payout-indexer.md).
- **M3 — Pilot:** small supervised field pilot; measure verified cards and scans.
- **M4 — Mainnet + funding:** mainnet deploy; open the transparent funding pool.

## The HenBridge org

Four repos. When a change here touches a shared contract (below), flag it so the matching repo moves with it.

| Repo | What it holds | Language |
| --- | --- | --- |
| **`henbridge_frontend`** *(this repo)* | Patient + responder web app — public card, profile editor, QR, offline | TypeScript / Next.js |
| [`henbridge_backend`](https://github.com/HenBridge/henbridge_backend) | CHW field service — register records, submit attestations, queue USDC payouts | TypeScript / Next.js |
| [`henbridge_contract`](https://github.com/HenBridge/henbridge_contract) | Soroban (Rust): attester allowlist + attestation registry + multisig | Rust |
| [`henbridge_docs`](https://github.com/HenBridge/henbridge_docs) | Concept note, data model, threat model, privacy design, ADRs | Markdown |

```
henbridge_docs ──(data model, threat model)──▶ henbridge_frontend ◀──(register/attest)── henbridge_backend
                                                       │                                        │
                                          public card + QR + verified flag        submits hash + queues USDC payout
                                                       │                                        ▼
                                              responder scans, trusts   ◀────────── henbridge_contract (Soroban)
```

### Shared contracts (keep in sync)

- **Attestation shape** — `record_hash: BytesN<32>` · `attester: Address` · `timestamp: u64`. Defined here conceptually, mirrored by the Rust `Attestation` struct in `henbridge_contract`. Change one, change both.
- **Emergency data model** — the field list above is canonical; `henbridge_docs` mirrors it. Renaming a field is a cross-repo change.
- **Config keys** — `.env.example` is the source for the Stellar/Supabase keys shared across repos.

### For anyone (or any agent) dropped into a single repo

- Treat this section as the source of truth for *cross-repo* contracts; each repo's README owns its local conventions.
- Never route personal health data into an on-chain call — only hash, attester identity, and timestamp belong there. Hard invariant.
- Keep record/attestation field names byte-identical across TypeScript, Rust, and the docs — translation layers are where bugs breed.

## Privacy & compliance

- **Nigeria Data Protection Act (2023)** governs all personal data — consent, encryption, and minimal disclosure are designed in, not bolted on.
- Patients opt in to exactly what their public page shows.
- Nothing personal on-chain — only non-reversible hashes and attestations.

The full threat model (public card, attestation lookup, avatars bucket, authed editor, accepted tradeoffs) lives in [`henbridge_docs`](https://github.com/HenBridge/henbridge_docs).

## Contributing

Setup, migration steps, and code conventions are in [CONTRIBUTING.md](CONTRIBUTING.md). Security or privacy concern? [SECURITY.md](SECURITY.md).

## License

**MIT** — see [LICENSE](LICENSE).

## Disclaimer

HenBridge is an information aid — **not a medical device**, and not a substitute for professional judgment. A verified indicator means a registered health worker attested the record; it is not a clinical guarantee. The attending clinician owns the treatment decision.

---

<div align="center">

**HenBridge** — the facts that save a life, carried to the moment they're needed.

_Built on Stellar · open source · community-owned._

</div>
