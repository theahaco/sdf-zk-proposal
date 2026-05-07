# Technical Appendix — ZK Proofs on Soroban: Recovery + Privacy Bundle

Companion to `SDF_PROPOSAL_ONE_PAGER.md`. This is for the SDF protocol team and any reviewer who wants the architecture, milestones, risks, and references in full.

This revision incorporates feedback from a 10-person internal/external review. Material changes vs. v1: a pre-grant benchmark milestone (A0), a dedicated `g2c-recovery-controller` contract for pending-rotation state, a fully-specified `auth_hash` preimage, a VK upgrade governance design, hard contractual track separation, named audit firms with separate scopes, a concrete budget, and a Pillar 2 compliance-gate section.

---

## 1. Executive recap

We are proposing a single coordinated grant that lands two ZK applications on Stellar, with hard contractual track separation:

- **Pillar 1 — g2c: ZK account recovery as a universal primitive.** A BIP-39 seed-derived `owner_secret` is committed on-chain at enrollment as `Poseidon2(owner_secret)`. On recovery, a Noir/UltraHonk proof bound to a fully-specified `auth_hash` triggers a 30-day time-locked key rotation through a new `g2c-recovery-controller` contract that integrates with g2c's OpenZeppelin SmartAccount surface. During the timelock window the WebAuthn signer is forbidden from removing the ZK signer.
- **Pillar 2 — neftwerk: ZK privacy as a sector validator.** Three Noir circuits (`arts1_mint`, `arts1_ownership`, `arts1_recovery`) implement "private title, public provenance" for the ARTS-1 token standard, with the recovery circuit being the same primitive as Pillar 1.

Framing: **Universal primitive in g2c + sector application in neftwerk.** The shared substrate is **Noir + UltraHonk + Poseidon2 + BN254 + Soroban 25.x**, and we estimate **~30–40% engineering shared** across the two tracks. Track A and Track B funds are disbursed independently; an A0 benchmark failure or a Pillar 2 compliance-gate slip does not affect Track A.

---

## 2. Shared ZK substrate

| Layer | Choice | Notes |
|---|---|---|
| Circuit DSL | Noir (`nargo` 1.0.0-beta.x) | Mature, audited tooling; first-class UltraHonk backend |
| Proving system | UltraHonk via Aztec's Barretenberg (`bb` 0.87.x) | Universal SRS (no per-circuit ceremony); succinct verification |
| Hash | Poseidon2 over the BN254 scalar field, **Aztec-default parameter set, pinned to `bb` 0.87.x** | Domain-separated per use (commitment vs. recovery vs. nullifier) |
| Curve | BN254 | ~100-bit pairing security; migration plan to BLS12-381 noted in §13 |
| Verifier | `ultrahonk-soroban-verifier` Rust crate (no_std + alloc) | Deployed to Stellar testnet today |
| Soroban SDK | 25.x (workspace-pinned) | Tracks current mainnet protocol; OZ `stellar-accounts` pinned to a specific commit per §3 |
| Browser prover | `bb.js` (WASM), lazy-loaded on `/recover/` route only | Cross-origin isolation requirement addressed in §5.3 |

**Why UltraHonk over Groth16, Plonk, Halo2, RISC0/SP1?** UltraHonk's universal SRS removes the per-circuit ceremony — the load-bearing requirement for a universal recovery primitive that wallets will instantiate against bespoke circuits. We will benchmark Groth16-on-Soroban as part of A0 to publish the verification-CPU delta; if Groth16 verifies at <1/3 the Soroban budget for the recovery circuit, we fund a Groth16 path for Pillar 1's fixed circuits and keep UltraHonk for Pillar 2's evolving ones.

---

## 3. Architecture: ZK signer plug-in to g2c

g2c's SmartAccount is built on OpenZeppelin's `stellar-accounts` library and implements `CustomAccountInterface` + `SmartAccount` + `ExecutionEntryPoint`. The integration introduces **two new contracts**:

