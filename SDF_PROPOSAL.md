# ZK Proofs on Soroban: Recovery + Privacy Bundle

*A proposal to the Stellar Development Foundation.*

---

## Executive summary

ZK proofs on Soroban are no longer hypothetical. We have a Noir/UltraHonk verifier validating real proofs on Stellar testnet today, and we are proposing to land it as a **recovery primitive in g2c** and a **privacy circuit in neftwerk** — under one coordinated grant with hard track separation.

**Why now.** Soroban Smart Accounts have shipped, and OpenZeppelin's `CustomAccountInterface` is the de-facto integration surface (g2c uses it today). Two structural gaps remain:

1. **Recovery.** g2c's current story is "register a backup passkey." That is not enough — there is no key-rotation path for the lone-passkey case.
2. **Privacy.** Stellar lacks a native primitive for hiding ownership while keeping provenance verifiable — a blocker for institutional and RWA use cases.

Both reduce to the same ZK substrate. **Universal primitive in g2c + sector application in neftwerk.**

**Pillar 1 — g2c: ZK account recovery as a universal primitive.** At enrollment the user records a 24-word BIP-39 seed on paper, offline. The wallet deterministically derives `owner_secret = HKDF(seed, account_id, network_passphrase)` and commits **only its one-way hash** `Poseidon2(owner_secret)` on-chain — the seed and `owner_secret` never leave the browser; on-chain observers see a meaningless 32-byte hash. On recovery, the user re-enters the seed, re-derives the same `owner_secret`, and generates a Noir/UltraHonk zero-knowledge proof of *"I know the preimage of the on-chain commitment"* — proving identity without revealing the secret. The proof is bound to a fully-specified `auth_hash` (account, network, contract, new pubkey with on-curve + canonical-limb checks, nonce, timelock duration), so it can't be replayed against a different rotation. A new `g2c-recovery-controller` contract holds the pending-rotation state machine; a separate `g2c-recovery-guard-policy` (OZ `Policy` scoped to `CallContract(self_addr)`) blocks any signer eviction during the 30-day window. Cancellation requires both the WebAuthn signer AND a fresh seed-knowledge proof, capped at 2 cancels per recovery (closes the cancel-loop attack). Integration plugs into g2c's existing OZ `do_check_auth` pipeline; no protocol changes.

**Pillar 2 — neftwerk: ZK privacy as a sector validator.** Neftwerk's ARTS-1 token standard for art on Stellar uses three Noir circuits — `arts1_mint` (dual-ECDSA institutional + collector), `arts1_ownership` (private transfer authority), and `arts1_recovery` (the same primitive as Pillar 1 with token-id binding). Privacy model: "private title, public provenance."

**Why bundle.** One Noir + UltraHonk + Poseidon2 + BN254 + Soroban 26.x substrate. One verifier crate. One team. The recovery circuit is a **key-rotation primitive any Soroban Smart Account can adopt**; neftwerk is the second consumer, not the only one. We estimate **~30–40% engineering shared** across the two tracks — concentrated in substrate work (verifier crate, browser SDK, Poseidon2 binding, OZ shim, audit RFP) rather than co-developed circuit/contract code. Track A and Track B are disbursed independently: an A0 benchmark failure or a Track B compliance gate does not affect Track A funding.

**Evidence we can ship today.**
- **Testnet E2E (`soroban-zk-demo`).** Browser generates Noir/UltraHonk proof; Soroban verifier contract validates on-chain. `./test_e2e.sh` runs build → deploy → prove → verify on Stellar testnet.
- **Verifier crate (`rs-soroban-ultrahonk`).** Standalone Soroban contract wrapping the UltraHonk verifier. VK ~1.9 KB, proof ~3.7 KB.
- **Nullifier-set + Merkle-frontier primitive.** Reference implementation with depth-20 tree, insertion proofs, and nullifier-set state — the substrate for revocation, replay protection, and future batched settlement.

**Budget headline.**

| Track | Scope | Total |
|---|---|---|
| **A0 (shared)** | Pre-grant Soroban gas/CPU benchmark gate | $15k split 50/50 |
| **Track A — g2c recovery** | A1 spec + g2c-OZ shim → A2 verifier + named-org VK multi-sig + recovery-controller + recovery-guard-policy → A4 integration + cancel-loop tests → A5 browser UX + recovery-card → A6 audit-ready freeze + mainnet + signed integrator agreement | $75k tranched |
| **Track B — neftwerk privacy** | B0–B7, gated on A0 pass + Pillar 2 compliance gates reviewed by a named SDF reviewer (B1 held until C1 in hand) | $100k tranched |
| **Total** | 14 weeks, ~3 FTE | **$190k** |

Audit credits requested separately from SDF's standard pool: **circuit audit** (zkSecurity / Veridise / Spearbit ZK) and **contract audit** (OtterSec / Trail of Bits / Halborn) named separately, with a 4-week circuit freeze and 3-week contract freeze.

---

## 1. Architecture: ZK signer plug-in to g2c

The integration introduces **three new contracts** plus a compatibility shim:

- **`g2c-zk-recovery-verifier`** — stateless OZ `Verifier`. **Implements the full trait**: `verify(env, hash, key_data, sig_data) -> bool`, `canonicalize_key(env, key_data) -> Bytes`, `batch_canonicalize_key(env, key_data_list) -> Bytes`. `KeyData = BytesN<32>` (Poseidon2 commitment). `SigData = (proof_bytes: Bytes, public_inputs: Vec<BytesN<32>>)`. Cross-invokes `ultrahonk-soroban-verifier`.
- **`g2c-recovery-controller`** — stateful contract owning the pending-rotation state machine (§3). Persistent storage with **explicit TTL extension paid from a per-account recovery deposit**; if deposit is exhausted, `complete_recovery` fails-closed and the pending state archives safely. Entrypoints: `initiate_recovery`, `cancel_recovery`, `complete_recovery`. **Controller upgrades are governed by the same 3-of-5 VK multi-sig + 14-day timelock as the verifier registry.**
- **`g2c-recovery-guard-policy`** — separate OZ `Policy` contract, scoped to `ContextRuleType::CallContract(self_address)` and **installed on every `ContextRule` that can authorize signer mutations**. Its `enforce()` reads the recovery-controller's pending-rotation state via cross-contract call; during a pending rotation it blocks `remove_signer(zk)` and requires both signers' auth for `add_signer(*)`. The cross-contract-call gas overhead is measured explicitly in A0 (Policy-with-cross-call benchmark).
- **`g2c-oz-compat`** — thin re-export crate over the OZ `stellar-accounts` surface area g2c uses. Pins the OZ version (commit hash + storage-layout XDR hash). CI test fails on storage-layout drift. Future OZ upgrades touch this one file. **A1 deliverable also publishes an `OZ_UPSTREAM_DIFF.md`** documenting every divergence between g2c's consumer-side trait usage and upstream OZ traits, so the contract auditor can scope the shim's compatibility surface explicitly.

