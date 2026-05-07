# ZK Proofs on Soroban: Recovery + Privacy Bundle

**A proposal to the Stellar Development Foundation**

ZK proofs on Soroban are no longer hypothetical. We have a Noir/UltraHonk verifier validating real proofs on Stellar testnet today, and we are proposing to land it as a **recovery primitive in g2c** and a **privacy circuit in neftwerk** — under one coordinated grant with hard track separation.

## Why now

Soroban Smart Accounts have shipped, and OpenZeppelin's `CustomAccountInterface` is the de-facto integration surface (g2c uses it today). Two structural gaps remain:

1. **Recovery.** g2c's current story is "register a backup passkey." That is not enough — there is no key-rotation path for the lone-passkey case.
2. **Privacy.** Stellar lacks a native primitive for hiding ownership while keeping provenance verifiable — a blocker for institutional and RWA use cases.

Both reduce to the same ZK substrate. **Universal primitive in g2c + sector application in neftwerk.**

## The two pillars

**Pillar 1 — g2c: ZK account recovery as a universal primitive.** A user holds a BIP-39 seed offline at enrollment. From it we derive `owner_secret = HKDF(seed, account_id, network_passphrase)` and store `Poseidon2(owner_secret)` as a commitment. On recovery, the user generates a Noir/UltraHonk proof in the browser bound to a fully-specified `auth_hash = H(domain_sep ‖ account_id ‖ network_passphrase ‖ contract_addr ‖ new_signer_pubkey ‖ nonce)`. A new dedicated `g2c-recovery-controller` contract holds the pending-rotation state machine; during the timelock window the WebAuthn signer is **forbidden** from removing the ZK signer. The integration plugs into g2c's existing OZ `do_check_auth` pipeline; no protocol changes.

**Pillar 2 — neftwerk: ZK privacy as a sector validator.** Neftwerk's ARTS-1 token standard for art on Stellar uses three Noir circuits — `arts1_mint` (dual-ECDSA institutional + collector), `arts1_ownership` (private transfer authority), and `arts1_recovery` (the same primitive as Pillar 1 with token-id binding). Privacy model: "private title, public provenance."

## Why bundle

One Noir + UltraHonk + Poseidon2 + BN254 + Soroban 25.x substrate. One verifier crate. One team. The recovery circuit is a **key-rotation primitive any Soroban Smart Account can adopt**; neftwerk is the second consumer, not the only one. We estimate **~30–40% engineering shared** across the two tracks. **Track A and Track B are disbursed independently**: an A0 benchmark failure or Track B compliance gate does not affect Track A funding.

## Evidence we can ship today

- **Testnet E2E (`soroban-zk-demo`).** Browser generates Noir/UltraHonk proof; Soroban verifier contract validates on-chain. `./test_e2e.sh` runs build → deploy → prove → verify on Stellar testnet.
- **Verifier crate (`rs-soroban-ultrahonk`).** Standalone Soroban contract wrapping the UltraHonk verifier. VK ~1.9 KB, proof ~3.7 KB.
- **Nullifier-set + Merkle-frontier primitive.** Reference implementation with depth-20 tree, insertion proofs, and nullifier-set state — the substrate for revocation, replay protection, and future batched settlement.

## Budget and milestones

| Track | Scope | Tranches (10/20/30/40) | Total |
|---|---|---|---|
| **A0 (shared)** | Soroban gas/CPU benchmark gate; published numbers for `arts1_mint`-shaped reference circuit before any other circuit work | $30k flat | $30k |
| **Track A — g2c recovery** | A1 spec → A2 verifier + VK governance → A4 SmartAccount integration + recovery controller → A5 browser UX → A6 audit-ready freeze + mainnet | $150k tranched | $150k |
| **Track B — neftwerk privacy** | B0–B7, gated on A0 pass + Pillar 2 compliance gates (MSB/VASP/MiCA opinions, on-chain freeze for legal process, Howey opinion) | $200k tranched | $200k |
| **Total** | 14 weeks, ~3 FTE | | **$380k** |

Audit credits requested separately from SDF's standard pool: **circuit audit** (zkSecurity / Veridise / Spearbit ZK) and **contract audit** (OtterSec / Trail of Bits / Halborn) named separately, with a 4-week circuit freeze and 3-week contract freeze.

**We're requesting a 30-minute technical review with the SDF protocol team to walk through the appendix, A0 benchmark plan, and milestone gating.**

---

Repos: `g2c` · `zk/rs-soroban-ultrahonk` · `zk/soroban-zk-demo` · `zk/neftwerk` — see `SDF_PROPOSAL_TECHNICAL_APPENDIX.md` for full architecture, recovery state machine, VK governance design, threat model, and Pillar 2 compliance gating.