- **`g2c-zk-recovery-verifier`** — stateless `Verifier` implementation following the same trait shape as the existing `g2c-webauthn-verifier`. Takes `KeyData = Bytes32` (the Poseidon2 commitment) and `SigData = (proof_bytes, public_inputs)`. Cross-invokes the shared `ultrahonk-soroban-verifier` crate.
- **`g2c-recovery-controller`** — stateful contract that owns the pending-rotation state machine (see §4). Per-account persistent storage with explicit TTL extension to survive the 30-day window. Provides `initiate_recovery(account, proof, sig_data, new_pubkey)`, `cancel_recovery(account)`, and `complete_recovery(account)` entrypoints.

### Critical safety property: signer-removal guard

The existing OZ `add_signer` / `remove_signer` (`smart-account/contract.rs:105–113`) requires only `current_contract_address().require_auth()` — which a stolen WebAuthn signer can satisfy. **A naive integration is bypassable: the attacker rotates away the ZK signer.**

The fix is a **policy-level guard registered as a `ContextRule`**:

```
during pending recovery → remove_signer(zk_recovery_signer) is forbidden
during pending recovery → add_signer(*) requires both the WebAuthn signer
                          AND the recovery_controller's auth
```

The `g2c-recovery-controller` is registered as an additional admin signer at enrollment. Its `require_auth()` is satisfied only after the timelock expires AND the recovery proof verified. This is enforced as an OZ `Policy`, not application logic — meaning it cannot be bypassed by `execute()`.

### Updated call path on recovery

```
Submitter → SmartAccount.execute(rotate_signer)
          → __check_auth → OZ do_check_auth
          → g2c-zk-recovery-verifier.verify(payload, commitment, sig_data)
          → ultrahonk-soroban-verifier.verify_proof(public_inputs, proof_bytes)
          → returns OK → recovery-controller stores pending rotation (persistent storage, TTL extended)
          → 30-day window begins
          → during window: remove_signer(zk) blocked by policy
          → after window: complete_recovery() rotates signer atomically
```

OZ `stellar-accounts` is pinned to commit `<TBD-at-A2>` for the duration of the grant. Any upgrade requires re-pinning and a regression test against the pending-rotation state machine.

---

## 4. Recovery flow end-to-end (Pillar 1)

Six steps. (1)–(2) at SmartAccount enrollment; (3)–(6) on recovery.

1. **Seed at enrollment.** During g2c onboarding the wallet asks the user to write down a BIP-39 mnemonic (passphrase optional, explicitly supported). The seed never touches the network.
2. **Commitment on-chain.** The wallet derives `owner_secret = HKDF-SHA256(ikm = seed, salt = network_passphrase, info = "g2c-recovery-v1" ‖ account_id)`, reduces modulo the BN254 scalar field, and submits `commitment = Poseidon2(owner_secret)` to the `recovery-controller` registered against the SmartAccount.
3. **Recovery trigger.** User has lost their passkey. Opens `mysoroban.xyz/recover/` (cross-account entry — finds the C-address from the seed, no need to remember the subdomain). Re-enters BIP-39 mnemonic with wordlist autocomplete + checksum validation + passphrase prompt.
4. **Proof generation in the browser.** Using `bb.js` (WASM, lazy-loaded), the browser generates a Noir/UltraHonk proof for the circuit:
   ```
   public inputs:
     commitment           : Field    // pinned at enrollment
     auth_hash            : Field    // see preimage spec below
   private inputs:
     owner_secret         : Field
   asserts:
     Poseidon2(owner_secret) == commitment
     auth_hash             == provided_auth_hash   // bound to public input
   ```
   The **`auth_hash` preimage is fully specified**:
   ```
   auth_hash = Poseidon2(
       domain_sep,            // = Poseidon2("g2c-recovery-v1") – constant per circuit version
       account_id,            // 32 bytes – Stellar StrKey decoded
       network_passphrase,    // 32 bytes – SHA-256 of the passphrase string
       contract_addr,         // 32 bytes – recovery-controller address
       new_signer_pubkey,     // 65 bytes – uncompressed P-256, split into 128-bit limbs (range-checked in-circuit)
       nonce,                 // 8 bytes – monotonic counter from recovery-controller
       timelock_duration      // 4 bytes – chosen at enrollment, minimum 7 days
   )
   ```
