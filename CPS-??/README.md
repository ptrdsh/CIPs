---
CPS: "?"
Title: Regulated Stablecoins on Cardano
Category: Tokens
Status: Open
Authors:
    - Alex Moser <alexander.moser@cardanofoundation.org>
Proposed Solutions: []
Discussions:
    - Original PR: https://github.com/cardano-foundation/CIPs/pull/????
    - CIP-0113 | Programmable tokens: https://github.com/cardano-foundation/CIPs/pull/444
    - CPS-???? | Discoverability and machine-readable description of programmable token substandards (draft): https://github.com/Kammerlo/CIPs/blob/docs/cip-113-substandard-cps/CPS-%3F%3F%3F%3F/README.md
Created: 2026-08-20
License: CC-BY-4.0
---
 
## Abstract
 
[CIP-0113][CIP-0113] gives Cardano a framework for programmable tokens: a shared registry, a shared custody model, core delegate validators, and pluggable per-token rule sets called *substandards*. It deliberately does not say what any particular token's rules should be.
 
Regulated stablecoins — asset-referenced tokens (ARTs) and e-money tokens (EMTs) under Regulation (EU) 2023/1114 (MiCAR) — are the use case most often cited to justify programmable tokens, and the one for which the mapping from obligation to mechanism has never been written down. Ingredients exist in isolation: a `freeze-and-seize` reference substandard, a BaFin-oriented securities substandard with a role model, a KYC substandard built on verifiable credentials. None is a stablecoin standard, and none addresses what distinguishes a MiCAR stablecoin from any other restricted-transfer token: unconditional redemption at par, reserve-to-supply integrity, mandated disclosure, and supervisory reporting.
 
CIP-0113 is now at Last Check and its core implementation has completed a professional audit. Issuers are building against it today. Without a shared substandard, each will specify its own, and every wallet, explorer, DEX and custodian will need per-issuer integrations for instruments that are legally interchangeable in kind. This CPS states the problem, enumerates the requirements a solution must satisfy, and identifies the questions a solution must answer.
 
## Problem
 
### Where CIP-0113 stands
 
Two things date this document.
 
**The proposal text and its reference implementation have diverged.** PR #444 is at Last Check, but its specification text still describes a monolithic `programmableLogicGlobal` validator, a five-field `RegistryNode`, and a skip-count `input_idxs` encoding. The Cardano Foundation implementation — what issuers build against, and what was audited — has a seven-field node, three separate delegate validators, a different third-party redeemer, and a protocol-parameters UTxO absent from the specification entirely. **This CPS is written against the implementation.** Reconciling the two is a matter for PR #444, but a solution CIP cannot target both.
 
**Adoption is under way.** The core has been through audit and re-audit with fixes merged, though the final report is unpublished; the substandards have not been audited. Tokenization platforms are building on CIP-0113 with mainnet as a stated target, and the Cardano Foundation is developing a securities substandard. The window to agree a shared stablecoin substandard, rather than retrofit one across incompatible deployments, is closing.
 
### What the framework provides, and what it leaves open
 
Tokens live at a shared `programmable_logic_base` (PLB) address where ownership is carried by the *stake* credential. PLB is a dispatcher: each spend selects one of three core delegate validators — `transfer`, `third_party`, or `unfracking` — and the selected delegate then requires the token's own substandard logic. Registered policies sit in a sorted linked-list registry supporting O(1) membership and non-membership proofs.
 
Three transaction kinds matter here. **Transfer** is owner-authorised and runs the substandard's transfer logic. **Third-party** bypasses owner authorisation — the rail for seizure, forced transfer and administrative burn — and acts on one policy per transaction. **Unfracking** is holder-driven, same-owner restructuring that runs a separate unfracking hook rather than transfer logic.
 
The framework is explicit that these are primitives, not policy: freeze is unconditional, extraction is the gated power, and which holders are seizable is a substandard decision because a script stake credential cannot be distinguished from a DEX or lending pool.
 