### No non-factory enrollment paths

ZK-signer registration is **only supported via the atomic factory path** (`g2c-factory.create_account_with_zk_signer`). No other enrollment path (manual SDK call, migration of existing g2c accounts, post-deployment registration) is supported. This forecloses pre-enrollment squatting on non-factory paths: a commitment cannot be registered against an `account_id` outside the atomic factory transaction. Migration of existing g2c accounts to ZK recovery is a separate post-grant deliverable with its own threat model.

### Critical safety property: signer-removal guard

The existing OZ `add_signer` / `remove_signer` (`smart-account/contract.rs:105–113`) requires only `current_contract_address().require_auth()` — a stolen WebAuthn signer satisfies this. Naive integration is bypassable.

The fix is the `g2c-recovery-guard-policy` above. **Three properties make it correct:**

1. The Policy is **registered on every `ContextRule` that lists a signer with `add_signer`/`remove_signer` authority** — not just the Default rule. A1 deliverable includes a per-rule policy-attachment audit.
2. The Policy is scoped to `ContextRuleType::CallContract(self_address)`, so it intercepts inner calls to the SmartAccount's signer mutation entrypoints regardless of how `auth_contexts` selects rules.
3. The recovery-controller is registered as a `Signer::Delegated(controller_addr)`, so `complete_recovery()` can satisfy `current_contract_address().require_auth()` when it cross-calls the SmartAccount to atomically rotate the signer — but only after timelock + proof-verified.

A4 acceptance criterion: differential tests cover (i) stolen-passkey-only bypass attempt, (ii) double-recovery, (iii) expiration race, (iv) cancellation during legitimate recovery, (v) policy-not-on-rule bypass, (vi) cancel-loop attack (see §3).

### Sequence diagram for `complete_recovery`

```
Submitter        recovery-controller       SmartAccount         guard-policy        webauthn-verifier (existing)
   │                    │                       │                    │                    │
   │── complete_rec ───▶│                       │                    │                    │
   │                    │── verify (now ≥ initiation_ledger          │                    │
   │                    │   + timelock_duration + Σ cancel_pauses)   │                    │
   │                    │── verify deposit_rent_sufficient (else     │                    │
   │                    │   fail-closed; pending entry archives)     │                    │
   │                    │── verify proof valid ─│                    │                    │
   │                    │── verify cancel_count │                    │                    │
   │                    │   ≤ cap (=2)          │                    │                    │
   │                    │── inner call ────────▶│ add_signer(new_pk) │                    │
   │                    │                       │── __check_auth ───▶│                    │
   │                    │                       │   Context = CallContract(self,           │
   │                    │                       │              add_signer)                 │
   │                    │                       │   ContextRule = Default                  │
   │                    │                       │   guard-policy.enforce(ctx, signers, …) │
   │                    │                       │   reads pending_recovery_state           │
   │                    │                       │   sees: pending, controller is signer,   │
   │                    │                       │   timelock_expired, proof_verified       │
   │                    │                       │   → permits add_signer                  │
   │                    │                       │── inner call ─────│ require_auth(ctlr) │
   │                    │                       │── ok ─────────────│                    │
   │                    │   inner call ────────▶│ remove_signer(pk_old)                   │
   │                    │                       │── __check_auth ───▶│ same path → permits │
   │                    │── refund deposit ────▶│                    │                    │
   │                    │── emit RecoveryCompleted                                          │
   │── ok ──────────────│                                                                   │
```

The completion call is **two inner calls** (add_signer then remove_signer) gated by the same guard-policy invocation against the same pending-recovery state. A0 measures this exact path.

---

## 2. Shared ZK substrate

| Layer | Choice | Notes |
|---|---|---|
| Circuit DSL | Noir (`nargo` 1.0.0-beta.x) | Mature tooling; first-class UltraHonk backend |
| Proving system | UltraHonk via Aztec's Barretenberg (`bb` 0.87.x) — pinned by SHA-256 of the release tarball | Universal SRS; A2 VK registry supports `bb`-version bumps |
| Hash | Poseidon2 over BN254 scalar field, **Aztec parameter set t=3 (commit width-2 sponge), round constants pinned by SHA-256** | Domain-separated per use (commitment / recovery / mint / ownership / nullifier) |
| Curve | BN254 | ~100-bit pairing security; migration plan to BLS12-381 (Appendix A) |
| ECDSA-P256 gadget | `noir-bigcurve` + `noir_ecdsa` from `noir-lang`, pinned commit + matched `bb` 0.87.x compatibility | A1 spec names exact tags |
| Verifier | `ultrahonk-soroban-verifier` Rust crate (no_std + alloc) | Deployed to Stellar testnet today |
| OZ compatibility | New `g2c-oz-compat` shim crate (see §1) | Single re-export surface; storage-layout XDR hash regression suite in CI |
| Soroban SDK | 26.x | Pinned to a specific commit at A1 |
| Browser prover | `bb.js` (WASM, lazy-loaded on `/recover/` only) | COOP/COEP deployment plan in §4 |

**Why UltraHonk over Groth16, Plonk, Halo2, RISC0/SP1?** Universal SRS removes per-circuit ceremonies — necessary for a recovery primitive instantiated against bespoke wallet circuits. A0 publishes a Groth16-on-Soroban verification-CPU comparison; if Groth16 verifies at <1/3 the budget for the fixed Pillar-1 recovery circuit, we add a Groth16 verifier *behind the same VK registry* (no fork of the substrate; same VK upgrade governance applies).

---

## 3. Recovery flow end-to-end (Pillar 1)

Six steps. (1)–(2) at SmartAccount enrollment; (3)–(6) on recovery.

