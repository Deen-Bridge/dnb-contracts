# DeenBridge — On-Chain Giving (Soroban escrow)

Non-custodial USDC escrow on **Soroban** for **scholarships** and **community events**.
Donors fund a campaign; the contract holds the USDC and releases it to the organizer
**only** when the platform has approved the campaign **and** its funding goal is met.
If the campaign is cancelled, or its deadline passes without approval/goal, every donor
can reclaim **100%** of their contribution. The platform can approve or cancel — it can
**never** move escrowed funds to itself (no admin custody path).

This is Milestone 1 (Proof of Intent) of DeenBridge's Stellar Community Fund (SCF) Build
submission. It brings a reusable **conditional-giving escrow** primitive to Soroban.

## ✅ On-chain evidence (Stellar Testnet)

Deployed and exercised end-to-end on testnet — both the **release** and **refund** paths.

- **Contract:** [`CBP3UFPBCVGNQPTWIHHRH52VDD4PE64JGFPREF7GVFIIF7G4DZOWKX7Z`](https://stellar.expert/explorer/testnet/contract/CBP3UFPBCVGNQPTWIHHRH52VDD4PE64JGFPREF7GVFIIF7G4DZOWKX7Z)
- **Test USDC (SAC):** `CA6GNEL2RFHOBE7GPRNXZEHEZML3MBDJC6NB3O3VEFEOJDXCK7VAF76P`

**Release flow — campaign #1** (goal 250; two donations totalling 300; approved; released 300 to the organizer):

| Step | Transaction |
|---|---|
| deploy | [`5daed7d9…`](https://stellar.expert/explorer/testnet/tx/5daed7d93d730773f1acb1249033a832166c7294fc2f19193d451bebf1447af8) |
| initialize | [`51912516…`](https://stellar.expert/explorer/testnet/tx/519125164f8ab60c767b9b4e2052f5f362d25e8f4312668338ede43f3d7554e8) |
| create_campaign | [`60364538…`](https://stellar.expert/explorer/testnet/tx/60364538b726a7fd200bcbc84d9c78bf71068498a8c49889880d57396baf3976) |
| donate 100 | [`de9904ca…`](https://stellar.expert/explorer/testnet/tx/de9904caf4dfd876a6269bd6d03f064778839b8ab31ccf24e7a5b7c20657b6ad) |
| donate 200 | [`9663eba6…`](https://stellar.expert/explorer/testnet/tx/9663eba6db0993f0d7d5b99075dd4ecf792d83e752c34854f328e3ed3a90f8f4) |
| approve_campaign | [`97ab0431…`](https://stellar.expert/explorer/testnet/tx/97ab043103f34aa19bceb7be0a92df8500c9a3e1109d551efa4a6aac9a40646f) |
| release → organizer | [`84c083b4…`](https://stellar.expert/explorer/testnet/tx/84c083b483e6f71648f8d69a269392c208be4fbeb669a5ff340cee9aa8eb8f5f) |

Result: organizer balance `300`, contract balance `0`, campaign status `Released`.

**Refund flow — campaign #2** (goal 500; donated 100; cancelled; donor refunded 100):

| Step | Transaction |
|---|---|
| donate 100 | [`298e73b6…`](https://stellar.expert/explorer/testnet/tx/298e73b67ad683943f6429e1b3987b03fc08b0d0a6fd5250d20725530a1898dc) |
| cancel_campaign | [`9e7b3caf…`](https://stellar.expert/explorer/testnet/tx/9e7b3caff8a3ab01e2c7c1f7201de50e688558d038316b1f179aedaac64de254) |
| refund → donor | [`7049a39e…`](https://stellar.expert/explorer/testnet/tx/7049a39eeb42a1c313efb5152fbfa6b0a9cfa87b47c5cc012ebbac1e4f8e8de2) |

Result: donor made whole (`100`), contract balance `0`.

## Why Soroban (and not classic Stellar payments)

DeenBridge already runs on classic Stellar (SEP-1/7/10/29, path payments, multi-asset
USDC). But "hold these donations; release only if approved AND the goal is met by the
deadline; otherwise refund every donor" is **conditional logic** that classic payments
cannot express. That state machine needs a smart contract — this is the "why Soroban."

## Contract interface

| Function | Auth | Effect |
|---|---|---|
| `initialize(admin, usdc, platform)` | admin | One-time setup. `usdc` is the USDC Stellar Asset Contract (SEP-41); `platform` is the approver (ideally a multisig). |
| `create_campaign(kind, organizer, goal, deadline) -> id` | platform | Open a Scholarship/Event campaign. |
| `donate(campaign_id, donor, amount)` | donor | Pull USDC into escrow; record the contribution. |
| `approve_campaign(campaign_id)` | platform | Mark the campaign approved (legitimacy gate). |
| `release(campaign_id)` | platform | Pay the **organizer** the full raise. Requires approved + goal met. |
| `cancel_campaign(campaign_id)` | platform | Cancel, opening refunds. |
| `refund(campaign_id, donor)` | anyone | Return a donor's contribution (idempotent). Funds only ever go back to the donor. |
| `get_campaign / get_contribution / get_config` | — | Views. |

USDC is moved via its **Stellar Asset Contract** (the SEP-41 token interface) — no custom token.

## Lifecycle

```
create_campaign        -> Active
donate (Active, <deadline)   -> Active (raised += amount)
approve_campaign       -> Active, approved = true
release (approved && raised >= goal)  -> Released -> USDC to organizer
cancel_campaign        -> Cancelled -> refunds open
deadline passed && (under goal || not approved)  -> refunds open
```

All-or-nothing: no approval / goal miss ⇒ donors reclaim 100%.

## Security properties (asserted by tests)

- **No admin/platform custody path** — `release` pays the organizer, `refund` pays the
  donor; the platform balance stays 0 (`full_flow_donate_approve_release`).
- **Release is gated** on approval *and* goal (`release_fails_without_approval`,
  `release_fails_when_goal_not_met`).
- **Refunds are idempotent** — a cleared contribution can't be refunded twice
  (`refund_is_idempotent`).
- **Refunds are not open early** — an active, un-expired, un-cancelled campaign rejects
  refunds (`cannot_refund_active_before_deadline`).
- Donations rejected after the deadline; double-initialize rejected.

## Build & test

Requires the Rust toolchain and the wasm target (`rustup target add wasm32-unknown-unknown`).

```bash
cargo test                                        # 9 unit tests, all green
cargo build --target wasm32-unknown-unknown --release   # -> target/.../deenbridge_giving_escrow.wasm
```

> Note: the lockfile pins `ed25519-dalek = 2.2.0`; the just-released 3.0.0 is incompatible
> with Soroban's test host (`cargo update -p ed25519-dalek@3.0.0 --precise 2.2.0` if needed).

## Deploy to testnet

Install the Stellar CLI (`cargo install --locked stellar-cli` or `brew install stellar-cli`), then:

```bash
# 1. Identity + funding (testnet)
stellar keys generate --global deployer --network testnet --fund

# 2. Build + deploy the contract
stellar contract build
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/deenbridge_giving_escrow.wasm \
  --source deployer --network testnet
# -> prints CONTRACT_ID

# 3. Resolve the testnet USDC Stellar Asset Contract id (issuer from your stellar.toml)
stellar contract id asset --asset USDC:<USDC_ISSUER_PUBKEY> --network testnet

# 4. Initialize (admin, usdc SAC, platform approver)
stellar contract invoke --id CONTRACT_ID --source deployer --network testnet -- \
  initialize --admin <ADMIN_G...> --usdc <USDC_SAC_ID> --platform <PLATFORM_G...>

# 5. Example: create -> donate -> approve -> release
stellar contract invoke --id CONTRACT_ID --source platform --network testnet -- \
  create_campaign --kind Event --organizer <ORG_G...> --goal 1000000000 --deadline 1893456000
```

Record the deployed `CONTRACT_ID` and the create/donate/approve/release transaction
hashes — those are the on-chain evidence for the SCF Build submission.

## Roadmap (from the SCF milestone plan)

- **M1 (this repo):** escrow contract + tests, testnet deploy. ✅ contract + tests done.
- **M2:** dnb-backend integration — build unsigned invocation XDR (user wallet signs;
  stays non-custodial), index contract events into the API.
- **M3:** dnb-frontend flows (create/donate/approve/refund via Stellar Wallets Kit) + metrics.
- **M4:** mainnet deploy + security review.

Follow-ups tracked for production: persistent-storage TTL extension (`extend_ttl`),
partial/milestone releases for scholarships, and a platform multisig for approvals.

## License

MIT