### Four constraints a solution cannot engineer away
 
1. **One frozen credential holds three powers.** `minting_logic_script` authorises minting/burning, registration, and in-place node updates. It is frozen for the node's life, and if set to a `VerificationKey` rather than a `Script` credential, the node's configuration becomes permanently immutable.
2. **Registry updates are retroactive and un-noticed.** Re-pointing transfer or third-party logic re-governs every existing holder's balance on next spend — no notice, no timelock, no version pinning for integrators.
3. **The core validation logic is itself mutable, by an authority the issuer does not hold.** PLB resolves *live* delegate credentials from a shared protocol-parameters datum carrying an `upgrade_cred` that can replace any of them. The implementation further notes that pairwise distinctness of the three delegates is *an assumption, not an invariant*: nothing enforces it, and two equal credentials collapse the redeemer arms.
4. **No de-registration; seizure is per-UTxO; unfracking is a separate path.** Nodes cannot be removed. A fragmented balance may exceed transaction or execution budgets, defeating atomic seizure. Unfracking helps (holders can consolidate) and hurts (it restructures without running transfer logic), so it needs a deliberate policy rather than a permissive default.
### Why the existing substandards are not enough
 
| Substandard | Contributes | Falls short |
|---|---|---|
| `freeze-and-seize` (Cardano Foundation) | Linked-list denylist with covering-node proofs; freeze and seize; shared admin credential | A sanctions primitive, not an instrument: no supply control, no redemption path, no disclosure, no role separation. Unaudited |
| BaFin / "Finest" (FluidTokens) | The ecosystem's fullest role model — Owner, Admin, Minter, Burner, Pauser, Blacklister, Verifier, ForceTransfer — plus users list, global pause, mint ceiling, `security_info` | Built to eWpG securities rules: discloses ISIN and nominal amount, not reference currency or redemption terms. Its unqualified global pause is hazardous for an EMT (R6, R10) |
| KYC substandard (Cardano Foundation) | Allowlist half: off-chain verification bound to on-chain eligibility | Eligibility gating only; silent on issuer-side obligations |
 
BaFin is the closest structural precedent and worth mining for its role taxonomy. It is not a MiCAR standard, and adapting it by analogy rather than deriving from the regulation is the failure mode this CPS exists to prevent.
 
### The regulatory boundary
 
MiCAR covers ARTs (Title III, Arts. 16–47) and EMTs (Title IV, Arts. 48–58); EMT issuers must be credit institutions or EMIs. Most obligations are institutional and cannot be discharged on-chain — authorisation, own funds, reserve custody and investment, audit, ICT resilience, AML/CFT programmes. A substandard must not pretend otherwise.
 
A meaningful subset, however, is:
 
- **Enforceable** on-chain: supply ceilings, transfer eligibility, freeze, seizure, burn-on-redemption, and the availability of the redemption path.
- **Evidenced** on-chain: the binding of a policy ID to an authorised issuer and its white paper; supply against attestations; a tamper-evident record of administrative actions.
- **Defeated** on-chain by a careless design: a pause that blocks redemption; one key holding mint, freeze and reconfiguration authority; a denylist that is irreversibly public personal data; an unfracking hook that lets a designated holder shard beyond the reach of seizure; an enforcement rail whose credential changes underneath the issuer.
That third category is why this is a problem statement rather than a nice-to-have. An *incomplete* substandard leaves an issuer with off-chain work. A *wrong* one can put an authorised issuer in breach.
 
### Relationship to other problem statements
 
[CPS-0003][CPS-0003] asked whether Cardano should have programmable tokens at all; CIP-0113 answers it. A separate draft, **[Discoverability and machine-readable description of programmable token substandards][CPS-SUBSTD]**, sits between the two: how does generic tooling learn which substandard a policy uses and how to build a valid transaction for it. That CPS is horizontal and explicitly does not prescribe what any substandard should enforce. This one is the vertical case. They are complementary but not the same problem — a solution to that one is a description format; a solution to this one is a substandard.
 