5. **On-chain submission.** Wallet calls `recovery-controller.initiate_recovery(account, proof, sig_data, new_pubkey)`. The controller verifies the proof, validates `nonce` is monotonic, confirms the public-input `auth_hash` matches the recomputed value, stores the pending rotation in persistent storage with explicit 35-day TTL, emits a `RecoveryInitiated` event, and starts the timelock.
6. **30-day timelock + completion.** During the window: the original passkey, if present, can call `cancel_recovery()` (signed by the WebAuthn signer over a domain-separated cancellation hash). After the window: anyone can call `complete_recovery()`, which atomically rotates the signer in the SmartAccount and clears the pending state.

The timelock duration is per-account, **floored at 7 days** to prevent attackers who phish the mnemonic from setting a 1-hour window at enrollment.

`RecoveryInitiated` events MUST be emitted; wallets are required to subscribe and notify users via push/email/SMS through whatever channel the user opted into at enrollment.

---

## 5. Privacy flow end-to-end (Pillar 2)

Neftwerk's three circuits, with explicit input/output specs.

### `arts1_mint` (heaviest)

Verifies at minting that **both** Neftwerk (institutional co-signer) and the collector authorized the mint over the same `provenance_secret`.

- **Public inputs:** `commitment` (output), `provenance_root`, `auth_hash`, `mint_id`, `token_id`.
- **Private inputs:** institutional ECDSA signature, collector ECDSA signature (both over `auth_hash`), `provenance_secret`, collector `owner_secret`, collector pubkey limbs.
- **Asserts:** institutional pubkey on-curve and `verify_ecdsa(institutional_pk, auth_hash, sig)`; same for collector; `auth_hash` binds `mint_id`, `token_id`, both pubkeys, and `provenance_secret`; `commitment = Poseidon2(domain_sep_mint, collector_pk_limbs, owner_secret)`; `provenance_root` derived from `provenance_secret` and metadata CID.

The dual-ECDSA load is the highest-risk circuit. Its constraint count is the load-bearing question for A0.

### `arts1_ownership`

Used at every transfer/sale. The collector proves they know the secrets behind the on-chain commitment AND that their passkey signed the current transaction.

- **Public inputs:** `commitment`, `auth_hash`, `token_id`.
- **Private inputs:** `owner_secret`, collector pubkey limbs, ECDSA signature over `auth_hash`.
- **Asserts:** `auth_hash` includes `token_id`, transaction nonce, contract address, function selector, call args; `commitment == Poseidon2(domain_sep_own, collector_pk_limbs, owner_secret)`; signature valid.

### `arts1_recovery`

The same primitive as Pillar 1, applied to a per-token commitment with token-scoping.

- **Public inputs:** `owner_secret_commitment`, `auth_hash`, `token_id`, `new_owner_pubkey_limbs`.
- **Private inputs:** `owner_secret`.
- **Asserts:** `Poseidon2(domain_sep_recovery, owner_secret) == owner_secret_commitment`; `auth_hash` includes `token_id`, contract address, `new_owner_pubkey_limbs`, monotonic nonce.

The token-id binding closes a cross-token replay attack flagged by the cryptography reviewer.

### Privacy properties and known gaps

The on-chain ARTS-1 contract knows *what* a token is (via the metadata CID) but not its price or its collector. Price/split logic is off-chain in Neftwerk's database and injected at sale time as transaction arguments ("late-binding").

**Known gap:** an observer can infer the price by watching USDC outflows on settlement. **Roadmap commitment:** a fourth circuit `arts1_settlement` using the nullifier-set primitive of §6 (out of scope for this grant; explicitly named here as the next milestone).

---

## 6. Deployment evidence

### `soroban-zk-demo` — full E2E on testnet

