# Republike On-Chain Migration Strategy

## Premise

Republike is a civic platform. Its users are citizens, not customers. Their identity, their reputation, their participation in governance, and their data belong to them — not to the platform that facilitates their interactions.

Today, all of this lives in a database controlled by us. That's a liability, not a feature. A compromised database, a rogue employee, a regulatory overreach, or a simple infrastructure failure can expose, corrupt, or erase everything a citizen has built on the platform. We've already seen what happens when third-party platforms are breached (the recent Vercel incident is a reminder that no stack is immune).

The on-chain migration is the project's answer to that structural vulnerability. It is not about "going web3" or adding blockchain for its own sake. It's about moving the proof of who our citizens are — and what they've earned — to a layer that we cannot unilaterally tamper with, lose, or be compelled to hand over in ways that violate their sovereignty.

---

## Principles

**1. Users never see the chain.** No wallet setup, no seed phrases, no gas fees, no MetaMask. The server derives non-custodial wallets deterministically from each user's identity — the keys belong to the user, the server holds them on their behalf until they choose to take ownership. The blockchain is infrastructure, not interface.

**2. The app doesn't change.** The read path (how the app checks "can this user do this?") stays exactly as it is. The write path (how a role gets granted or revoked) is what migrates: from a direct database write to an on-chain transaction confirmed by an indexer that writes to the same database. The app reads from the database either way.

**3. Identity first, everything else follows.** Roles and identity are the foundation. Once they're on-chain, rewards, governance, reputation, and ownership become natural extensions — each one a separate, incremental step, not a monolithic migration.

**4. Zero-knowledge by design.** A citizen should be able to prove "I am a Founding Citizen" without revealing which Founding Citizen they are. This enables anonymous moderation, private governance votes, and reputation attestations that carry weight without carrying personal data.

**5. Reversibility.** Every step of the migration is designed so that if the chain layer fails, the platform falls back to the database and keeps working. The chain is the source of truth when available; the database is the operational copy always.

---

## Architecture

```
CURRENT STATE (off-chain only)
──────────────────────────────
Stripe webhook ──→ Prisma write ──→ UserContract table
                                         │
                                    resolveAccess()
                                         │
                                    App reads role + capabilities


FUTURE STATE (hybrid)
─────────────────────
Stripe webhook ──→ Server signs tx ──→ Smart contract (on-chain)
                                            │
                                       emits RoleGranted event
                                            │
                                       Indexer picks it up
                                            │
                                       Prisma write (mirror)
                                            │
                                       resolveAccess()
                                            │
                                       App reads role + capabilities
                                       (unchanged)
```

The `UserContract` Prisma model is already shaped to mirror what the on-chain contract stores: role, start date, expiry, source, status, plus nullable fields for `walletAddress`, `txHash`, and `onChainConfirmed` that will be populated when the chain layer comes online.

---

## Phase 1 — Immediate (in progress)

### Goal: establish the off-chain contract primitives

**Status: built, not deployed.**

What's done:
- `AccessRole` enum: FREE_CITIZEN → CITIZEN → FOUNDING_CITIZEN → FOUNDING_FATHER → FOUNDING_CONSUL → TEAM_MEMBER
- `UserContract` model: grants a role to a user for a period. Contracts are granted, revoked, expired, superseded — never deleted (audit trail).
- 13 capabilities mapped to roles with inheritance. Pure code, no database.
- `capabilityProcedure` tRPC middleware: enforces capabilities at the API layer with structured errors.
- Stripe/Google Play/App Store integration: subscription events automatically grant and revoke contracts.
- Session enrichment: every authenticated request carries the user's resolved role and capabilities.
- Free tier: users can enter the platform as FREE_CITIZEN with limited capabilities.
- Admin panel: full contract management (grant, revoke, view per user, capability breakdown).

What remains:
- Deploy to production with backfill of existing users.
- Validate the full flow end-to-end in staging.
- Unlock the ~2,000 users currently stuck at the paywall.

### Why this matters for the on-chain migration

The contract model IS the on-chain model. When we deploy the smart contract, its storage layout will mirror `UserContract` exactly. The migration from "off-chain contracts" to "on-chain contracts" is a write-path change, not a schema change. The app, the admin panel, the mobile client — none of them change.

---

## Phase 2 — Near-term (weeks)

### Goal: non-custodial wallets + L2 selection

**2a. Non-custodial wallet service**

Every user gets a blockchain address derived deterministically from a master seed + their user ID (HD wallet derivation). The private keys never leave the server. The user never interacts with crypto concepts.

- `deriveWallet(userId) → { address, signer }`
- Master seed stored in KMS (not the database, not environment variables)
- User's address populated in `UserContract.walletAddress`
- If a user ever wants to "take ownership" of their on-chain identity, we hand them the derived key. But that's a future feature, not a requirement.

**2b. L2 selection**

The chain must align with the platform's principles:

| Criterion | What it means for Republike |
|-----------|---------------------------|
| Fee delegation | Users never see gas. The platform sponsors transactions. |
| Accountable governance | The chain itself should be governed by known, vetted entities — not anonymous token holders. Mirrors Republike's citizen-based governance. |
| Identity primitives | Protocol-level support for verified identity without exposing personal data. |
| European regulatory posture | Compatible with GDPR-era thinking about data sovereignty. |
| Sustainability | Low energy, long-term viability. Not a chain that disappears in 2 years. |