**This CPS depends on it for part of Goal 4.** R1, R2 and R11 all need a carrier binding a machine-readable claim to the scripts in force, resolvable without issuer-specific code. Substandard identity, versioning under rotating hashes, description-to-script binding and capability declaration are that CPS's problem. This document states only what a MiCAR stablecoin's declaration must *contain*.
 
Two seams are worth flagging to both authors: that draft is written against the stale PR #444 text, which affects its question on adding a `RegistryNode` field (the node has already gained two fields since); and it treats a CIP-0113 version as its bootstrap transaction hash, which no longer pins validation logic now that delegates are mutable — the same dependency raised here as R24 and R25.
 
## Use Cases
 
**An authorised EMI issues a EUR EMT.** It must mint against received funds, cap supply to the reserve, publish the binding between policy ID and white paper, freeze sanctioned holders within hours, and burn on redemption — while never blocking a non-sanctioned holder from redeeming. Today it specifies and audits all of this itself, and no wallet supports it without bespoke work.
 
**A compliance officer executes a restrictive measure.** On an EU designation they must freeze immediately and, on instruction, seize — under an authority separate from the minting key, evidenced with a machine-readable reason, and reversible if the designation lifts. `freeze-and-seize` supplies the mechanism and a shared admin credential; nothing supplies the separation or the record.
 
**A holder redeems while partially restricted.** A holder subject to a transfer limit, an operational pause or an eligibility lapse still holds an unconditional Art. 49 claim. The substandard must distinguish restrictions that lawfully suspend redemption from those that must not, legibly rather than as an emergent property of validator code.
 
**A wallet decides what to show.** Given a policy ID: is this MiCAR-authorised; ART or EMT; what currency and decimals; who is the issuer; where is the white paper; is this balance frozen and why; who can seize it. Today it can answer none of these generically.
 
**A DEX or lending protocol decides whether to list.** It must know what the issuer can do to tokens in its pools. The framework tells protocols to read the third-party script's source. There is no declared capability set to check.
 
**An issuer monitors its own dependencies.** Enforcement runs through core delegate credentials a protocol-level authority can rewrite, and through registry nodes whose spending by unrelated parties invalidates in-flight transactions. An issuer under Art. 34 obligations must detect both, and has no standard way to.
 
**An auditor or NCA reconciles.** Given a policy ID and a period: circulating supply, mint and burn history, administrative actions, and the Art. 22 / 58 transaction metrics. The chain holds the raw material; nothing tells an indexer how to read it consistently across issuers.
 
In every case the current alternative is to read the substandard's source and hand-write an integration — which does not scale, breaks silently on upgrades, and cannot be verified against the chain by the tool relying on it.
 
## Goals
 
Ranked by importance.
 
1. **Establish that a MiCAR stablecoin substandard should be a shared ecosystem standard, not a per-issuer artefact** — while that is still achievable rather than retrofittable.
2. **Produce a requirement set traceable to regulation,** so a reader can audit the mapping rather than trust it.
3. **Ensure holder rights survive issuer powers.** No configuration may render Art. 39 / 49 redemption unavailable except where a restrictive measure lawfully requires it.
4. **Make compliance state legible to integrators** from chain data plus a documented schema, with no issuer-specific knowledge. Partly discharged by [CPS-????][CPS-SUBSTD]: this CPS fixes *what* a stablecoin must make legible, that one fixes *how*.
5. **Separate powers MiCAR expects to be separated,** notwithstanding the frozen `minting_logic_script`.
6. **Make the dependency on shared, mutable infrastructure explicit and monitorable,** so an issuer can tell its competent authority what it controls and what it does not.
7. **Support supervisory observability** — reserve-to-supply reconciliation and Art. 22 / 23 / 58 metrics derivable by an independent party.
8. **Extend cleanly to adjacent regimes** by factoring a reusable regulated-asset core from the MiCAR-specific layer, so eWpG, CMTA and the US GENIUS Act can reuse the former.
9. **Do not fork the ecosystem.** Reuse existing denylist and eligibility mechanics rather than reimplementing them.
**Non-goals.** Redesigning CIP-0113 core, though a solution may identify core changes it needs (Q5). Resolving the specification-implementation divergence in PR #444. Legal advice, or any substitute for authorisation. Algorithmic stabilisation, which MiCAR's reserve rules effectively exclude. Moving off-chain obligations on-chain. Personal data on-chain. Specifying a general substandard description format — that is [CPS-????][CPS-SUBSTD].
 