- `./test_e2e.sh` runs build → deploy → prove → verify on testnet.
- Three-contract pattern: `zk-contract` (commitment + cross-call), `ultrahonk-soroban-contract` (proof verifier), `zk-factory-contract` (factory).
- **Note:** the current on-disk circuit (`circuits/ownership_proof/src/main.nr`) is a hash-knowledge demo and does not yet bind the public input to the recovery circuit's commitment. **A0 deliverable #1 is to bring the on-disk circuit into agreement with §4's spec** before any other circuit work.

### `rs-soroban-ultrahonk` — verifier crate

- Soroban contract wrapping the UltraHonk verifier; integration tests for simple and Fibonacci-chain circuits.
- Cost-measurement script in `scripts/invoke_ultrahonk/` for testnet gas readings.
- VK currently immutable at `__constructor`; A2 delivers the upgrade governance.

### Nullifier-set + Merkle-frontier primitive

- Reference implementation with depth-20 tree, insertion proofs, and nullifier-set state on Soroban.
- Demonstrates the substrate for revocation, replay protection, and future batched settlement (`arts1_settlement`).
- Repo directory will be renamed to `nullifier_set_primitive` before grant kickoff.

---

## 7. Milestone breakdown

Two parallel tracks plus a shared pre-grant benchmark. All milestones have measurable acceptance criteria; tranche release is gated on each milestone's acceptance.

### A0 — Pre-grant Soroban gas benchmark (weeks 1–2, $30k flat)

**Pre-grant gating milestone**: paid as a flat fee on completion, regardless of whether the rest of the grant proceeds. Track A and Track B funds are not released until A0 publishes numbers.

**Deliverables:**
- Published CPU instruction count, memory bytes, and resource fee (in stroops) for `verify_proof` on testnet, against three circuits: simple-1-input, Fibonacci-chain, and an `arts1_mint`-shaped reference circuit (~50k constraints, dual-ECDSA placeholder).
- Same measurements for a Groth16 verifier on Soroban as comparison.
- Browser proving: peak heap, prove time, WASM bundle size on iPhone 14, Pixel 7, 2020 MacBook Air.
- The on-disk `ownership_proof` circuit brought into spec agreement.

**Pass criterion:** `arts1_mint`-shaped circuit verification fits within 70M instructions on Soroban (3× headroom under the 100M cap). If pass: Track A and Track B fund. If fail: SDF and team meet to decide between (a) descope (Pillar 1 with smaller circuit only), (b) re-architect with recursive aggregation, (c) terminate the grant.

### Track A — g2c ZK recovery ($150k tranched 10/20/30/40)

| # | Milestone | Weeks | Acceptance criteria |
|---|---|---|---|
| A1 | Spec doc + Noir circuit | 2–4 | Single PDF pinning Poseidon2 parameter set, limb canonicalization rules, `auth_hash` preimage byte layout, HKDF parameters, field-reduction rule. Recovery circuit compiles; replay/cross-account/cross-network unit tests pass. |
| A2 | Verifier crate v2 + VK governance + recovery-controller | 4–7 | `ultrahonk-soroban-verifier` v2 with VK registry pattern (circuit_id → VK). VK upgrade governance: 3-of-5 multi-sig with named SDF-team + 2 founder + 2 community signers, 14-day timelock on upgrades, opt-in per account, orphan-token migration documented. `g2c-recovery-controller` deployed to testnet with full state machine. |
| A3 | Generalized factory | 6–7 | g2c-factory accepts multi-signer construction and pre-registered ZK signer at `create_account`. |
| A4 | SmartAccount integration + signer-removal guard | 7–9 | OZ `Policy` registered preventing WebAuthn-signer eviction of ZK-signer during pending rotation. Differential tests covering: stolen-passkey bypass attempt, double-recovery, expiration, cancellation. |
| A5 | Browser proving UX | 8–10 | <5 min recovery proof on 2020 MacBook Air, <8 min on Pixel 7. BIP-39 wordlist autocomplete, checksum, passphrase, language selection. Cross-account `mysoroban.xyz/recover/` entry. Cross-origin isolation deployment plan validated. Graduated unlock during timelock (read-only access). |
| A6 | Audit-ready freeze + mainnet launch | 10–14 | All contracts and circuits frozen for 4 weeks circuit audit + 3 weeks contract audit + 2-week fix window. Mainnet deployment of recovery-controller + verifier with at least one external wallet integrator (LOI required at A4). |