1. **Seed at enrollment.** During g2c onboarding the wallet asks the user to write down a BIP-39 mnemonic (passphrase optional, explicitly supported). The seed never touches the network.
2. **Commitment + recovery deposit on-chain.** The wallet derives `owner_secret = HKDF-SHA256(ikm = seed, salt = network_passphrase, info = "g2c-recovery-v1" ‖ account_id)`, reduces modulo BN254 scalar order, and submits `commitment = Poseidon2(domain_sep_commitment, owner_secret)` to the `recovery-controller` along with a one-time **recovery deposit** (covers ~40 days of TTL extension at current Soroban rent). The deposit is refundable on `complete_recovery` net of consumed rent.
3. **Recovery trigger.** User has lost their passkey. Opens `mysoroban.xyz/recover/`. **Cross-account discovery decision**: the page does NOT maintain a server-side `commitment → C-address` index (privacy regression). Instead the user enters the C-address from a **recovery card**. **Recovery-card content rule (security-critical):** the recovery card contains *only* the C-address and a human-readable account label. **The card MUST NOT contain the BIP-39 seed**, which is held on a separate paper artifact never digitized as QR. This separation defeats the physical-card exfiltration attack (photo of QR ≠ access to seed). Wallets MUST provide a recovery-card export at enrollment as an A5 acceptance criterion. Re-enters BIP-39 mnemonic with wordlist autocomplete + checksum validation + passphrase prompt + language selection.

   **Per-account recovery initiation rate limit:** the controller enforces a maximum of 3 `initiate_recovery` calls per 90-day rolling window. This prevents recovery-deposit griefing (initiate→cancel→re-initiate). The 4th attempt within the window is rejected; the user must wait until the oldest initiation falls outside the window.
4. **Proof generation in the browser.** Using `bb.js` (WASM, lazy-loaded), the browser generates a Noir/UltraHonk proof for the circuit:
   ```
   public inputs:
     commitment           : Field    // pinned at enrollment
     auth_hash            : Field    // single Field, see preimage spec below
   private inputs:
     owner_secret         : Field
     new_signer_pubkey    : (Field, Field, Field, Field, Field)  // 5 limbs ≤128 bits
   asserts:
     Poseidon2(domain_sep_commitment, owner_secret) == commitment
     auth_hash == Poseidon2(domain_sep_recovery, account_id, network_passphrase, contract_addr,
                            limb0, limb1, limb2, limb3, limb4, nonce, timelock_duration)
     // Pubkey integrity:
     each_limb < 2^128                                // range check
     limbs_compose_canonically_to_65_bytes            // canonical decomposition
     point_on_secp256r1(decompressed_pubkey)          // on-curve check
   ```
   The **`auth_hash` is exposed as a single `Field`** (Poseidon2-hashed externally to one element). All subordinate fields enter the Poseidon2 input vector with explicit byte-encoding rules (Appendix A.1).