### Requirement matrix
 
What a solution CIP must satisfy, and where the ecosystem stands today. **Absent** — nothing addresses it. **Partial** — an existing substandard addresses part of it. **Core** — constrained by CIP-0113 itself. **Hazard** — an existing design actively gets this wrong.
 
**A. Instrument identity and disclosure**
 
| | Requirement | MiCAR anchor | Status |
|---|---|---|---|
| R1 | Policy ID binds to a machine-readable instrument record: class, reference currency or basket, decimals, issuer legal name and LEI, home NCA and authorisation reference | Arts. 19, 51; DR (EU) 2025/305 | Absent |
| R2 | Cryptographic binding to the published white paper (hash plus resolvable URI), amendable with prior versions still resolvable | Arts. 19, 51 | Absent |
 
**B. Supply and reserve integrity**
 
| | Requirement | MiCAR anchor | Status |
|---|---|---|---|
| R3 | Enforced supply ceiling, and no issuance while the instrument is in a state where issuance must halt — impossible, not merely prohibited | Arts. 23, 36, 54, 58 | Partial |
| R4 | Circulating supply independently computable from chain data and comparable against published attestations | Arts. 22, 36, 54 | Partial |
| R5 | On-chain pointer to the current reserve attestation: hash, URI, period, attestor | Arts. 36, 54; EBA RTS | Absent |
 
**C. Holder rights**
 
| | Requirement | MiCAR anchor | Status |
|---|---|---|---|
| R6 | A redemption path — burn against off-chain payout — always available to a holder not under a lawful restrictive measure | Arts. 39, 49 | **Hazard** |
| R7 | Restriction taxonomy separating states that suspend redemption (sanctions, court order) from those that must not (operational pause, eligibility lapse, limits) | Arts. 39, 49 | Absent |
| R8 | No interest or time-dependent benefit anywhere in the design | Art. 50 and the ART equivalent | Absent |
 
**D. Restrictive measures**
 
| | Requirement | MiCAR anchor | Status |
|---|---|---|---|
| R9 | Freeze — whole balance or partial amount — effective on next spend, unconditional and immediate | Restrictive measures; AMLR | Partial |
| R10 | Global pause explicitly scoped, and unable to block R6 | Art. 46 | **Hazard** |
| R11 | Seizure and forced transfer under a stated, counterparty-verifiable policy on script-staked holders (protocol allowlist, or same-transaction consent) | Court orders; Art. 47 | Partial |
| R12 | Every restrictive action carries a machine-readable reason code, legal basis and optional expiry; reversal is equally evidenced | Art. 34; DORA | Absent |
| R13 | A deliberate unfracking policy: the hook must not let a designated holder shard a balance beyond atomic seizure, while still permitting ordinary consolidation | Effectiveness of measures | Absent |
 
**E. Eligibility and personal data**
 
| | Requirement | MiCAR anchor | Status |
|---|---|---|---|
| R14 | Both eligibility modes supported and selectable per instrument: denylist (the bearer-like EMT default) and allowlist (institutional) | AMLR | Partial |
| R15 | Neither mode places personal data on-chain in an irreversible structure | GDPR | Absent |
| R16 | Travel-rule data referenced, never embedded | Reg. (EU) 2023/1113 | Absent |
 
**F. Governance and roles**
 