### Track B — neftwerk ZK privacy ($200k tranched 10/20/30/40)

**Pillar 2 funding is gated on Pillar 2 compliance gates (§11) AND A0 pass.**

| # | Milestone | Weeks | Acceptance criteria |
|---|---|---|---|
| B0 | Coinflow Soroban transaction-attachment spike + dual-ECDSA feasibility | 2–3 | Coinflow ↔ Soroban primary-sale flow validated on testnet. Dual-ECDSA constraint count published. |
| B1 | Three-circuit spec (`arts1_mint`, `arts1_ownership`, `arts1_recovery`) | 3–5 | Single spec PDF following A1's format. All three circuits compile. |
| B2 | `arts1_ownership` (lightest, validates pattern) | 5–7 | Compiled circuit + verifier integration; replay tests pass. |
| B4 | `arts1_mint` dual-ECDSA | 7–10 | Compiled circuit; verification cost within A0 budget; stretch goal — descope to single-ECDSA collector signature if A0 left <2× headroom. |
| B5 | `arts1_recovery` (parameterized A1 primitive) | 9–10 | Same Noir circuit as A1, parameterized for `token_id`. |
| B6 | ARTS-1 token contract + browser pipeline + royalty enforcement | 10–13 | End-to-end mint and transfer with privacy on testnet. Royalty-locked secondary transfers as acceptance criterion. |
| B7 | Live ARTS-1 pilot mint with **Margaret Ellison Gallery** | 13–14 | Conditional on signed MoU at A0 completion; 5 works, 5 collectors, 3-month observation. **If MoU not secured by week 6, B7 is reframed as a fully-rigged testnet pilot ready for partner activation, no schedule slip to Track A.** |

### Shared dependencies

- A2 deliverables feed B2/B4/B5 (verifier registry pattern is consumed by all neftwerk circuits).
- Browser proving SDK (A5) is consumed by B6.
- Audit firms cover both tracks (see §9).

---

## 8. Budget

**Total: $380k over 14 weeks.**

| Line item | Amount | Tranching |
|---|---|---|
| A0 — pre-grant benchmark | $30k | flat on completion |
| Track A — g2c ZK recovery | $150k | 10% / 20% / 30% / 40% on A1 / A2 / A4 / A6 |
| Track B — neftwerk ZK privacy | $200k | 10% / 20% / 30% / 40% on B1 / B4 / B6 / B7, gated on A0 pass + Pillar 2 compliance gates |
| **Total** | **$380k** | |

**Audit credits requested separately** from SDF's standard pool. Two auditors, two scopes:

- **Circuit audit** (4-week scope): zkSecurity, Veridise, or Spearbit ZK practice. Scope: all four Noir circuits, Poseidon2 parameter set, ECDSA-in-circuit gadgets, limb canonicalization, replay protection, `auth_hash` preimage binding.
- **Contract audit** (3-week scope): OtterSec, Trail of Bits, or Halborn. Scope: `g2c-zk-recovery-verifier`, `g2c-recovery-controller`, generalized factory changes, ARTS-1 token contract, OZ `Policy` for signer-removal guard.

2-week fix window after each audit. No mainnet deployment until both audits clean.

**Team:** 3 FTE — one ZK/circuit lead, one Soroban/Rust contracts engineer, one frontend/browser-prover engineer. Names and FTE percentages provided to SDF on grant signing.

**Track independence clause:** Track A funds are NOT collateralized by Track B's progress. Track A continues to disburse on its own milestones even if Track B is paused for compliance, A0 fails for `arts1_mint`-sized circuits, or B7's gallery MoU does not materialize. Conversely, Track B is held back if Track A's audit findings invalidate the shared verifier crate.

---

