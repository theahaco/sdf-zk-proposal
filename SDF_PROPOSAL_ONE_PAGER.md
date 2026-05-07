# ZK Proofs on Soroban: Recovery + Privacy Bundle

**A proposal to the Stellar Development Foundation**

ZK proofs on Soroban are no longer hypothetical. We have a Noir/UltraHonk verifier validating real proofs on Stellar testnet today, and we are proposing to land it as a **recovery primitive in g2c** and a **privacy circuit in neftwerk** — under one coordinated grant with hard track separation.

## Why now

Soroban Smart Accounts have shipped, and OpenZeppelin's `CustomAccountInterface` is the de-facto integration surface (g2c uses it today). Two structural gaps remain:

1. **Recovery.** g2c's current story is "register a backup passkey." That is not enough — there is no key-rotation path for the lone-passkey case.
2. **Privacy.** Stellar lacks a native primitive for hiding ownership while keeping provenance verifiable — a blocker for institutional and RWA use cases.

Both reduce to the same ZK substrate. **Universal primitive in g2c + sector application in neftwerk.**

## The two pillars

**Pillar 1 — g2c: ZK account recovery as a universal primitive.** At enrollment the user records a 24-word BIP-39 seed on paper, offline. The wallet deterministically derives `owner_secret = HKDF(seed, account_id, network_passphrase)` and commits **only its one-way hash** `Poseidon2(owner_secret)` on-chain — the seed and `owner_secret` never leave the browser; on-chain observers see a meaningless 32-byte hash. On recovery, the user re-enters the seed, re-derives the same `owner_secret`, and generates a Noir/UltraHonk zero-knowledge proof of *"I know the preimage of the on-chain commitment"* — proving identity without revealing the secret. The proof is bound to a fully-specified `auth_hash` (account, network, contract, new pubkey with on-curve + canonical-limb checks, nonce, timelock duration), so it can't be replayed against a different rotation. A new `g2c-recovery-controller` contract holds the pending-rotation state machine; a separate `g2c-recovery-guard-policy` (OZ `Policy` scoped to `CallContract(self_addr)`) blocks any signer eviction during the 30-day window. Cancellation requires both the WebAuthn signer AND a fresh seed-knowledge proof, capped at 2 cancels per recovery (closes the cancel-loop attack). Integration plugs into g2c's existing OZ `do_check_auth` pipeline; no protocol changes.

**Pillar 2 — neftwerk: ZK privacy as a sector validator.** Neftwerk's ARTS-1 token standard for art on Stellar uses three Noir circuits — `arts1_mint` (dual-ECDSA institutional + collector), `arts1_ownership` (private transfer authority), and `arts1_recovery` (the same primitive as Pillar 1 with token-id binding). Privacy model: "private title, public provenance."

## Why bundle

One Noir + UltraHonk + Poseidon2 + BN254 + Soroban 25.x substrate. One verifier crate. One team. The recovery circuit is a **key-rotation primitive any Soroban Smart Account can adopt**; neftwerk is the second consumer, not the only one. We estimate **~30–40% engineering shared** across the two tracks — concentrated in substrate work (verifier crate, browser SDK, Poseidon2 binding, OZ shim, audit RFP) rather than co-developed circuit/contract code. **Track A and Track B are disbursed independently**: an A0 benchmark failure or Track B compliance gate does not affect Track A funding.

## Evidence we can ship today

- **Testnet E2E (`soroban-zk-demo`).** Browser generates Noir/UltraHonk proof; Soroban verifier contract validates on-chain. `./test_e2e.sh` runs build → deploy → prove → verify on Stellar testnet.
- **Verifier crate (`rs-soroban-ultrahonk`).** Standalone Soroban contract wrapping the UltraHonk verifier. VK ~1.9 KB, proof ~3.7 KB.
- **Nullifier-set + Merkle-frontier primitive.** Reference implementation with depth-20 tree, insertion proofs, and nullifier-set state — the substrate for revocation, replay protection, and future batched settlement.

## Budget and milestones

| Track | Scope | Tranches (10/20/30/40) | Total |
|---|---|---|---|
| **A0 (shared)** | Pre-grant Soroban gas/CPU benchmark gate. **Pre-A0 (no fee): publish `arts1_mint_skeleton.nr` gate count.** Then A0a $7.5k (testnet `verify_proof` numbers + Policy-with-cross-call benchmark + on-disk circuit aligned to spec) and A0b $7.5k (Groth16 comparison + browser benchmarks on iPhone 14 / Pixel 7 / 2020 MacBook Air) | $15k split 50/50 | $15k |
| **Track A — g2c recovery** | A1 spec + g2c-OZ shim → A2 verifier + named-org VK multi-sig + recovery-controller + recovery-guard-policy → A4 integration + cancel-loop tests → A5 browser UX + recovery-card → A6 audit-ready freeze + mainnet + signed integrator agreement | $75k tranched | $75k |
| **Track B — neftwerk privacy** | B0–B7, gated on A0 pass + Pillar 2 compliance gates reviewed by a named SDF reviewer (B1 held until C1 in hand) | $100k tranched | $100k |
| **Total** | 14 weeks, ~3 FTE | | **$190k** |

Audit credits requested separately from SDF's standard pool: **circuit audit** (zkSecurity / Veridise / Spearbit ZK) and **contract audit** (OtterSec / Trail of Bits / Halborn) named separately, with a 4-week circuit freeze and 3-week contract freeze.

**We're requesting a 30-minute technical review with the SDF protocol team to walk through the appendix, A0 benchmark plan, and milestone gating.**

---

Repos: `g2c` · `zk/rs-soroban-ultrahonk` · `zk/soroban-zk-demo` · `zk/neftwerk` — see `SDF_PROPOSAL_TECHNICAL_APPENDIX.md` for full architecture, recovery state machine, VK governance design, threat model, and Pillar 2 compliance gating.
