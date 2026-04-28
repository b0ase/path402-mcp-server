---
doctitle: Cross-Project KYC Architecture
docdate: 2026-04-28
docversion: 0.1.0 (draft)
audience: Engineers building any b0ase product that gates features on KYC
related: docs/OPENCLAW_AGENT_SPEC.md
---

# Cross-Project KYC Architecture

**Status:** Draft v0.1.0 · 2026-04-28
**Goal:** A single KYC verification per user, reusable across every b0ase product without re-running Veriff.
**Scope:** bMovies · path401 · bit-sign · Bitcoin-Mint family · bmovies-exchange · OpenClaw agent manifests · future products.

---

## TL;DR

Today the b0ase portfolio runs **three parallel KYC systems** against the same Hetzner Postgres instance, with three different vendor integrations and three different schemas. Users could end up paying for Veriff three times to access products that should treat them as the same person.

The unified model collapses these to a **single canonical KYC issuer** (bit-sign.online) attached to a **single canonical login identity** (Sigma Identity, BAP-based). Every other product becomes a *reader* — it asks "does this BAP id have a `kyc/veriff` strand at level X?" and routes accordingly. Veriff runs once. The strand is on-chain via path401. PII stays inside bit-sign.

```
                 Sigma Identity (auth.sigmaidentity.com)
                       └── returns bap_id + pubkey on login
                                    │
                                    ▼
                       bit-sign.online — KYC ISSUER
                        ├── runs Veriff once per bap_id
                        └── mints kyc/veriff strand on path401 chain
                                    │
                                    ▼ (strand readable on-chain)
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
Sigma Identity (`auth.sigmaidentity.com`, by Luke Rohenaz / b-open-io) provides OAuth 2.1 + OIDC sign-in backed by **BAP** (Bitcoin Attestation Protocol) identities. Each user has a stable `bap_id` + `pubkey`. Plugin: `@sigma-auth/better-auth-plugin`. Same model as "Sign in with Google" / "Sign in with Apple" — but the identity primitive is on-chain.

**Layer 2 — KYC issuance (have you been verified)**
bit-sign.online runs Veriff once per `bap_id` and, on approval, mints a `kyc/veriff` **strand** on the user's $401 identity chain. PII (document image, DOB, full name) is held by bit-sign and Veriff — *no other product receives it*. Levels match the existing $401 hierarchy: `none | basic | enhanced | full`.

**Layer 3 — KYC consumption (does this user qualify)**
Every other product (bMovies, the Mints, exchange, future apps) becomes a *reader*. It asks path401 — *"does bap_id X have a `kyc/veriff` strand at level Y or higher?"* — and gates features on the answer. No vendor integration, no PII storage, no re-verification.

### 2.2 Why this works

- **Sigma Identity is built for this** — it returns a stable `bap_id` per user that any consumer can use as the canonical identifier.
- **bit-sign already has the complete Veriff integration** — `/api/bitsign/kyc/veriff/start` + HMAC-verified webhook + strand minting. We don't have to build it again.
- **path401's protocol already defines KYC as a strand** — see `Path401/packages/core/src/kyc/index.ts`. The shape is already specified; only the read API needs to be exposed.
- **It composes with OpenClaw** — see `OPENCLAW_AGENT_SPEC.md` §4. The `fiduciary_kyc_handle` field becomes `kyc:bap:<bap_id>:lvl4` instead of an opaque per-product handle. Any marketplace can resolve it.

### 2.3 Why we accept the dependency on Sigma Identity

Sigma Identity is **not our project**. Same as `bopen.io` and `1sat.market` — both Luke Rohenaz' projects, both heavily relied on. The pattern is established: when someone in the BSV ecosystem builds a clean primitive that fits, we use it rather than re-invent. The cost of dependency is real (Sigma Identity outages would block our login) but the cost of rebuilding it is bigger and the result would be worse.

Mitigation: bMovies / bit-sign keep their existing email + HandCash login paths as fallbacks. Sigma is *added* alongside, not the only option.

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

A small `@b0ase/kyc-reader` package (in path401 monorepo or a new shared lib repo) exposes:

```typescript
import { readKycStrand, requireKyc } from '@b0ase/kyc-reader';

// Pure read — returns the strand or null
const strand = await readKycStrand(bapId, { minLevel: 'full' });