## 9. Risk and mitigation table

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | UltraHonk gas/budget on Soroban exceeds the 100M-instruction cap for `arts1_mint`-sized circuits | Medium | High | **A0 measurement gate paid as a flat pre-grant fee.** Published numbers before any other funds release. Failure paths defined: descope, re-architect with aggregation, or terminate. |
| 2 | Larger circuits (~50+ constraints, dual-ECDSA in `arts1_mint`) untested | Medium | Medium | A0 includes an `arts1_mint`-shaped reference circuit. B0 then runs the real dual-ECDSA feasibility spike. B4 has an explicit single-ECDSA descope path. |
| 3 | Recovery proof front-running or replay across accounts/networks | Medium | High | `auth_hash` preimage fully specified in §4 with `domain_sep`, `account_id`, `network_passphrase`, `contract_addr`, `new_signer_pubkey`, `nonce`, `timelock_duration`. Unit tests in A1 cover all five replay vectors. |
| 4 | **WebAuthn signer is rotated by an attacker who removes the ZK signer first** | Medium | Critical | OZ `Policy` (registered at enrollment) forbids `remove_signer(zk)` during pending rotation; `add_signer` during pending rotation requires both signers' auth. Differential test in A4 explicitly attempts the bypass. |
| 5 | VK governance attack (malicious VK swap) | Low | Catastrophic | A2 delivers a **3-of-5 multi-sig VK registry** with named signers (SDF-team + founder + community), 14-day on-chain timelock for any VK upgrade, opt-in per account. Existing accounts pinned to the VK they enrolled with; opt-in to new VKs is a separate user action. |
| 6 | Browser proving fails on mobile or under cross-origin isolation | Medium | Medium | A5 publishes measured numbers per device tier. WASM lazy-loaded only on `/recover/`. COOP/COEP headers + Cloudflare Worker compatibility validated. **Server-assisted prover fallback removed from this proposal**; if needed in v2, designed as MPC proving where the server provably never sees the secret. |
| 7 | No audit yet | Certain | High | Two named auditors (circuit + contract), 4+3 weeks separate scopes, 2-week fix window each, no mainnet without clean reports. |
| 8 | Pillar 2 securities/AML/sanctions exposure | Medium | High | Track B funds gated on Pillar 2 compliance gates (§11). Track A is independent and continues regardless. |
| 9 | BN254 ~100-bit pairing security degrades within RWA token lifespan | Low | High | Migration plan documented in §13 (Appendix A). VK registry supports adding BLS12-381 verifiers without disrupting BN254-pinned accounts. |
| 10 | OZ `stellar-accounts` library upgrades break pending recoveries | Medium | Medium | Library pinned to a specific commit at A2; upgrade requires regression test against the pending-rotation state machine; existing accounts opt-in. |

---

## 10. Why bundle, expanded

The engineering case for one grant over two:

- **One verifier crate** (`ultrahonk-soroban-verifier`) serves both g2c-recovery and neftwerk-privacy.
- **One browser proving harness** (bb.js + WebWorker + `/recover/` route).
- **One Poseidon2 parameter set**, domain-separated per use, audited once.
- **One VK registry**, one storage layout, one proof format.
- **One audit RFP** covering circuits + contracts + integration, scoped across both tracks.

We deliver a cross-repo dependency graph as the first artifact of A1.

The reviewer-friendly version of "why bundle" is: bundling saves engineering time **only** because Track A and Track B share a substrate. They do not share business logic, end-users, regulatory posture, or success criteria. Disbursement is therefore independent (§8).

---

## 11. Pillar 2 compliance gates

Track B funds are gated on the following deliverables from the neftwerk team, independent of any technical milestone. These items are exactly the launch-blocking concerns identified in neftwerk's own legal review.

| Gate | Deliverable | Track B effect |
|---|---|---|
| C1 | MSB / NY VTL / EU VASP / MiCA classification opinions for ARTS-1 primary issuance | No Track B funds release until delivered |
| C2 | On-chain freeze function pursuant to documented legal process (court orders, OFAC SDN matches, probate). Implemented on the ARTS-1 token contract as B6 acceptance criterion. | Blocking for B6 |
| C3 | Howey opinion for single-edition ARTS-1 tokens, including treatment of enforced royalties; explicit no-fractionalization covenant | Blocking for B6 |
| C4 | Geographic scoping plan: Reg-S-style geofence during grant period; KYC threshold for secondary transfers above $X | Blocking for B6 |
| C5 | Estate-planning disclosure flow at enrollment | Blocking for B7 |