5. **On-chain submission.** Wallet calls `recovery-controller.initiate_recovery(account, proof, sig_data, new_pubkey)`. The controller:
   - Recomputes `auth_hash` from supplied fields and verifies match against the proof's public input
   - **Verifies `timelock_duration ≥ 7 days` (controller-side floor; not just in-circuit)**
   - Verifies `nonce` is monotonic (controller's per-account counter)
   - Pays TTL extension from the recovery deposit (35 days)
   - Stores pending rotation in persistent storage
   - Emits `RecoveryInitiated(account, new_pubkey_hash, completes_at_ledger, cancel_count=0)` event
   - Wallets MUST subscribe and notify via the user's opted-in channel (push/email/SMS)
6. **30-day timelock + cancellation policy + completion.** During the window, `cancel_recovery()` requires:
   - WebAuthn signer signature over `cancel_hash = Poseidon2(domain_sep_cancel, account_id, contract_addr, current_recovery_nonce, ledger_seq_at_cancel)`
   - **AND** a fresh `cancel_proof`: a mini-Noir circuit proving `Poseidon2(domain_sep_commitment, secret) == commitment` for the same `commitment`. This means cancellation requires *either* the legitimate user (who has the seed) *or* an attacker who has both the WebAuthn signer AND the seed (in which case rotation is the easier attack — cancellation is pointless).
   - **Cap of 2 cancels per recovery initiation.** A 3rd `cancel_recovery` is rejected; the recovery completes after the 30-day window. Cancel count is included in `RecoveryInitiated`/`RecoveryCancelled` events for monitoring.
   - 24h cooldown between consecutive cancels.

   After the timelock + total cancellation pauses, anyone may call `complete_recovery()`. The controller — registered as a `Signer::Delegated` admin signer — atomically calls `SmartAccount.add_signer(new_pubkey)` and `SmartAccount.remove_signer(old_webauthn_signer)` in a single inner call, gated by the `g2c-recovery-guard-policy` which now permits both because the controller's auth is valid post-timelock-post-proof.

The cancel-loop attack is closed by the cap + the cancel-proof requirement. The TTL-archival race is closed by the per-account deposit covering the full window.

---

## 4. Privacy flow end-to-end (Pillar 2)

Three circuits, with explicit input/output specs and gate-count estimates (to be confirmed at A0).

### `arts1_mint` (heaviest)

Verifies at minting that **both** Neftwerk (institutional co-signer) and the collector authorized the mint over the same `auth_hash`.

- **Public inputs:** `commitment` (output), `provenance_root`, `auth_hash` (single Field), `mint_id`, `token_id`.
- **Private inputs:** institutional ECDSA-P256 signature, collector ECDSA-P256 signature, `provenance_secret`, collector `owner_secret`, collector pubkey limbs.
- **Asserts:** institutional and collector pubkeys both on-curve; both ECDSA verify against `auth_hash`; canonical-limb encoding for both pubkeys; `auth_hash` binds `mint_id`, `token_id`, both pubkeys, `provenance_secret`, and an institution-side freshness counter; `commitment = Poseidon2(domain_sep_arts1_mint, collector_pk_limbs, owner_secret)`.
- **Estimated gates:** 220–280k UltraHonk gates. Browser prove: 90–180s in WASM, peak heap 1.2–1.8 GB. Will OOM iOS Safari; **mobile mint must use server-side proving via Neftwerk Protocol** (acceptable trust model at mint because Neftwerk is the institutional co-signer anyway).

### `arts1_ownership`

- **Public inputs:** `commitment`, `auth_hash`, `token_id`.
- **Private inputs:** `owner_secret`, collector pubkey limbs, ECDSA signature over `auth_hash`.
- **Asserts:** `auth_hash` binds `token_id`, transaction nonce, contract address, function selector, call args; pubkey on-curve; canonical limbs; commitment matches; signature valid.
- **Estimated gates:** ~110k UltraHonk gates. Browser prove: 8–15s.

### `arts1_recovery`

- **Public inputs:** `owner_secret_commitment`, `auth_hash`, `token_id`, `new_owner_pubkey_limbs`.
- **Private inputs:** `owner_secret`.
- **Asserts:** `auth_hash` binds `token_id`, contract address, `new_owner_pubkey_limbs`, monotonic nonce, timelock floor; commitment matches.
- **Estimated gates:** 6–9k UltraHonk gates. Browser prove: 2–4s.

### Privacy properties and known gaps

The on-chain ARTS-1 contract knows *what* a token is (via metadata CID) but not its price or its collector. Price/split logic is off-chain in Neftwerk's database, injected at sale time as transaction arguments ("late-binding").

**Known gap:** an observer can infer the price by watching USDC outflows on settlement. **Roadmap commitment:** a fourth circuit `arts1_settlement` using the nullifier-set primitive of §5 (out of scope for this grant; explicitly named here as the next milestone).

---

## 5. Deployment evidence

### `soroban-zk-demo` — full E2E on testnet

- `./test_e2e.sh` runs build → deploy → prove → verify on testnet.
- Three-contract pattern: `zk-contract` (commitment + cross-call), `ultrahonk-soroban-contract` (proof verifier), `zk-factory-contract` (factory).
- **Note:** the current on-disk circuit (`circuits/ownership_proof/src/main.nr`) is a hash-knowledge demo; the `assert(new_commitment == new_commitment)` no-op does not bind the public input. **A0 deliverable #1 brings this into agreement with §3's spec** before any other circuit work.

### `rs-soroban-ultrahonk` — verifier crate

- Soroban contract wrapping the UltraHonk verifier; integration tests for simple and Fibonacci-chain circuits.
- Cost-measurement script in `scripts/invoke_ultrahonk/` for testnet gas readings.
- VK currently immutable at `__constructor`; A2 delivers the upgrade governance.

### Nullifier-set + Merkle-frontier primitive

- Reference implementation with depth-20 tree, insertion proofs, and nullifier-set state on Soroban.
- Demonstrates the substrate for revocation, replay protection, and future batched settlement (`arts1_settlement`).
- Repo directory will be renamed from `tornado_classic` to `nullifier_set_primitive` before grant kickoff.

---

## 6. Milestone breakdown

Two parallel tracks plus a shared pre-grant benchmark. All milestones have measurable acceptance criteria; tranche release is gated on each milestone's acceptance.

### Pre-A0 — Mandatory pre-disbursement skeleton (no SDF fee)

**Hard precondition for ANY SDF disbursement, including A0.** Before grant signing, the team publishes the `arts1_mint_skeleton.nr` gate count under `bb` 0.87.x — two `noir-bigcurve` ECDSA-P256 verifies + dummy Poseidon2, no real binding. ~2 days of work, performed at the team's own cost. The published number replaces the prior estimate as the reference for A0's pass criterion. **No SDF money flows until this is published.**

### A0 — Pre-grant Soroban gas benchmark (weeks 1–2, $15k split 50/50)

**Cash-flow:** A0 is **reimbursement after delivery**, not advance funding. The team carries A0 work at risk; SDF disburses A0a's $7.5k on A0a deliverables verified, A0b's $7.5k on A0b deliverables verified. If A0 fails the pass criterion, the $15k is not paid; no clawback because no advance was made.

**A0 deliverables, paid 50/50 on completion:**

| Half | Amount | Deliverables |
|---|---|---|
| A0a | $7.5k | (1) On-disk `ownership_proof` circuit brought into spec agreement. (2) Published CPU instructions, memory bytes, and resource fee for `verify_proof` on testnet for: simple-1-input, Fibonacci-chain, and the `arts1_mint`-shaped reference circuit (using the pre-A0 skeleton). (3) Policy-with-cross-contract-call benchmark (the `g2c-recovery-guard-policy` + controller cross-call cost). |
| A0b | $7.5k | (1) Groth16-on-Soroban verifier comparison for the recovery circuit. (2) Browser proving benchmarks (peak heap, prove time, WASM bundle size) on three named device tiers: iPhone 14, Pixel 7, 2020 MacBook Air. (3) Mobile-prove feasibility report for `arts1_mint` (expected: server-side mint required). |

**Pass criterion (calibrated against pre-A0 skeleton number, not vapor):** `arts1_mint`-shaped circuit verification cost on Soroban must satisfy **BOTH**:

- **Design assumption check:** cost ≤ `1.5 × measured_skeleton_cost` (the 1.5× multiplier accommodates the gap between skeleton with dummy Poseidon2 and the full circuit with real binding/provenance work)
- **Operational headroom check:** cost ≤ **80M instructions** (= 80% of Soroban's 100M-instruction cap; leaves 20M for auth, storage, and policy cross-call overhead under load)

AND Policy-with-cross-call adds <10M instructions overhead per `__check_auth`. **Pass = both bounds met.**

If the actual `arts1_mint` cost exceeds either bound, SDF and team decide between (a) descope, (b) re-architect with recursive aggregation, (c) terminate. **If pre-A0 skeleton itself measures above 50M instructions, the 1.5× bound exceeds 75M which collides with the 80M ceiling — Track B's `arts1_mint` is descoped to single-ECDSA in scope before any funds are committed.**

### Track A — g2c ZK recovery ($75k tranched 10/20/30/40)

**Tranche-payment milestones are bolded.** Other milestones are unpaid prerequisites whose work is rolled into the next paid tranche's release.

| # | Milestone | Weeks | Acceptance criteria |
|---|---|---|---|
| **A1** *(10% — $7.5k)* | Spec doc + Noir circuits (recovery + cancel-proof) | 2–4 | Single PDF pinning: Poseidon2 round constants by SHA-256; **`noir-bigcurve` and `noir_ecdsa` commit hashes pinned**; limb canonicalization rules + on-curve check spec; `auth_hash` byte-encoding; HKDF parameters; field-reduction rule; cancellation-hash spec; security argument (reduction sketch); replay matrix (account / network / contract / nonce / token / encoding). Recovery + cancel-proof circuits compile; replay/cross-account/cross-network/canonical-encoding unit tests pass. **`g2c-oz-compat` shim crate published** with storage-layout XDR hash regression suite in CI; **`OZ_UPSTREAM_DIFF.md` published** documenting all consumer-side divergences. **Verifier trait fully implemented** (`verify`, `canonicalize_key`, `batch_canonicalize_key`); `canonicalize_key` malleability fuzzer in tests. **`pnpm test:parity` CI test** running round-trip across `bb.js` prover, JS Poseidon2 commitment generator, and Soroban verifier; covers all 7 domain separators with adversarial vectors (cross-implementation drift kills recovery silently). OZ `stellar-accounts` commit pinned. |
| **A2** *(20% — $15k)* | Verifier crate v2 + VK governance + recovery-controller + recovery-guard-policy | 4–7 | `ultrahonk-soroban-verifier` v2 with VK registry (circuit_id → VK). **VK governance: 3-of-5 multi-sig with diversified-quorum rule across three groups**: Group A (Stellar-aligned): 1 SDF + 1 Stellar Foundation external (independently selected; treated as a single effective vote for collusion analysis). Group B (auditing org): 1 OpenZeppelin. Group C (community): 2 signers via a **two-stage public RFP** (open nomination → 30-day public comment → on-chain attestation by Group A + Group B). **Quorum rule: any 3-of-5 quorum MUST include at least 1 signer from Group A AND at least 1 signer from {B ∪ C}**. This forecloses both an all-Stellar quorum (Stellar-only collusion impossible — Group B or C must consent) AND an all-non-Stellar quorum (OZ + community alone cannot push a VK upgrade). All keys **FIDO2 + WebAuthn attestation required, with accepted AAGUIDs published** (YubiKey 5 series, SoloKeys, T-series; HSM-only operation). **180-day mandatory key rotation** verified on-chain. **Kill-switch with anti-DoS:** any 1-of-5 can pause a pending upgrade for 7 days; a second pause within 30 days requires 4-of-5 to overcome (prevents one compromised signer from indefinitely DoS'ing emergency rotation). 14-day timelock on upgrades, opt-in per account, orphan-token migration ceremony documented. `g2c-recovery-controller` deployed to testnet with full state machine + state-machine PDF + storage-layout diagram + concrete `Context` example for `complete_recovery` per §1 sequence diagram. `g2c-recovery-guard-policy` deployed; per-rule attachment audit produced. **Controller upgrade governance: same multi-sig + same diversified-quorum rule + same timelock + same opt-in.** |
| A3 | Atomic enrollment-time registration | 6–7 | g2c-factory `create_account` accepts pre-registered ZK signer at construction with `Vec<Signer>`, atomically registers commitment with recovery-controller, and enforces `timelock_duration ≥ 7 days` on enrollment. *Unpaid; rolled into A4.* |
| **A4** *(30% — $22.5k)* | SmartAccount integration + signer-removal guard + cancel-loop differential tests | 7–9 | OZ `Policy` correctly scoped to `CallContract(self_addr)` and installed on every signer-mutating ContextRule. Differential tests covering: stolen-passkey bypass, double-recovery, expiration, cancellation, policy-not-on-rule bypass, cancel-loop, TTL-archival race, restore-after-cancel ordering. |
| A5 | Browser proving UX + recovery-card export | 8–10 | <5 min recovery proof on 2020 MacBook Air, <8 min on Pixel 7 (median across 3 devices of each tier). BIP-39 wordlist autocomplete, checksum, passphrase, language selection. **Recovery-card export at enrollment** (printable C-address QR; seed remains on separate paper). Cross-origin isolation deployment plan validated. Graduated unlock during timelock (read-only access). *Unpaid; rolled into A6.* |
| **A6** *(40% — $30k, with $7.5k withhold if integrator agreement not signed)* | Audit-ready freeze + mainnet launch + integrator signed | 10–14 | All contracts and circuits frozen for 4 weeks circuit audit + 3 weeks contract audit + 2-week fix window each. Mainnet deployment with **at least one external wallet integrator's signed integration agreement**. **Fallback (no schedule slip):** if signed agreement is not secured by week 12, A6 ships with (a) integrator's-guide published + (b) public integration RFP open + (c) ≥1 LOI in hand; SDF holds back $7.5k of the $30k tranche pending signed agreement within 60 days post-grant, releasing the withhold on signature. |

### Track B — neftwerk ZK privacy ($100k tranched 10/20/30/40)

**Pillar 2 funding gated on Pillar 2 compliance gates (§9) AND A0 pass. B1 disbursement held until C1 legal opinions are in SDF's hands.**

**Tranche-payment milestones are bolded.** Other milestones are unpaid prerequisites whose work is rolled into the next paid tranche's release.

| # | Milestone | Weeks | Acceptance criteria |
|---|---|---|---|
| B0 | Coinflow ↔ Soroban transaction-attachment spike + dual-ECDSA feasibility | 2–3 | Coinflow ↔ Soroban primary-sale flow validated on testnet. Dual-ECDSA constraint count published (real circuit, not skeleton). *Unpaid; rolled into B1.* |
| **B1** *(10% — $10k)* | Three-circuit spec | 3–5 | Spec PDF following A1 format. All three circuits compile. **C1 legal opinions delivered to SDF reviewer** as gate condition. |
| B2 | `arts1_ownership` | 5–7 | Compiled circuit + verifier integration; replay tests pass; on-curve + canonical-limb tests pass. *Unpaid; rolled into B4.* |
| B3 | `arts1_recovery` (parameterized A1 primitive) | 7–8 | Same Noir circuit pattern as A1, parameterized for `token_id`. *Unpaid; rolled into B4.* |
| **B4** *(20% — $20k)* | `arts1_mint` | 7–10 | Compiled circuit; verification cost within A0 budget; **descope to single-ECDSA collector signature** if A0 left <2× headroom (decision documented); server-side proving infrastructure for mobile mint live. |
| B5 | Browser pipeline + royalty enforcement scaffolding | 10–12 | Royalty-locked secondary-transfer logic compiles on testnet. *Unpaid; rolled into B6.* |
| **B6** *(30% — $30k)* | ARTS-1 token contract + on-chain freeze | 10–13 | E2E mint and transfer with privacy on testnet. Royalty-locked secondary transfers as acceptance criterion. **C2 on-chain freeze function for legal process implemented and tested.** |
| **B7a** *(40% — $40k less $2.5k withhold if mainnet)* | ARTS-1 pilot launch (mainnet or testnet) | 13–14 | Conditional on signed MoU at A0 completion. **Acceptance:** pilot transaction(s) executed on mainnet (or fully-rigged testnet pilot ready for partner activation if MoU not secured by week 6) + telemetry instrumentation live + **7-day in-grant retention metric** (not 30-day, which exceeds grant period). |
| B7b *(post-grant — $0 delta or released $2.5k withhold)* | 30-day retention report | week 17–18 | Disbursed conditionally **after grant close**: telemetry already running, report generation only. SDF releases the $2.5k withhold from B7a on receipt of B7b (only applicable if mainnet pilot launched at B7a). |

### Shared dependencies

- A2 deliverables feed B2/B4/B5.
- Browser proving SDK (A5) is consumed by B6.
- Audit firms cover both tracks (see §7).

---

## 7. Budget

**Total: $190k over 14 weeks.**

| Line item | Amount | Tranching |
|---|---|---|
| A0 — pre-grant benchmark (split 50/50) | $15k | A0a $7.5k / A0b $7.5k on completion of each half |
| Track A — g2c ZK recovery | $75k | 10% / 20% / 30% / 40% on A1 / A2 / A4 / A6 |
| Track B — neftwerk ZK privacy | $100k | 10% / 20% / 30% / 40% on **B1 / B4 / B6 / B7a** (B7b is post-grant, $0 delta with $2.5k withhold from B7a if mainnet launched), gated on A0 pass + Pillar 2 compliance gates; B1 held until C1 in SDF reviewer's hands |
| **Total** | **$190k** | |

**Audit credits** requested separately from SDF's standard pool. Two auditors, two scopes:

- **Circuit audit** (4-week scope): zkSecurity, Veridise, or Spearbit ZK practice. Scope: all four Noir circuits + cancel-proof circuit, Poseidon2 parameter set, ECDSA-in-circuit gadgets, limb canonicalization, on-curve checks, replay protection, `auth_hash` preimage binding.
- **Contract audit** (3-week scope): OtterSec, Trail of Bits, or Halborn. Scope: `g2c-zk-recovery-verifier`, `g2c-recovery-controller`, `g2c-recovery-guard-policy`, `g2c-oz-compat` shim, generalized factory changes, ARTS-1 token contract.

2-week fix window after each audit. No mainnet deployment until both audits clean.

**Team:** 3 FTE — one ZK/circuit lead (Aztec/Noir background required, must have shipped a production ECDSA-in-circuit gadget), one Soroban/Rust contracts engineer, one frontend/browser-prover engineer. Names and FTE percentages provided to SDF on grant signing. **Per-FTE loading chart** (delivered with grant agreement, summary here):

| Week | ZK lead (Track-A bias) | Contracts eng (cross-track) | Frontend/prover eng |
|---|---|---|---|
| 1–2 | A0 skeleton + measurements | A0 Policy benchmark + cost script | A0 browser benchmarks |
| 3–4 | A1 spec PDF + circuits | A1 OZ shim + verifier trait | A1 parity CI test scaffold |
| 5–7 | B1 / B2 spec + circuit | A2 controller + guard-policy + VK registry | A5 wireframes + recovery-card export |
| 7–9 | B4 mint circuit | A3 factory + A4 integration tests | A5 BIP-39 UX + WASM lazy-load |
| 9–11 | B5 / B6 circuits + parity tests | B6 token contract + freeze function | A5 final UX + cross-origin isolation |
| 11–13 | Audit-prep freeze + circuit-audit interface | Audit-prep freeze + contract-audit interface | A5 mainnet polish |
| 13–14 | B7 pilot launch support | A6 mainnet deploy + integrator handoff | B7 telemetry + retention metrics |

**Budget transparency:** $190k = $15k (A0, partial allocation across 3 FTE for 2 weeks) + $175k for the 12-week Track A + Track B build (≈ $4.86k/FTE/week × 3 FTE × 12 weeks). The 14-week envelope includes A0; A0's $15k is sized as partial-allocation pre-grant work, not a full 3-FTE × 2-week sprint. All figures are **gross** (fully loaded comp, taxes, benefits, overhead). At ~$4.86k/FTE/week the budget assumes either (a) the team is taking ecosystem-rate compensation below senior-crypto market, treating the grant as partial cost-recovery for work they'd do anyway, or (b) part-time allocation per FTE. Team commitments are documented at grant signing. SDF retains the right to descope or restructure if the loading chart proves untenable at week 4 review.

**Track independence clause** (codified in grant agreement): Track A milestone payments are not contingent on Track B status, and vice versa, except for the A0 prerequisite. Track A continues even if Track B is paused for compliance, A0 fails for `arts1_mint`-sized circuits, or B7's MoU does not materialize. Track B is held back if Track A's audit invalidates the shared verifier.

**Compliance reviewer designation:** SDF designates a named internal counsel or external reviewer for Pillar 2 compliance gates C1/C3 (legal opinions). Reviewer name disclosed at grant signing. No tranche release without reviewer sign-off.

---

## 8. Risk and mitigation

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | UltraHonk gas/budget exceeds 100M-instruction cap for `arts1_mint` | Medium | High | A0 measurement gate paid as flat pre-grant fee; pre-built `arts1_mint_skeleton.nr` published before A0 signs; failure paths defined (descope / aggregate / terminate). |
| 2 | Larger circuits (~50+ constraints, dual-ECDSA) untested | Medium | Medium | A0 includes `arts1_mint`-shaped reference; B0 runs real dual-ECDSA spike; B4 has explicit single-ECDSA descope path. |
| 3 | Recovery proof front-running or replay across accounts/networks/contracts/encodings | Medium | High | `auth_hash` preimage fully specified §3; on-curve + canonical-limb checks added; replay matrix in A1 covers six vectors. |
| 4 | WebAuthn signer evicts ZK signer (stolen-passkey bypass) | Medium | Critical | `g2c-recovery-guard-policy` scoped to `CallContract(self_addr)`, installed on every signer-mutating ContextRule; differential tests in A4. |
| 5 | VK governance attack (malicious VK swap) | Low | Catastrophic | 3-of-5 multi-sig with **named organizational signers** (1 SDF + 1 Stellar Foundation external + 1 OZ + 2 community via public RFP); **FIDO2 hardware attestation required**; 14-day on-chain timelock; opt-in per account; 180-day mandatory rotation; **kill-switch (any 1-of-5 pauses)**; existing accounts pinned to enrolled VK at the verifier-crate level. |
| 6 | Browser proving fails on mobile or under cross-origin isolation | Medium | Medium | A5 publishes measured numbers; WASM lazy-loaded only on `/recover/`; COOP/COEP headers + Cloudflare Worker compatibility validated. **`arts1_mint` mobile mint uses server-side proving via Neftwerk Protocol** (institutional co-signer is online anyway). |
| 7 | No audit yet | Certain | High | Two named audit firms (circuit + contract), 4+3 weeks separate scopes, 2-week fix window each, no mainnet without clean reports. |
| 8 | Pillar 2 securities/AML/sanctions exposure | Medium | High | Track B funds gated on §9 compliance gates; SDF reviewer designated; B1 held until C1 in hand. Track A independent. |
| 9 | BN254 ~100-bit pairing security degrades within RWA token lifespan | Low | High | Migration plan in Appendix A; VK registry supports parallel BLS12-381 verifier. |
| 10 | OZ `stellar-accounts` library upgrades break pending recoveries | Medium | Medium | `g2c-oz-compat` shim crate; storage-layout XDR hash CI regression test; library pinned at A1. |
| 11 | Cancel-loop attack — stolen passkey blocks legitimate recovery indefinitely | Medium | High | `cancel_recovery` requires both WebAuthn signature AND fresh seed-knowledge proof; cap of 2 cancels per initiation; 24h cooldown between cancels; cancel events emitted to user notification channel. |
| 12 | TTL archival race — pending-rotation state archived during 30-day window | Medium | Critical | Per-account recovery deposit at enrollment covers ~40 days of TTL extension; `complete_recovery` fails-closed if rent exhausted; `RestoreFootprint` ordering specified (restored state cannot retroactively re-open a completed/cancelled recovery). |
| 13 | Pubkey encoding malleability | Medium | Critical | In-circuit on-curve check on P-256 pubkey; canonical-limb decomposition (`hi*2^128 + lo < secp256r1_modulus`); A1 unit tests cover non-canonical encodings as adversarial inputs. |
| 14 | Pre-enrollment squatting — attacker phishes mnemonic before user enrolls, registers commitment first | Low | High | Enrollment binds `commitment` to `account_id`; an unenrolled C-address has no commitment to attack; user education at enrollment emphasizes sequential ordering (create account → write down phrase → register commitment, all in one flow). |
| 15 | VK-upgrade UI coercion attack | Medium | High | "Required VK refresh" prompts disallowed in integrator's-guide UX standard; VK upgrades surface as opt-in only with 14-day visibility; 2 independent on-chain monitors required to publish `VKUpgradeProposed` alerts before any wallet defaults to opt-in. |
| 16 | bb.js supply-chain attack | Medium | Critical | SRI hash for the `bb.js` bundle pinned in the wallet HTML; reproducible-build verification published in A5; npm registry compromise mitigated by vendoring the WASM artifact. |
| 17 | Recovery-card QR exfiltration | Medium | High | Recovery card contains C-address only, NEVER the seed. Seed lives on a separate paper artifact never digitized. A5 acceptance criterion for the export tooling. |
| 18 | Recovery-deposit griefing | Low | Medium | Per-account recovery initiation rate-limit: max 3 `initiate_recovery` per 90-day rolling window. |
| 19 | mysoroban.xyz origin compromise | Low | Critical | Acknowledged as a top-of-stack risk that SRI cannot defend against. Mitigations in scope: DNSSEC, COOP/COEP, strict CSP, signed-loader bootstrap. **Out of scope (post-grant):** native-app option + hardware-wallet integration path; user education that recovery from a compromised origin requires waiting + filing a SmartAccount disclosure. Risk explicitly enumerated so SDF and integrators can plan defense-in-depth. |
| 20 | Kill-switch DoS | Low | High | First pause is 1-of-5 for 7 days; second pause within 30 days requires 4-of-5. One compromised signer cannot indefinitely block emergency VK rotation. |
| 21 | VK upgrade governance bias | Low | High | SDF + Stellar Foundation external signers treated as one effective vote *for collusion analysis only*; the on-chain rule is 3-of-5 with the diversified-quorum predicate. Every accepted quorum requires at least one outside-Group-A signature, so SDF+SF collusion still requires OZ or community consent. Documented in A2 governance design + grant agreement. |

---

## 9. Pillar 2 compliance gates

Track B funds gated on the following deliverables from the neftwerk team. Reviewed by **a named SDF compliance reviewer** (designated at grant signing).

| Gate | Deliverable | Track B effect |
|---|---|---|
| C1 | MSB / NY VTL / EU VASP / MiCA classification opinions for ARTS-1 primary issuance | **B1 held until C1 in SDF reviewer's hands** |
| C2 | On-chain freeze function pursuant to documented legal process. Implemented as B6 acceptance. | Blocking for B6 |
| C3 | Howey opinion for single-edition ARTS-1 tokens, including treatment of enforced royalties; explicit no-fractionalization covenant | Blocking for B6, reviewed by SDF reviewer |
| C4 | Geographic scoping plan: Reg-S-style geofence during grant period; KYC threshold for secondary transfers above $X | Blocking for B6 |
| C5 | Estate-planning disclosure flow at enrollment | Blocking for B7 |

If any gate is not met by week 9, Track B is paused; Track A continues unaffected.

**Branding:** SDF logo and "funded by SDF" language do not appear on Pillar 2 marketing pages until C1–C4 are implemented.

---

## 10. Why bundle, expanded

The engineering case for one grant over two:

- **One verifier crate** (`ultrahonk-soroban-verifier`) serves both pillars.
- **One browser proving harness** (bb.js + WebWorker + `/recover/` route).
- **One Poseidon2 parameter set**, domain-separated per use, audited once.
- **One VK registry + one governance** mechanism.
- **One audit RFP** covering circuits + contracts + integration.

We deliver a cross-repo dependency graph as the first artifact of A1.

Bundling saves engineering time **only because** Track A and Track B share a substrate. They do not share business logic, end-users, regulatory posture, or success criteria. Disbursement is therefore independent (§7).

---

## 11. Universality

The recovery circuit is a key-rotation primitive any Soroban Smart Account can adopt; neftwerk is the second consumer, not the only one. Concrete future consumers:

- Custody wallets needing non-custodial paper-mnemonic recovery
- RWA issuers (real estate, supply-chain, carbon credits) needing ownership privacy with provable transfer history
- Compliance-sensitive institutions wanting counterparty privacy without surrendering provability

A6 deliverable includes a one-page **integrator's guide**: the minimal interface a Soroban contract has to implement to consume the ZK recovery primitive. **At least one external wallet integrator's signed integration agreement is required at A6.**

---

## 12. Roadmap: fast-recovery path (post-grant)

The 30-day timelock is incompatible with high-velocity collector use cases (insurance renewals, museum loans, estate updates). Post-grant deliverable: a Shamir-style fast-recovery path with the threshold scheme `(collector + 1-of-{gallery, Neftwerk})` — collector's share is mandatory, foreclosing institutional-only recovery. Additive on top of the current recovery primitive; no breaking change.

---

## 13. References / repo file map

### g2c
- `g2c/APPLICATION.md`
- `g2c/ARCHITECTURE.md`
- `g2c/contracts/smart-account/src/contract.rs`
- `g2c/contracts/factory/src/contract.rs`
- `g2c/contracts/webauthn-verifier/src/contract.rs`

### ZK substrate / proof-of-concept
- `zk/rs-soroban-ultrahonk/README.md`
- `zk/rs-soroban-ultrahonk/ultrahonk-soroban-verifier/src/`
- `zk/rs-soroban-ultrahonk/<nullifier_set_primitive>/` (renamed before kickoff)
- `zk/soroban-zk-demo/README.md` and `test_e2e.sh`
- `zk/soroban-zk-demo/circuits/ownership_proof/src/main.nr` (to be brought into spec at A0)

### Neftwerk
- `zk/neftwerk/Neftwerk_White_Paper_Summary.md`
- `zk/neftwerk/Neftwerk_Creation_Privacy_Flow.md`
- `zk/neftwerk/Neftwerk_Sale_Privacy_Flow.md`
- `zk/neftwerk/milestones.md`
- `zk/neftwerk/review-technical.md`
- `zk/neftwerk/review-legal.md`
- `zk/neftwerk/review-gallery.md`

### External
- Noir: <https://noir-lang.org/>
- Barretenberg / UltraHonk: <https://github.com/AztecProtocol/aztec-packages>
- `noir-bigcurve`, `noir_ecdsa`: `noir-lang` org
- OpenZeppelin Stellar Contracts: `stellar-contracts-OZ/`

---

## Appendix A — Cryptographic parameters

### A.1 — Encoding spec

All Field-bound inputs to Poseidon2 follow these rules:

- **65-byte uncompressed P-256 pubkey** (`0x04 ‖ x ‖ y`): split into 5 limbs of ≤128 bits via big-endian decomposition. Order: `(prefix_byte, x_hi, x_lo, y_hi, y_lo)`. Each limb range-checked `< 2^128` in-circuit. Composition check: `prefix == 0x04` and `x = x_hi * 2^128 + x_lo` and `y = y_hi * 2^128 + y_lo` and `(x, y)` lies on secp256r1.
- **`account_id`** (Stellar StrKey / `BytesN<32>`): 32 bytes interpreted as 2 BN254 field elements (`hi`, `lo`) via 16-byte big-endian split. Range-checked.
- **`network_passphrase`** (variable string): SHA-256 hashed, then encoded as 2 BN254 field elements via the same 16-byte split.
- **`contract_addr`** (Stellar contract address / `BytesN<32>`): same as `account_id`.
- **`nonce`**: 8 bytes BE → 1 Field.
- **`timelock_duration`**: 4 bytes BE seconds → 1 Field.
- **`token_id`**: 32 bytes → 2 Field elements (16-byte split).

### A.2 — Domain separators

- `domain_sep_commitment = Poseidon2("g2c-recovery-v1-commitment")`
- `domain_sep_recovery = Poseidon2("g2c-recovery-v1-auth-hash")`
- `domain_sep_cancel = Poseidon2("g2c-recovery-v1-cancel-hash")`
- `domain_sep_arts1_mint = Poseidon2("arts1-mint-v1")`
- `domain_sep_arts1_own = Poseidon2("arts1-ownership-v1")`
- `domain_sep_arts1_rec = Poseidon2("arts1-recovery-v1")`
- `domain_sep_nullifier = Poseidon2("nullifier-set-primitive-v1")`

All domain separators are passed as the **first** input element of every Poseidon2 call. The Aztec Poseidon2 sponge is multi-input-collision-resistant under standard assumptions.

### A.3 — Other parameters

- **Curve.** BN254 (alt_bn128), 254-bit prime field. ~100-bit pairing security.
- **Migration plan.** VK registry supports parallel BLS12-381 verifier deployment without disrupting BN254-pinned accounts. BLS12-381 prototype committed as post-grant deliverable; full migration estimated 2027–2028.
- **HKDF.** SHA-256 PRF; `ikm = BIP-39 seed`, `salt = network_passphrase` (raw string), `info = "g2c-recovery-v1" ‖ account_id` (33 bytes), output length = 32 bytes, then reduced modulo BN254 scalar order. Bias from naive `mod q` is ~2⁻²⁵² — acceptable.
- **Proof system.** UltraHonk via Barretenberg; Fiat-Shamir, Sumcheck, Shplonk.
- **Sizes (measured).** VK ≈ 1.9 KB, proof ≈ 3.7 KB.
- **G2 SRS.** Aztec's powers-of-tau ceremony output, hardcoded in `ec.rs:7-27`.

---

## Appendix B — Prior art

- **Semaphore-style group-membership patterns.** Source of the commit-then-prove + nullifier-set pattern.
- **Aztec Protocol.** UltraHonk's home.
- **OpenZeppelin Stellar SmartAccount.** Verifier-per-signer + Policy pattern.
- **BIP-39.** Mnemonic format for offline seed storage.
- **Argent / Safe wallet recovery.** Reference for timelock + guardian patterns; we deliberately use cryptographic guardianship instead of social.
