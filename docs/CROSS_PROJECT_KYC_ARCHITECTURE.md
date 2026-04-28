---
doctitle: Cross-Project KYC Architecture
docdate: 2026-04-28
docversion: 0.2.0 (draft) — login layer revised for consumer-grade adoption
audience: Engineers building any b0ase product that gates features on KYC
related: docs/OPENCLAW_AGENT_SPEC.md
---

# Cross-Project KYC Architecture

**Status:** Draft v0.2.0 · 2026-04-28
**Goal:** A single KYC verification per user, reusable across every b0ase product without re-running Veriff.
**Scope:** bMovies · path401 · bit-sign · Bitcoin-Mint family · bmovies-exchange · OpenClaw agent manifests · future products.

> **Revision history**
> v0.1.0 (2026-04-28) — initial draft proposed Sigma Identity (BAP-based OAuth) as the login layer.
> v0.2.0 (2026-04-28) — login layer changed to **familiar OAuth (Google + Twitter) + server-derived ephemeral keys** for consumer reach. Sigma Identity remains supported as an opt-in "power user" alternative. KYC layer (bit-sign + Veriff) and consumption layer are unchanged.

---

## TL;DR

Today the b0ase portfolio runs **three parallel KYC systems** against the same Hetzner Postgres instance, with three different vendor integrations and three different schemas. Users could end up paying for Veriff three times to access products that should treat them as the same person.

The unified model collapses these to a **single canonical KYC issuer** (bit-sign.online) attached to **familiar consumer login** (Google + Twitter OAuth → server-derived ephemeral keys → BAP id minted server-side). Every other product becomes a *reader* — it asks "does this BAP id have a `kyc/veriff` strand at level X?" and routes accordingly. Veriff runs once. The strand is on-chain via path401. PII stays inside bit-sign.

The login layer deliberately uses **OAuth tools consumers already trust** (Google, Twitter) rather than bespoke wallet-based sign-in. The blockchain identity is created and managed *for* the user, not *by* the user.

```
        Google OAuth          Twitter OAuth         (HandCash for power users)
              │                     │                          │
              └─────────────────────┴──────────────────────────┘
                                    │
                                    ▼
                       Auth shim (per-product)
                        ├── derives a deterministic BSV keypair from the
                        │   OAuth subject id + a server-held HMAC seed
                        └── mints a BAP id on first login (path401 root)
                                    │
                                    ▼  (bap_id is now stable across products)
                                    │
                       bit-sign.online — KYC ISSUER
                        ├── runs Veriff once per bap_id
                        └── mints kyc/veriff strand on path401 chain
                                    │
                                    ▼  (strand readable on-chain)
                                    │
        ┌─────────────┬─────────────┴─────────────┬──────────────────┐
        │             │                           │                  │
   bMovies       Bitcoin Mint               NPGX/NPG Mint        bmovies-exchange
   (reader)      (reader)                   (reader)             (reader)
                                                                       │
                                                                       └── readKycStrand(bapId, lvl4) → bool
```

---

## 1. Current state (the problem)

### 1.1 Three KYC systems on the same database

A `\dt` on the Hetzner Postgres reveals three distinct schemas, all using Veriff, none aware of each other:

| System | Tables | Verb | Owner |
|---|---|---|---|
| **bMovies** | `user_kyc`, `bct_kyc_protect_verified` (mig 038) | Direct upload of ID + proof-of-address to bMovies' own DB; later switched to Veriff via `/api/kyc-start.ts`, `/api/kyc-webhook.ts` | bmovies-app/ |
| **bit-sign** | `bit_sign_identities`, `bit_sign_kyc_sessions`, `bit_sign_strands`, `bit_sign_signatures` | Veriff API → mint `kyc/veriff` strand on the user's $401 identity chain | bit-sign/ |
| **Mint family / path401-com** | `kyc_sessions`, `kyc_subjects` | Veriff session record + a kyc_subjects row carrying PII (first_name, last_name, document_type, dob…) | path401-com/ or one of the Mint apps |