| | Requirement | MiCAR anchor | Status |
|---|---|---|---|
| R17 | Distinct authorities for mint/burn, freeze, seizure, eligibility administration and reconfiguration | Art. 34 | **Core** |
| R18 | Threshold or multi-party authorisation for high-impact actions, with key rotation that does not require token migration | Art. 34 | **Core** |
| R19 | Reconfiguration of transfer or third-party logic is announced and delayed, not silently retroactive | Holder protection | **Core** |
 
**G. Observability**
 
| | Requirement | MiCAR anchor | Status |
|---|---|---|---|
| R20 | A documented convention letting an independent indexer compute the Art. 22 / 23 / 58 means-of-exchange metrics | Arts. 22, 23, 58; EBA RTS under Art. 22(6) | Absent |
| R21 | Ordered, non-repudiable history of issuance and administrative actions reconstructable from chain data | Art. 34 | Partial |
 
**H. Lifecycle**
 
| | Requirement | MiCAR anchor | Status |
|---|---|---|---|
| R22 | An executable redemption-plan path — mass burn against payout, with a defined treatment of unreachable holders — and a migration path between substandard versions | Art. 47 | Absent |
| R23 | A defined terminal state for a wound-down policy, whose registry node persists forever | Art. 47 | **Core** |
 
**I. Dependency on shared infrastructure**
 
| | Requirement | MiCAR anchor | Status |
|---|---|---|---|
| R24 | The dependency on the shared protocol-parameters datum is disclosed, and any change to the live delegate credentials — including loss of their pairwise distinctness — is detectable by issuer and integrators | Art. 34; DORA | Absent |
| R25 | A stated position on whether the instrument remains compliant if the upgrade authority replaces a delegate, and what the issuer does if not; likewise for registry-node contention as a griefing vector | Arts. 34, 46; DORA | Absent |
 
## Open Questions
 
1. **One substandard or a profile family — and what is reusable?** ARTs and EMTs differ in reserve model, redemption, disclosure and reporting. Is the answer one configurable substandard or an EMT profile and an ART profile? Which requirements belong to a regulated-asset core that eWpG, CMTA and GENIUS Act deployments share (Goal 8)? How does an integrator discover which profile a policy implements — noting the discovery mechanism itself belongs to [CPS-????][CPS-SUBSTD]?
2. **How is redemption made unblockable?** Transfer logic governs every owner-initiated spend, so what construction guarantees a holder can always reach the burn-to-redeem path when not lawfully restricted — and how does a validator distinguish a genuine redemption from a transfer disguised as one? What is the precedence rule between a restrictive measure and an Art. 49 claim, and where is it encoded (R6, R7)?
3. **GDPR versus an append-only denylist.** Credential hashes are personal data in a structure with no de-registration path. Can entries be removed rather than marked inactive? Does off-chain state with an on-chain commitment satisfy enforcement? Is there a construction that is both erasable and non-membership-provable (R15)?
4. **Who can change the rules under a holder, and with what notice?** Two levels: registry-node updates are immediate and retroactive over all holders; the protocol-level `upgrade_cred` can replace the delegate validators for every programmable token at once. Should a stablecoin substandard timelock its own reconfiguration — and does that undermine responding to a designation within hours (R19 versus R9)? Who holds `upgrade_cred` on mainnet, under what controls? Can a supervised issuer's enforcement lawfully depend on it, and is an opt-in core deployment a coherent alternative or does it destroy the interoperability that motivates CIP-0113 (R24, R25)?
5. **Does this require CIP-0113 core changes, and which text does a solution reference?** Candidates: the frozen `minting_logic_script` conflating three authorities (R17, R18); one policy per third-party transaction, forcing sequential transactions when a designated holder holds several regulated instruments; no de-registration or terminal node state (R23); on-chain enforcement of delegate distinctness (R24). Are these amendments to PR #444 before merge or a follow-up CIP? And until the specification and implementation are reconciled, which does a substandard normatively target?
6. **What is the unfracking policy for a regulated stablecoin?** The hook runs instead of transfer logic, so denylist checks do not apply by default. Should a frozen or designated holder be able to unfrack at all? Should the hook bound UTxO count or minimum denomination per holder to keep seizure feasible, and at what cost to ordinary holders' fee efficiency (R13)?
7. **How is the means-of-exchange metric actually computed?** The EBA methodology excludes personal-wallet-to-personal-wallet transfers and counts only same-currency-area pairs; neither is determinable from chain data alone. What is the minimum viable annotation convention, who supplies it, and what happens when it is absent or false (R20)?
8. **Contention and cost at payment volumes.** Registry-node contention is an accepted limitation mitigated off-chain by rebuild-and-retry; a denylist linked list has the same shape, and every transfer carries covering-node reference inputs plus withdrawals. Measured reference-script footprints are roughly 3.0 kB per transfer and 2.7 kB per seize. Is that acceptable for a payment instrument, and can a griefing actor materially disrupt one by targeting its nodes (R25)?
9. **What is the DeFi contract?** The core offers allowlist and consent patterns for extraction from script-staked UTxOs but enforces neither. Should a stablecoin substandard mandate one, and how does a protocol verify the issuer's policy before integrating (R11)?
10. **Multi-chain supply reconciliation.** The same authorised EMT will exist on several ledgers. Is Cardano-side circulating supply meaningful in isolation, and must the substandard express its share of a global cap (R3, R4)?
## References
 