If any gate is not met by week 9, Track B is paused; Track A continues unaffected.

**Branding:** SDF logo and "funded by SDF" language do not appear on Pillar 2 marketing pages until C1–C4 are implemented.

---

## 12. Universality argument

The recovery circuit is a key-rotation primitive any Soroban Smart Account can adopt; neftwerk is the second consumer, not the only one. Concrete future consumers:

- **Custody wallets** wanting non-custodial paper-mnemonic recovery.
- **RWA issuers** (real estate, supply-chain, carbon credits) needing ownership privacy with provable transfer history.
- **Compliance-sensitive institutions** wanting counterparty privacy without surrendering provability to a custodian.

A6 deliverable includes a one-page **integrator's guide**: the minimal interface a Soroban contract has to implement to consume the ZK recovery primitive. **At least one external wallet integrator's LOI is required at A4** as a tranche-release condition.

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
- `zk/neftwerk/review-gallery.md` (Margaret Ellison's pilot offer)

### External
- Noir: <https://noir-lang.org/>
- Barretenberg / UltraHonk: <https://github.com/AztecProtocol/aztec-packages>
- OpenZeppelin Stellar Contracts: `stellar-contracts-OZ/`

---

## Appendix A — Cryptographic parameters

- **Curve.** BN254 (alt_bn128), 254-bit prime field, pairing-friendly. ~100-bit pairing security after recent TNFS analyses.
- **Migration plan.** VK registry (A2) supports parallel BLS12-381 verifier deployment without disrupting BN254-pinned accounts. We commit to publishing a BLS12-381 prototype as a post-grant deliverable; full migration estimated 2027–2028.
- **Hash.** Poseidon2 over BN254 scalar field, **Aztec-default parameter set** (`bb` 0.87.x), with **domain separation** per use:
  - `domain_sep_commitment = Poseidon2("g2c-recovery-v1-commitment")`
  - `domain_sep_recovery = Poseidon2("g2c-recovery-v1-auth-hash")`
  - `domain_sep_arts1_mint = Poseidon2("arts1-mint-v1")`
  - `domain_sep_arts1_own = Poseidon2("arts1-ownership-v1")`
  - `domain_sep_arts1_rec = Poseidon2("arts1-recovery-v1")`
- **Commitment scheme.** `commitment = Poseidon2(domain_sep, secret_limbs)` with all field-larger-than-BN254-scalar inputs split into 128-bit limbs, **range-checked in-circuit**.
- **HKDF.** SHA-256 as PRF; `ikm = BIP-39 seed`, `salt = Stellar network passphrase`, `info = "g2c-recovery-v1" ‖ account_id`. Output reduced modulo BN254 scalar order.
- **Proof system.** UltraHonk via Barretenberg; Fiat-Shamir transcript, Sumcheck, Shplonk multi-opening.
- **Sizes (measured).** VK ≈ 1.9 KB, proof ≈ 3.7 KB (456 BN254 field elements). Public-input count varies per circuit.
- **G2 SRS.** Aztec's powers-of-tau ceremony output, hardcoded in `ec.rs:7-27`. SDF acceptance of this SRS is a discussion point in the 30-minute review.

---

## Appendix B — Prior art

- **Semaphore-style group-membership patterns.** Source of the commit-then-prove + nullifier-set pattern. Our nullifier-set primitive (§6) ports this design to Soroban with a depth-20 Merkle tree.
- **Aztec Protocol.** UltraHonk's home; we follow Aztec's circuit conventions for Poseidon2 and ECDSA verification.
- **OpenZeppelin Stellar SmartAccount.** Verifier-per-signer pattern; our extension mechanism.
- **BIP-39.** Mnemonic format for offline seed storage; we layer HKDF for account-derivation.

---

*End of appendix. See `SDF_PROPOSAL_ONE_PAGER.md` for the executive summary.*