**Consequence:** a user who signs up on bMovies, then mints an NPG issue on the Mint, then opens a $401 identity at bit-sign would face Veriff THREE TIMES at three URLs, paying ~80¢ each round, and ending with three disconnected verification records that don't recognise each other.

### 1.2 PII landed in three places

- `user_kyc` stores ID document base64, proof-of-address base64, full name, DOB.
- `kyc_subjects` stores first_name, last_name, dob, document_type, document_country.
- `bit_sign_kyc_sessions` stores the Veriff response payload (JSON blob with PII).

This fans out the GDPR / data-minimisation surface area across three codebases and three teams of (one) engineer. Concentrating PII in **one** system reduces the attack surface and makes data-deletion requests answerable.

### 1.3 No portable login

Each product has its own login system. There is no concept of "I am the same person across these surfaces" without re-doing identity work each time.

---

## 2. The unified model

### 2.1 Three layers, three responsibilities

**Layer 1 — Login (who are you)**
**Familiar consumer OAuth — Google + Twitter as the primary path.** When a user logs in for the first time, the auth shim derives a **deterministic BSV keypair** from the OAuth subject id + a server-held HMAC seed, and mints a **BAP id** (path401 root inscription) for them. The user never sees a wallet, never installs anything, never copies a seed phrase. From the second login forward, the same OAuth subject deterministically resolves to the same keypair and same `bap_id`.

This is the **Privy / Magic / HandCash model**: the user logs in with tools they already trust, the platform runs the blockchain mechanics on their behalf. Consumer adoption beats sovereignty for the 99%; the 1% who want self-custody can opt into HandCash or Sigma Identity instead (see §2.3).

**Layer 2 — KYC issuance (have you been verified)**
bit-sign.online runs Veriff once per `bap_id` and, on approval, mints a `kyc/veriff` **strand** on the user's $401 identity chain. PII (document image, DOB, full name) is held by bit-sign and Veriff — *no other product receives it*. Levels match the existing $401 hierarchy: `none | basic | enhanced | full`. **Unchanged from v0.1**: bit-sign is still the canonical issuer and the only product that integrates with Veriff.

**Layer 3 — KYC consumption (does this user qualify)**
Every other product (bMovies, the Mints, exchange, future apps) becomes a *reader*. It asks path401 — *"does bap_id X have a `kyc/veriff` strand at level Y or higher?"* — and gates features on the answer. No vendor integration, no PII storage, no re-verification. **Unchanged from v0.1**: `@path401/kyc-reader` is the lib, `kyc_strands` is the SQL view, both ship in path401 monorepo.

### 2.2 Why this works (revised)

- **Consumer reach.** The instinct to use Sigma Identity was architecturally clean but adoption-blocked. Every BSV-native app launched in 2024-2026 with wallet-only sign-in has plateaued at four-figure user counts. Google + Twitter OAuth is what every consumer has — the friction is zero. The 90% case becomes invisible to the user; only the 10% who want sovereignty go through extra steps.
- **bit-sign already has the complete Veriff integration** — `/api/bitsign/kyc/veriff/start` + HMAC-verified webhook + strand minting. Unchanged.
- **path401's protocol already defines KYC as a strand** — see `Path401/packages/core/src/kyc/index.ts`. Unchanged.
- **It composes with OpenClaw** — `fiduciary_kyc_handle` becomes `kyc:bap:<bap_id>:lvl4`. Whether the BAP id was created from a Google login or a self-sovereign HandCash wallet is invisible to consumers of the strand. The marketplace just sees a verified BAP id.
- **Recovery story is trivial.** "Lost your account?" → "Log in with Google again." Same OAuth subject id → same derived keypair → same BAP id. No seed phrases, no recovery codes, no support tickets about lost wallets.

### 2.3 Power-user / self-custody escape hatches

The login layer is **multi-provider** by design. Consumers default to Google/Twitter; power users opt into:

- **HandCash** — already wired in bMovies, path401-com. Custodial BSV wallet with phone-number-style handles. Pre-existing relationship in the user's portfolio.
- **Sigma Identity** — for users who want OAuth-2.1-with-BAP and don't want any custodial component. Plugin already integrates cleanly (`@sigma-auth/better-auth-plugin`).
- **Direct WIF / hardware wallet** — for users who want to bring their own keypair and prove control via signature.