**Cardano**
 
- [CIP-0113][CIP-0113] — Programmable tokens (at Last Check)
- [CPS-0003][CPS-0003] — Smart Tokens
- [CPS-????][CPS-SUBSTD] — Discoverability and machine-readable description of programmable token substandards, Thomas Kammerlocher (draft)
- [CIP-0057][CIP-0057] — Plutus Contract Blueprint
- Core implementation: https://github.com/cardano-foundation/cip113-programmable-tokens — see `documentation/02-ARCHITECTURE.md`, `03-CONTROL-SCOPE-AND-ADMIN-AUTHORITY.md`, `08-INTEGRATION-GUIDES.md`, `09-DEVELOPING-SUBSTANDARDS.md`
- Platform and reference substandards: https://github.com/cardano-foundation/cip113-programmable-tokens-platform
- BaFin securities substandard: https://github.com/FluidTokens/fn-bafin-cardano-sc
**Regulation**
 
- Regulation (EU) 2023/1114 (MiCAR) — Title III (ARTs, Arts. 16–47) and Title IV (EMTs, Arts. 48–58); in particular Arts. 19 and 51 (white paper), 22 (reporting), 23 and 58 (means-of-exchange thresholds), 34 (governance), 36–38 and 54 (reserve), 39 and 49 (redemption), 46–47 (recovery and redemption plans), 43 and 56 (significance)
- Commission Delegated Regulation (EU) 2025/305 — white paper templates
- Commission Delegated Regulation (EU) 2024/2730 — highly liquid financial instruments
- EBA final draft RTS under Art. 22(6) — means-of-exchange estimation methodology
- Regulation (EU) 2023/1113 (travel rule); (EU) 2024/1624 (AMLR); (EU) 2022/2554 (DORA); (EU) 2016/679 (GDPR)
*Regulatory citations identify the obligations a technical solution must serve. They are not legal advice; any issuer must rely on its own counsel and its competent authority.*
 
## Acknowledgements
 
This problem statement builds on the CIP-0113 authors' work, on the Cardano Foundation's reference substandards and architecture documentation, and on the BaFin substandard developed by Matteo Coppola as part of the Finest team, which established the role model treated here as the closest existing precedent.
 
[CIP-0113]: https://github.com/cardano-foundation/CIPs/pull/444
[CIP-0057]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0057
[CPS-0003]: https://github.com/cardano-foundation/CIPs/tree/master/CPS-0003
[CPS-SUBSTD]: https://github.com/Kammerlo/CIPs/blob/docs/cip-113-substandard-cps/CPS-%3F%3F%3F%3F/README.md
 
## Copyright
 
This CPS is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