Candidates under evaluation:
- **Optimism**: Citizens' House governance model, Retroactive Public Goods Funding, OP Stack open source.
- **Gnosis Chain**: European (Berlin), community-owned, xDAI stablecoin gas (predictable costs), true decentralization.
- **VeChain**: Native fee delegation (MPP protocol-level, not a hack), authority-based governance (101 vetted nodes), VeVID identity primitives, designed for regulated environments.

---

## Phase 3 — Mid-term (months)

### Goal: on-chain role registry + indexer

**3a. Smart contract: RepublikeRoles**

~200 lines of Solidity. Pure role management:
- `grantRole(address, role, vestingEnd)` — operator-only
- `revokeRole(address, role)` — operator-only
- `hasRole(address, role) → bool` — view function, free to call
- Events: `RoleGranted`, `RoleRevoked`

No token economics, no DeFi complexity. The contract knows one thing: this address holds this role until this date.

**3b. Server signing service**

When a Stripe webhook fires (subscription created/deleted), instead of writing directly to Prisma, the server signs a transaction on the user's behalf (using their derived wallet) and submits it to the smart contract. The operator wallet pays gas.

Fallback: if the chain is slow or down, write directly to Prisma with `onChainConfirmed = false`. A reconciliation job picks up unconfirmed rows and retries the on-chain submission.

**3c. Indexer**

Listens for `RoleGranted` / `RoleRevoked` events on the smart contract. Writes to the `UserContract` Prisma table with `txHash` and `onChainConfirmed = true`. This closes the loop: the app reads from Prisma, which is now a mirror of on-chain state.

**3d. Backfill**

For every active `UserContract` row, submit a `grantRole` transaction. This brings the on-chain state in sync with the off-chain state that was built in Phase 1. One-time migration, batched to stay within gas budgets.

### What this achieves

After Phase 3, every citizen's role is backed by an immutable, timestamped, publicly auditable on-chain record. If someone questions "did I pay for this role?", the answer isn't "our database says so" — it's "the blockchain says so, and here's the transaction hash." We can't alter it, we can't deny it, and neither can anyone else.

---

## Phase 4 — Long-term (6-12 months)

### Goal: zero-knowledge identity + verifiable credentials

**4a. ZK role attestations**

A citizen should be able to prove "I hold at least CITIZEN role" without revealing their address, their email, or their name. This enables:

- **Anonymous moderation**: a moderator proves they have `moderation:vote` capability without revealing which citizen they are. The moderation vote is verifiable (it came from a real moderator) but unlinkable (you can't tell which one).
- **Private governance**: a citizen votes on a proposal with their role-weighted power, but the vote itself is zero-knowledge. The outcome is auditable; the individual votes are not.
- **Reputation proofs**: a citizen can present a proof to a third party (another platform, an employer, a partner) that says "this person has a reputation score above X on Republike" without revealing who they are on Republike.

Technologies under evaluation:
- Semaphore (group membership proofs)
- EAS (Ethereum Attestation Service) for role attestations
- Soulbound tokens (ERC-5192) as non-transferable role badges
- Zupass-style portable credentials

**4b. AURE token migration**

Credits (currently a database integer) become a real ERC-20 token on the same L2:
- Epoch reward distribution → on-chain batch transfer
- Tip transfers → on-chain token transfers
- Credit balance → token balance (auditable, portable)
- Governance voting power could be weighted by token holdings + role (hybrid model)

This requires careful regulatory evaluation (utility token vs security token) and is the last phase, not the first.

**4c. Data sovereignty**

The ultimate goal: a citizen's identity, reputation, roles, and participation history exist as verifiable credentials they control. If Republike disappears tomorrow, the citizen still has cryptographic proof of everything they built. Their reputation is portable. Their governance participation is auditable. Their data is theirs.

---

## What this is NOT

- **Not a crypto product.** Users never buy tokens, trade assets, or interact with DeFi. The blockchain is infrastructure for sovereignty, not speculation.
- **Not a migration away from the current platform.** The app, the feed, the content, the moderation — all stay exactly as they are. The chain layer is underneath, invisible.
- **Not dependent on any single chain.** The architecture (contract model → signing service → indexer → Prisma mirror) works on any EVM-compatible L2. If a chain fails or declines, we redeploy to another. The app doesn't know the difference.
- **Not all-or-nothing.** Each phase delivers value independently. Phase 1 (contracts) already improves access control. Phase 2 (wallets) is preparatory. Phase 3 (on-chain) adds auditability. Phase 4 (ZK) adds sovereignty. You can stop at any phase and still have a better system than what existed before.

---

## Timeline

| Phase | Scope | Status | Horizon |
|-------|-------|--------|---------|
| 1 | Off-chain contract primitives | Built, deploying | Weeks |
| 2 | Non-custodial wallets + L2 selection | Research + build | Weeks |
| 3 | On-chain role registry + indexer | Build + migrate | Months |
| 4 | ZK identity + AURE token | Research + build | 6-12 months |

---

## The identity imperative

Everything starts with identity. If we get identity right — cryptographically verifiable, zero-knowledge, user-owned — then rewards, governance, ownership, and moderation all inherit those properties automatically. A reputation score attested on-chain is portable. A governance vote cast in zero-knowledge is private. A founder badge backed by a smart contract is unforgeable.

If we get identity wrong — or skip it — then everything built on top is just a database with extra steps.

That's why identity and user data protection are the top priority. Not because regulators demand it (though they do), but because the platform's promise to its citizens is that their participation has value, and value requires proof, and proof requires infrastructure that neither we nor anyone else can unilaterally manipulate.

The on-chain migration is that infrastructure. We're building it now.