// Gate helper — throws if not present
await requireKyc(bapId, { minLevel: 'enhanced' });
```

Implementations call path401's MCP server (`packages/core/src/mcp.ts` already has `check_kyc`) or the Postgres view directly with a service-role connection.

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

## 4. Migration plan

The goal is to move every product to "Sigma Identity for login + path401 strand for KYC" without disrupting existing users. Each step is independently shippable.

### Step 1 — Sigma Identity sign-in alongside existing auth (1-2 days per product)

Add `@sigma-auth/better-auth-plugin` to:
1. **bMovies** — alongside Supabase email + HandCash + X OAuth.
2. **bmovies-exchange** — same.
3. **The Mint family** — adds login since the Mint currently has no auth (the Privacy Principle says "never save user data" but a *signed-in* user state is compatible — it just means we know which BAP signed, not who they are).
4. **path401-com** — already has the closest thing (HandCash + OAuth strands); Sigma slots in as another strand source.
5. **bit-sign** — already on similar footing; switch its primary login to Sigma so the BAP id from login matches the BAP id strands attach to.

### Step 2 — Read KYC from path401 in every product (~1 day per product)

For each product, replace the existing KYC check with `readKycStrand(bapId, ...)`:

- **bMovies** — `api/_lib/require-kyc.ts` becomes a wrapper around `readKycStrand` (~10 lines).
- **bmovies-exchange** — currently has no KYC gate; add one for share-listing flows.
- **The Mints** — gate on-chain mint operations behind a strand check. Tokens can't be issued by anonymous addresses (legal requirement).
- **NPGX agents on Claw-Dex** — when an agent token gets registered, the listing reads `manifest.owner.fiduciary_kyc_handle`, resolves it to the strand, refuses listing if not found.

### Step 3 — Deprecate duplicate KYC tables (slow, 6+ months)

Once every product reads from `kyc_strands`, the duplicate tables can be archived:

- `user_kyc` (bMovies) — strand-only after migration of existing rows.
- `kyc_subjects` (path401-com / Mint) — view-backed shim during transition, then deleted.
- `bct_kyc_protect_verified` (bMovies migration 038) — replaced by view.

PII is purged from the deprecated tables after the cutover; only `bit_sign_kyc_sessions` retains the Veriff-response blob, and that's pruned on a 30-day rolling window.

### Step 4 — Single Veriff tenant + secret rotation (1 day, infra)

bit-sign owns the only `VERIFF_API_KEY` and `VERIFF_WEBHOOK_SECRET` env vars across the portfolio. Other products' Veriff env vars get retired. Stripe-style: one billing account, one webhook source, one set of dashboards to monitor.

---

## 5. Open questions

These need answers before any code lands. None of them are blockers for the spec; they're product decisions:

1. **Pricing — does bit-sign charge for KYC, and at what level?** bMovies currently bundles 80¢ Veriff + on-chain mint into a $0.99 publish step. If KYC moves to bit-sign, does bit-sign charge $0.99 once and other products see it free? Or does bit-sign charge $0.50 (Veriff at cost) and downstream products stack their own fees?

2. **Strand ownership — who pays the BSV inscription fee for the kyc/veriff strand?** The user (covered by their initial $0.99) or the platform (eaten as ops cost)?

3. **Re-verification cadence — annually, or only on a compliance trigger?** Veriff itself recommends annual re-checks; the spec currently sets `expires_at` to attested + 12 months.

4. **What about users who don't (yet) have a BAP id?** Sigma Identity creates one on first login if needed (the BAP standard supports lazy creation). But for users coming in via an existing email+password account on bMovies, there's a binding step — link their bap_id once they sign in with Sigma. Schema-wise this means `bct_accounts` gains a nullable `bap_id` column.

5. **Does the legacy `kyc_subjects` table belong to a Mint or to path401-com?** The schema says it stores PII — *if it's the Mints*, that violates their Privacy Principle. Track down which app populates it and either retire the table or move the PII into bit-sign's storage.

6. **Should Sigma's Better Auth integration replace Supabase auth, or sit alongside?** bMovies depends on `auth.uid()` in RLS policies. Switching auth providers is a bigger move than the cross-project KYC question. Defer — for v1 we can run Sigma sign-in *in addition to* existing Supabase auth, with a `bct_accounts.bap_id` link column.

---

## 6. What to build first (proposed order)

1. **`@b0ase/kyc-reader` shared package** — a 50-line library that any product can import to read strands. Lives in path401 monorepo. ~half a day.
2. **`kyc_strands` Postgres view** — ~10 lines of SQL. ~30 minutes.
3. **Sigma Identity sign-in on bMovies** as a third login option, with `bct_accounts.bap_id` linking column. ~1 day.
4. **bit-sign's KYC flow accepts a `bap_id` parameter and `return_to` URL** so any product can deep-link to KYC and bring the user back. ~half a day.
5. **OpenClaw v2 spec amendment** — formalise `kyc:bap:<id>:<level>` as the canonical `fiduciary_kyc_handle`. ~10 minutes.
6. **Re-migrate NPGX's 26 manifests** — replace `kyc:b0ase:518f51fe` with the proper `kyc:bap:<...>` handle once the operator has a BAP id. ~minutes (re-run migration script).

After steps 1-4, every new product can plug in immediately. After step 5, the agent manifest spec is consistent with this architecture. Step 6 is cleanup.

---

## 7. References

- **OpenClaw Agent Manifest** — `path402/docs/OPENCLAW_AGENT_SPEC.md` (this doc's sibling)
- **bit-sign Veriff integration** — `bit-sign/src/app/api/bitsign/kyc/veriff/start/route.ts`, `bit-sign/src/app/api/webhooks/veriff/route.ts`
- **bMovies KYC** — `bmovies-app/api/kyc-start.ts`, `bmovies-app/migrations/038_kyc_protect_verified.sql`, `bmovies-app/api/_lib/require-kyc.ts`
- **path401 KYC service** — `Path401/packages/core/src/kyc/index.ts` (currently a stub)
- **Sigma Identity docs** — https://sigmaidentity.com/docs/introduction/quickstart
- **Sigma Auth plugin** — https://github.com/b-open-io/better-auth-plugin
- **BAP** — Bitcoin Attestation Protocol, the on-chain identity primitive Sigma builds on