Critically, **all four paths produce the same `bap_id`-shaped artefact** for downstream consumers. A bMovies feature gated on KYC doesn't care which login path the user took; it only cares that a strand exists for the BAP id presented in the session. This keeps the architecture composable while accepting that consumer onboarding and power-user onboarding are different journeys.

### 2.4 The custody question (read carefully)

Server-derived keys mean the platform is **technically able** to sign on behalf of the user. That carries regulatory weight — in many jurisdictions, custodying user keys that hold tradeable financial instruments is a regulated activity (money transmitter / VASP).

Three mitigations make this defensible:

1. **The keys hold no balance by default.** They sign identity strands (free) and tokens minted *for* the user (the user's own creative work). They do not hold investor funds. There is no "withdraw" button on a server-derived BAP key — it's an *identity*, not a wallet.
2. **Tradeable assets settle to addresses the user controls separately.** When a user receives royalty payouts, those route to a payout destination they configure (their HandCash handle, their own BSV/ETH/SOL address). The server-derived key never touches the funds.
3. **Power-user paths bypass custody entirely.** A user who logs in with HandCash or Sigma is non-custodial from minute one. They get the same BAP id shape; their keys are never on our servers.

This isn't a complete legal answer — it's a structural one. Counsel review (already on the legal-tracker for the comfort letter) needs to confirm the framing before we ship to UK / US / EU end users.

---

## 3. Concrete architecture

### 3.1 The flow for a new user signing up on bMovies

```
1. User visits bmovies.online/studios-signup
2. Clicks "Sign in with Sigma" (alongside "Sign in with X" + "Sign in with HandCash")
3. Browser redirects to auth.sigmaidentity.com
4. User authenticates with their Bitcoin wallet (BAP)
5. Sigma redirects back with code; bMovies exchanges it for { bap_id, pubkey, email? }
6. bMovies looks up the bap_id in path401:
     readKycStrand(bap_id, requiredLevel: 'full') → null
7. KYC required to publish on-chain. bMovies redirects to:
     bit-sign.online/kyc/start?bap_id=<X>&return_to=<bMovies-callback>
8. bit-sign creates a Veriff session, captures the result via webhook,
   and on approval mints a kyc/veriff/full strand on the user's identity chain.
9. bit-sign redirects user back to bMovies callback.
10. bMovies re-checks the strand → present → enables on-chain publish.
```

Crucially, **steps 7-9 only happen for the first product the user hits that requires KYC.** When the same user later visits NPG Mint or Bitcoin-Mint, step 6 returns `kyc/veriff/full` and the KYC step is skipped.

### 3.2 The shared schema

A single `kyc_strands` view sits over the canonical strand storage — readable by every product's service-role connection but not directly mutable.

```sql
-- In path401-com or Path401 monorepo's shared schema:
CREATE VIEW kyc_strands AS
SELECT
  bap_id,
  level,                      -- 'basic' | 'enhanced' | 'full'
  provider,                   -- 'veriff' | 'manual' | future
  attested_at,
  expires_at,
  attestation_inscription_txid,
  -- intentionally NO PII columns. Lookups, not personal data.
  CASE
    WHEN expires_at IS NOT NULL AND expires_at < NOW() THEN 'expired'
    WHEN level IS NULL THEN 'none'
    ELSE 'active'
  END AS status
FROM bit_sign_strands
WHERE strand_type = 'kyc' AND strand_subtype = 'veriff';
```

PII (document images, full names, DOB) live ONLY in `bit_sign_kyc_sessions.veriff_response` (JSONB), encrypted at rest and pruned on a 30-day rolling window after attestation. Other products NEVER touch this table.

### 3.3 The shared client library

`@path401/kyc-reader` ships in the path401 monorepo (`packages/kyc-reader`). Read-only checker that any product imports to gate features. Already built — see commit `ad0da17` in path401.

```typescript
import { createKycReader } from '@path401/kyc-reader';

const kyc = createKycReader({
  supabaseUrl: process.env.SUPABASE_URL!,
  supabaseServiceRoleKey: process.env.SUPABASE_SERVICE_ROLE_KEY!,
});

// Pure read — returns the strand or null
const strand = await kyc.read({ bapId: '1abc...xyz' });

// Gate helper — throws KycRequiredError if missing
await kyc.require({ bapId: '1abc...xyz' }, 'enhanced');
```

The reader accepts any of: `userHandle` (HandCash), `rootTxid`, `bapId`, `bsvAddress`. The `bapId` path returns null today and starts working when `bit_sign_identities.bap_id` is added. Code written today is forward-compatible.

### 3.4 The auth-shim that derives keys from OAuth (NEW in v0.2)

The new piece this revision adds. A small package — proposed `@path401/auth-shim` — that:

1. Receives the OAuth callback (Google, Twitter, future).
2. Extracts the **stable subject id** the provider returns (`sub` claim — Google's stable user id, Twitter's user id).
3. Derives a deterministic ed25519 keypair from `HMAC-SHA256(serverSeed, "<provider>:<subject_id>")`.
4. Mints (or fetches) a path401 root inscription for that keypair on first call → produces a stable `bap_id`.
5. Returns `{ bapId, derivedAddress, providerSubject }` to the caller.

The same provider+subject always derives the same keypair, so re-login = same identity. The HMAC seed lives in **one place per environment** (production: a single seed shared across all products via the secret manager; dev: per-developer seed) so any product can resolve any other product's user to the same `bap_id`.

```typescript
import { resolveBapFromOAuth } from '@path401/auth-shim';

// In a /auth/google/callback handler:
const { bapId, derivedAddress } = await resolveBapFromOAuth({
  provider: 'google',
  subjectId: googleProfile.sub,
  email:     googleProfile.email,         // optional, attached as a non-PII strand label
});

// Now `bapId` is the user's stable cross-product identity. Use it in:
// - bct_accounts.bap_id (bMovies)
// - clawdex_holders.bap_id (Claw-Dex)
// - any future product's user table
// And read KYC via @path401/kyc-reader against the same id.
```

Out-of-scope for v0.2 of this doc — the shim's full design (key escrow, recovery, rotation under provider-id-change scenarios) is its own doc. This section just sketches the interface so consumers can plan against it.

### 3.4 OpenClaw manifest mapping

Per the v2 spec at `path402/docs/OPENCLAW_AGENT_SPEC.md` §4, the `owner.fiduciary_kyc_handle` is intentionally opaque so different identity authorities can be plugged in. With this architecture, the canonical handle becomes:

```
kyc:bap:<bap_id>:<level>:<expiry-yyyy-mm>
```

Example: `kyc:bap:1abc...xyz:full:2027-04`

Resolvable by any consumer:

```
GET https://api.path401.com/strand?bap_id=1abc...xyz&type=kyc
→ { level: "full", attested_at: "...", expires_at: "2027-04-...", inscription_txid: "..." }
```

This replaces the placeholder `kyc:b0ase:518f51fe` used in NPGX's migrated manifests — that handle is bMovies-account-specific and can't be resolved by Claw-Dex or any third party.

---

## 4. Migration plan (revised in v0.2)

The goal is to move every product to "Google/Twitter OAuth → derived keypair → BAP id → path401 strand for KYC" without disrupting existing users. Each step is independently shippable. **HandCash + Sigma Identity remain available as power-user paths**, not the primary funnel.

### Step 1 — Build `@path401/auth-shim` (~1 day)

Sibling package to `@path401/kyc-reader` (already shipped). Exposes `resolveBapFromOAuth({ provider, subjectId, email? })` → `{ bapId, derivedAddress }`. Deterministic key derivation via HMAC-SHA256 over a server-held seed. Lives in `Path401/packages/auth-shim/`.

### Step 2 — Add Google + Twitter OAuth login per product (~1 day per product)

Each product adds two consumer-grade login buttons alongside the existing power-user options:

1. **bMovies** — Google + Twitter alongside the existing HandCash + X (already wired) + email. The new flow on success calls `resolveBapFromOAuth` and writes `bct_accounts.bap_id`.
2. **bmovies-exchange** — same.
3. **The Mint family** — Google + Twitter alongside (currently auth-less). The Mint's Privacy Principle is preserved by NOT writing the OAuth subject anywhere — only the `bap_id`. The mint never knows who the OAuth user is.
4. **path401-com** — already has Google/Twitter as OAuth *strand* sources; this step routes them through the shim so they additionally produce a `bap_id` and a derived keypair, not just attached strands.
5. **bit-sign** — same auth path as the others. The `bit_sign_identities` table gains a `bap_id` column linking to the path401 root.

### Step 3 — Schema link columns per product (~1 day total)

A nullable `bap_id text` column on the user table of every product:

- `bct_accounts.bap_id` (bMovies)
- `clawdex_holders.bap_id` (Claw-Dex)
- `bit_sign_identities.bap_id` (bit-sign)
- `kyc_subjects.bap_id` (path401-com / mint shared, if it stays around)

Indexes: `(bap_id) WHERE bap_id IS NOT NULL` on each table.

This unlocks the `bapId` lookup path in `@path401/kyc-reader` that currently returns null.

### Step 4 — Read KYC from `@path401/kyc-reader` in every product (~1 day per product)

For each product, replace the existing KYC check with `kyc.require({ bapId }, 'enhanced')` or similar:

- **bMovies** — `api/_lib/require-kyc.ts` becomes a 10-line wrapper.
- **bmovies-exchange** — add a KYC gate to share-listing flows.
- **The Mints** — gate on-chain mint operations behind a strand check.
- **NPGX agents on Claw-Dex** — when an agent token gets registered, the listing reads `manifest.owner.fiduciary_kyc_handle`, resolves to the strand, refuses listing if not found.

### Step 5 — Deprecate duplicate KYC tables (slow, 6+ months)

Once every product reads from `kyc_strands`, the duplicate tables can be archived:

- `user_kyc` (bMovies) — strand-only after migration of existing rows.
- `kyc_subjects` (path401-com / Mint) — view-backed shim during transition, then deleted.
- `bct_kyc_protect_verified` (bMovies migration 038) — replaced by view.

PII is purged from the deprecated tables after the cutover; only `bit_sign_kyc_sessions` retains the Veriff-response blob, and that's pruned on a 30-day rolling window.

### Step 6 — Single Veriff tenant + secret rotation (1 day, infra)

bit-sign owns the only `VERIFF_API_KEY` and `VERIFF_WEBHOOK_SECRET` env vars across the portfolio. Other products' Veriff env vars get retired. Stripe-style: one billing account, one webhook source, one set of dashboards to monitor.

---

## 5. Open questions (revised in v0.2)

These need answers before any code lands. None of them are blockers for the spec; they're product decisions:

1. **Pricing — does bit-sign charge for KYC, and at what level?** bMovies currently bundles 80¢ Veriff + on-chain mint into a $0.99 publish step. If KYC moves to bit-sign, does bit-sign charge $0.99 once and other products see it free? Or does bit-sign charge $0.50 (Veriff at cost) and downstream products stack their own fees?

2. **Strand ownership — who pays the BSV inscription fee for the kyc/veriff strand?** The user (covered by their initial $0.99) or the platform (eaten as ops cost)?

3. **Re-verification cadence — annually, or only on a compliance trigger?** Veriff itself recommends annual re-checks; the spec currently sets `expires_at` to attested + 12 months.

4. **HMAC seed management.** The auth-shim's deterministic key derivation depends on a server-held HMAC seed. Compromise of the seed = ability to forge any user's identity. Where does it live? Recommendation: a single secret per environment, stored in the Vercel/Hetzner secret manager, rotated annually with a 6-month overlap window during which BOTH the old and new seed are accepted (so existing users' deterministic keys keep resolving). Detailed rotation flow is its own ticket.

5. **OAuth subject id stability.** Google's `sub` claim is stable forever. Twitter's user id is stable. Email is **NOT stable** (users change emails). The shim MUST derive from `sub`/`user_id`, never email. Documented invariant — review every OAuth integration to confirm.

6. **Custody framing for counsel.** Section 2.4 lays out the structural argument for why server-derived keys aren't custody-of-funds. Counsel review (already on the legal-tracker for the comfort letter) needs to bless this framing before we ship to UK/US/EU end users. A negative opinion would push us to a non-custodial model (threshold keys, user-side signing) — possible but materially more complex.

7. **Does the legacy `kyc_subjects` table belong to a Mint or to path401-com?** The schema says it stores PII — *if it's the Mints*, that violates their Privacy Principle. Track down which app populates it and either retire the table or move the PII into bit-sign's storage.

8. **Power-user identity bridging.** A user who first signs up via Google (gets a server-derived BAP id) and later wants to take self-custody by importing into HandCash needs a clean bridge. Design TBD: probably a "claim into wallet" flow that signs over the on-chain identity to a user-controlled key while keeping the bap_id stable.

9. **Sigma Identity status — keep, drop, or defer?** v0.2 demotes Sigma from primary to power-user-opt-in. We can ship without integrating it at all and add later if real demand emerges. Recommendation: defer — don't integrate `@sigma-auth/better-auth-plugin` until at least one user actively asks.

---

## 6. What to build first (revised in v0.2)

Foundation (already done):
1. ✅ **`@path401/kyc-reader` shared package** — shipped at `Path401/packages/kyc-reader` (commit `ad0da17`).
2. ✅ **`kyc_strands` Postgres view** — applied to Hetzner.

New foundation work:
3. **`@path401/auth-shim` package** — sibling of kyc-reader. Implements `resolveBapFromOAuth`. Includes the HMAC-seed key derivation, BAP root inscription on first call, idempotent re-resolution. ~1-2 days.
4. **`bit_sign_identities.bap_id` column** + index. Backfill existing rows by deriving from their HandCash handle. ~half a day.
5. **OpenClaw v2 spec amendment** — formalise `kyc:bap:<id>:<level>` as the canonical `fiduciary_kyc_handle`. ~10 minutes.

Per-product integration (each ~1 day):
6. **bMovies** — Google + Twitter login buttons → auth-shim → `bct_accounts.bap_id` populated → `kyc-reader` replaces existing KYC checks.
7. **bmovies-exchange** — same pattern, share-listing gate added.
8. **The Mints** — same pattern, mint operations gated.
9. **path401-com** — strand sources route through shim so they additionally produce a bap_id.
10. **bit-sign** — primary login becomes Google/Twitter; KYC start endpoint accepts `bap_id` + `return_to`.

Cleanup:
11. **Re-migrate NPGX manifests** — replace placeholder KYC handle with real `kyc:bap:<...>` once b0ase is logged in via the shim. ~minutes.
12. **Deprecate parallel KYC tables** — after a 6-month soak, drop `user_kyc`, `kyc_subjects`, `bct_kyc_protect_verified`.

After steps 3-5, the foundation is complete. Steps 6-10 can happen in any order, in parallel across products.

---

## 7. References

- **OpenClaw Agent Manifest** — `path402/docs/OPENCLAW_AGENT_SPEC.md` (this doc's sibling)
- **@path401/kyc-reader** — `Path401/packages/kyc-reader/` (shipped, commit `ad0da17`)
- **bit-sign Veriff integration** — `bit-sign/src/app/api/bitsign/kyc/veriff/start/route.ts`, `bit-sign/src/app/api/webhooks/veriff/route.ts`
- **bMovies KYC** — `bmovies-app/api/kyc-start.ts`, `bmovies-app/migrations/038_kyc_protect_verified.sql`, `bmovies-app/api/_lib/require-kyc.ts`
- **path401 KYC service** — `Path401/packages/core/src/kyc/index.ts` (currently a stub)
- **path401 OAuth strand pattern** — `path401-com/app/api/auth/strand/[provider]/` (existing reference for OAuth → on-chain inscription)
- **HandCash Connect SDK** — `@handcash/handcash-connect` (existing power-user path)
- **Sigma Identity docs** — https://sigmaidentity.com/docs/introduction/quickstart (deferred / power-user opt-in)
- **BAP** — Bitcoin Attestation Protocol, the underlying identity primitive (works with any keypair, regardless of how derived)
