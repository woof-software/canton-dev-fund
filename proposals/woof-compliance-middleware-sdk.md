# Compliance Middleware SDK for EVM Developers on Zenith

| Field | Value |
| :---- | :---- |
| **Organization** | Woof |
| **Author / Primary Contact** | Mykola Ilchuk, Woof ([@Noosphere-314](https://github.com/Noosphere-314)) |
| **Status** | Submitted |
| **Created** | 2026-08-06 |
| **Proposal Type** | RFP-aligned |
| **RFP / Roadmap Area** | RFP 12 (RWA Standards), item 12.1: Identity, Credentials and KYC Standards for RWA Workflows |
| **Champion** | `Needs Champion` |
| **Total Funding Request** | $120,000 USD, paid in Canton Coin at each milestone's acceptance rate |
| **Project Duration** | 6 months |
| **Label** | `regulatory-compliance` |

---

# Abstract

This proposal funds an open-source **Compliance Middleware SDK** that lets Solidity contracts deployed on Zenith EVM act on Canton-side compliance decisions: the check runs on Canton, and the contract verifies an authorization issued on the basis of that Canton state. The SDK delivers a Solidity contract layer (base contracts and interfaces), a TypeScript middleware layer, and a set of production-shaped reference dApps demonstrating common compliant DeFi patterns (e.g. permissioned lending, compliant vaults, KYC-gated trading).

The SDK consumes a `ComplianceGuard`-shaped DAML interface as its asset-level compliance source. The reference implementation of that interface is [TokenProof](https://github.com/Compliledger/canton_tokenproof), published by Compliledger under [PR #231](https://github.com/canton-foundation/canton-dev-fund/pull/231); that proposal was withdrawn by its author on 8 September 2026 and resubmitted as [#771](https://github.com/canton-foundation/canton-dev-fund/pull/771) with a different scope, so the SDK treats `ComplianceGuard` as a public interface it can implement against rather than as a funded dependency, and ships its own minimal implementation (see Dependency isolation). Without an EVM-facing surface, the EVM-background developers who made up 71% of respondents to the Foundation's 2026 Developer Experience survey cannot act on Canton-side compliance facts from their dApps without building the authorization path themselves — each team repeating compliance integration work in the highest-stakes part of the stack.

**Total funding:** $120,000 USD, denominated in Canton Coin at the prevailing USD/CC rate at each milestone's acceptance, across 2 milestones. The volatility-adjustment mechanism required by CIP-0100 is set out in the Funding section.
**Duration:** 6 months across 2 milestones.

---

# Motivation

Canton processes $9T+ monthly volume across institutional rails, and Zenith EVM opens the ecosystem to EVM-background developers, 71% of respondents to the Foundation's 2026 Developer Experience survey (41 respondents). 83% of the projects in that survey are TradFi or hybrid — domains where regulatory compliance is non-negotiable.

Canton's CIP-56 token standard, party-based privacy, and atomic Delivery-vs-Payment (DvP) settlement provide institutional-grade compliance primitives — but only on the DAML side. EVM developers deploying through Zenith face a structural gap:

- **Compliance facts are not reachable from Solidity.** Canton holds them: asset-level proofs such as TokenProof's `ComplianceProof` behind the DAML `ComplianceGuard` interface, and holder-level records such as an issuer's allowlist decision or an operator's own compliance record, with credential standards following under RFP 12.1. A Solidity contract on Zenith has no way to consult them, because Zenith's cross-VM primitive, `external_call()`, is invoked from a Daml workflow rather than from an EVM contract. The check therefore has to happen on the Canton side and reach the EVM contract as an authorization, and no active PR provides that path to a dApp developer.
- **Compliance-gated settlement.** Zenith's Gateway now demonstrates atomic settlement between a CIP-56 asset and an ERC-20, coordinated from a Canton-native workflow (September 2026). What no active PR provides is the compliance precondition on that path from the EVM developer's side. [LynoBridge (#147)](https://github.com/canton-foundation/canton-dev-fund/pull/147) bridges EVM tokens to Canton but does not enforce compliance during settlement.
- **Reference DeFi implementations.** [OpenZeppelin Canton Stack (#262)](https://github.com/canton-foundation/canton-dev-fund/pull/262) ships Daml libraries. EVM-side compliance-integrated DeFi primitives (vaults, lending, AMM) are absent from every active PR.
- **Party ↔ address mapping for EVM.** Solidity contracts speak `address`. Canton speaks `Party`. Registration flows exist in pieces ([#453](https://github.com/canton-foundation/canton-dev-fund/pull/453) registers EVM addresses to external parties; Zenith's Gateway represents an EVM user as a Canton party), but no reusable helper layer exposes them to a dApp developer.

**A note on native registry controls.** Canton's token layer is not compliance-blind. Digital Asset's Registry App gives registrars both a party **blocklist** (the `checkBlocklist` field on `InstrumentConfiguration`, since Registry App v0.13 — blocked parties cannot transfer, mint, or burn) and a credential-based **allowlist** (per-instrument rules restricting holding, transfers, and mint/redeem to credentialed parties). These are **asset-model controls** — the issuer deciding who may hold or move *their instrument*. DA's own docs draw the line precisely: "[r]ather than application-layer logic, the DA Registry embeds authorization directly into the asset model" ([blocklist](https://docs.digitalasset.com/registry/guides/blocklist), [allowlist](https://docs.digitalasset.com/registry/features/allowlist)).

This SDK operates on the other side of that line — the **application layer**: giving a dApp developer on Zenith a Solidity-native way to enforce *their own* per-action permissions inside their contract ("may this address open a position in my lending market", "may it swap in my KYC-gated pool"), across whatever assets the dApp handles — including app-level objects (positions, LP shares, app permissions) that are not registry instruments at all. The layers compose: a party barred at the registry cannot move the instrument regardless of what any dApp does, while nothing at the instrument level expresses a dApp's own business-logic permissions — and no native control surface of either kind currently exists for Solidity on Zenith. The SDK deliberately reimplements neither blocklisting nor instrument-level allowlisting. Its compliance interface is backend-agnostic, so registry-level state can serve as one signal source where a registry exposes it — but nothing in the SDK's design depends on that: the layers compose by default, because instrument-level controls enforce themselves at the token layer no matter what any dApp does.

A large share of dApps deploying through Zenith will need **application-level compliance gating beyond instrument-level registry controls** — permissions expressed in their own contract logic rather than at the token layer — and today every team rebuilds it from scratch. This SDK eliminates that duplication and reduces ecosystem-wide compliance-implementation risk.

**This is a distinct layer from other EVM↔CIP-56 work in the queue.** Several proposals touch the broad "EVM meets CIP-56" space, but at different layers — none deliver compliance enforcement embedded in Solidity contracts:

- [CIP-56/ERC-20 Middleware (#453)](https://github.com/canton-foundation/canton-dev-fund/pull/453) is a **transport/wallet layer** — a MetaMask-compatible Ethereum JSON-RPC facade over CIP-56 tokens, plus an indexer and a bridge relayer. It lets EVM wallets and tooling *reach* CIP-56 tokens; it does not give a Solidity developer compliance modifiers (`onlyKYC`), atomic ERC-20↔CIP-56 DvP, or party-based gating inside their own contract. The two are complementary: #453 is how an EVM wallet connects, this SDK is how an EVM *contract* enforces compliance.
- [BlockTravel (#190)](https://github.com/canton-foundation/canton-dev-fund/pull/190) is an **off-ledger decisioning service** (pre-transaction Travel-Rule/AML checks). It is a compliance data source, not a Solidity contract layer; it could sit behind our `ICantonKYC` interface as one possible backend.
- [TokenProof](https://github.com/Compliledger/canton_tokenproof) provides the **DAML-side `ComplianceGuard` interface** this SDK implements against (see below). Its funding proposal #231 was withdrawn on 8 September 2026; the published interface and PoC remain available under their own licence.

Our layer — security-reviewed Solidity base contracts that a Zenith dApp inherits to enforce Canton-side compliance decisions, plus the Canton-side templates that produce those decisions — is unaddressed by any of them.

---

# Specification

## 1. Objective

Deliver a single, security-reviewed, MIT-licensed SDK that lets a Solidity developer with no prior Canton experience add Canton-backed compliance gating and party-based permissions to their Zenith dApp, and compose with Zenith's Gateway settlement path where it is available, in under one hour following the quickstart guide.

The scope explicitly does **not** include building a compliance-decision engine or an evidence pipeline (that is the domain of dedicated providers such as Compliledger, whose current proposal is [#771](https://github.com/canton-foundation/canton-dev-fund/pull/771)), full Canton DevNet orchestration (that is [Denex LocalNet's domain](https://github.com/canton-foundation/canton-dev-fund/pull/318)), or institutional API gateway / enterprise IAM bridging (out of Woof's core competency).

## 2. Implementation Mechanics

A two-layer SDK published under `@woof-software/canton-compliance` (TypeScript, npm) and `woof-software/canton-compliance-contracts` (Solidity).

### Layer 1 — Solidity contract layer on Zenith EVM

**Architecture note (revised September 2026).** An earlier draft of this section described Solidity contracts resolving compliance state through `external_call()`. That is not how Zenith composes the two sides: `external_call()` is invoked from a Daml workflow, and EVM execution is coordinated and verified from the Canton side (CIP-0091; Zenith's documentation; Zenith's atomic-swap write-up of 1 September 2026). This revision corrects the direction. The compliance check happens on Canton, and what reaches the EVM contract is a bound authorization. The exact authorization path for Milestone 2 is being aligned with Zenith and a design partner before it is fixed; both candidates are set out below, and the contract-level surface is intended to be the same for either.

A set of inherit-and-extend base contracts and interfaces through which a Zenith dApp accepts Canton-authorized actions. A developer inherits a base contract and annotates a function with a modifier such as `onlyCantonAuthorized("DEPOSIT")`; the SDK verifies that a valid authorization bound to that account, action and parameters accompanies the call, and reverts otherwise.

Two authorization paths, one contract surface:

- **Operator-signed permit (available today).** An authorizing service run by the dApp operator, holding a Canton party with read access to the relevant compliance fact (as the asset issuer, through TokenProof's disclosure endpoint, or as a credential verifier), performs a fresh Ledger API read of the fact at issuance time (an active-contract query or a disclosed-contract fetch, never a cache) and issues a short-lived EIP-712 permit bound to chain, contract, account, call and parameters, a single-use nonce, and a validity window that starts at the time of that read. The service refuses to issue when the read fails, when the ledger source is unreachable, or when the fact is older than a configured maximum age. The contract verifies signature, window and nonce. What the EVM side can establish is only that a signer the dApp has configured as trusted vouched for the action; the freshness of the underlying Canton state is a commitment of that signer, bounded by the window and the maximum age, and revocation inside the window is bounded, not eliminated. This path needs neither the Gateway nor `external_call()` and is testable on the public Zenith EVM testnet now.
- **Gateway-coordinated authorization (as Zenith opens the path to third-party workflows).** Design goal: the same contract surface accepts an authorization produced inside a Canton workflow that runs the compliance check as a transaction precondition (for asset-level facts, TokenProof's `ComplianceGuard.checkCompliance`) and coordinates the EVM leg through Zenith's Gateway, so that the check and the EVM effect commit or fail together. This is the pattern Zenith demonstrated with bound vouchers in September 2026. Availability on Zenith's dev or test environments depends on Zenith exposing the Gateway path to third-party Daml workflows; availability on MainNet additionally depends on `external_call()` shipping with Canton 3.6. The SDK aims to treat it as an adapter behind the same interface; whether that holds without changes to integrators' contracts is to be confirmed with Zenith.

Two issuer control models, and only one of them is what an authorization covers. Issuers on Canton gate different things: some gate minting and redemption and leave transfers free; others need holder eligibility enforced on every transfer. Gating mint and redemption is a per-action decision, and both paths above fit it. Eligibility on transfers is not per-action: a permit per transfer is impractical, and what the EVM side needs is an eligibility record kept in step with Canton, with a published revocation latency. That is a third mechanism, neither a permit nor a Gateway flow; it is not a Milestone 1 deliverable and is scoped into the Milestone 2 reference dApps only once a concrete issuer workflow names it.

The contract layer covers the following functional areas (delivered across however many contracts/interfaces prove cleanest in implementation):

- **Canton-authorized entry points** — reusable base contract(s) exposing modifiers/guards (KYC-gated, permission-gated, party-gated) that verify a bound authorization: signer set, binding to account, action and parameters, expiry, single use.
- **Authorization producers** — the Canton-side counterpart: DAML templates and a reference authorizing service that evaluate a compliance fact and produce the authorization, for both paths above. Asset-level facts come from TokenProof's `ComplianceProof`: the adapter checks the proof's binding to the expected asset, issuer, evaluator and policy version, not only that its status is Active. Holder-level facts in Milestone 1 come from a source the operator already controls: the operator's own DAML compliance-decision template (the pattern exercised in our `canton-localnet-poc`) or the issuer's credential-based allowlist decision in the Registry App; the credential and party-metadata standards under RFP 12.1 are adopted as they land, not assumed. Reading a fact, using it inside a DAML operation, and being a signer a given dApp trusts are three separate rights: access to TokenProof proofs is granted by disclosure from the issuer or evaluator, and each dApp's trusted signer set is configured explicitly by its operator.
- **Compliance-gated settlement** — reference wiring that places a compliance precondition on Zenith's Gateway settlement path. The SDK does not implement cross-VM settlement itself; Zenith's Gateway does.
- **CIP-56 asset interaction** — off-chain: TypeScript helpers over the JSON-RPC facade in [#453](https://github.com/canton-foundation/canton-dev-fund/pull/453) for reading and moving CIP-56 assets from the dApp's backend, not duplicating it. On-chain: a Solidity contract can only hold or move an EVM representation of a Canton asset, so contract-level CIP-56 helpers are conditional on such a representation being available to third parties (Zenith's Gateway and registry layer), as for the reference dApps below.
- **Party ↔ address resolution** — helpers over existing registration flows (EIP-191 registration to an external party as in #453; the Gateway's party representation), not a new registry.

The intent is unchanged: a familiar inherit-and-compose surface for EVM developers. The breakdown into individual contracts and interfaces will be driven by what produces the cleanest, most auditable API during M1.

### Layer 2 — TypeScript middleware

A TypeScript package that gives application and frontend developers typed, high-level access to the compliance layer without hand-writing cross-VM plumbing. It covers the following functional areas (the exact module/class breakdown is determined during implementation):

- **Typed authorization flow** — typed client utilities that request an authorization for an action (from the operator's authorizing service today, from the Gateway-coordinated workflow as it becomes available) and submit the EVM call with it, so the two-step flow is one typed call for the developer.
- **Party ↔ address resolution** — helpers over existing registration flows (EIP-191 registration to an external party as in #453; the Gateway's party representation), with caching. Not a new registry.
- **Compliance-gated settlement assembly** — where the Gateway path is available, helpers for composing a Gateway-coordinated settlement with a compliance precondition; on the permit path, helpers for the authorization-then-call sequence with explicit handling of expiry and revocation between the two steps.
- **Frontend integration** — a set of UI building blocks (e.g. React components/hooks) surfacing authorization status, permissions, and settlement history for a connected wallet, read through the operator's authorized backend rather than from a public on-chain mirror.

The goal is a `wagmi`/`viem`-style developer experience: a thin, well-typed layer over the contracts. Final package structure (number of modules, components, and helpers) will follow what is cleanest in implementation.

### Reference implementations

A set of open-source reference dApps (Apache-2.0), each with Solidity + DAML + deployment scripts + a test suite, demonstrating how the SDK is used for the most common compliant DeFi patterns. The planned patterns — finalized during M2 based on community demand — include:

- **Permissioned lending** — Compound-shaped: Canton-authorized deposits and borrows, compliant liquidations. The Milestone 1 build uses EVM-native test ERC-20 assets. A CIP-56 collateral variant depends on an EVM representation of the Canton asset (Zenith's Gateway and registry layer) and is scoped only once that mechanism is available to third parties.
- **Compliant vault** — ERC-4626 with Canton-authorized entry and exit. ERC-4626 requires an ERC-20 underlying, so the Milestone 1 build uses an EVM-native test ERC-20; a variant over an EVM representation of a CIP-56 asset is conditional on the same mechanism as above.
- **KYC-gated trading** — an AMM-style pool that enforces party permissions on swaps.

These are illustrative of the patterns we expect to be most valuable; the final set and count of reference dApps will be confirmed with the community during M2. The permit-path builds of all three are independent of Zenith-specific features. Moving a Canton asset into or out of an EVM contract is not something an authorization provides, and it is not claimed here.

### Technologies and operational approach

- Solidity contract layer deployed on Zenith, published as an open-source package.
- TypeScript middleware published to npm.
- Hardhat as the primary test/dev environment; complementary tooling (e.g. Foundry) for invariant/fuzz coverage where useful.
- Static analysis (Slither) as part of the security baseline, targeting zero high-severity findings.
- Any DAML packages used pass [Daml Package Analyzer (PR #130, Certora)](https://github.com/canton-foundation/canton-dev-fund/pull/130) where available — leveraging a funded ecosystem tool as an audit baseline.

### Proof of concept (already built)

The core pattern is already validated in public PoCs — this is not greenfield:

- [woof-software/canton-compliance-poc](https://github.com/woof-software/canton-compliance-poc) — the EVM-side pattern: a base contract with an `onlyKYC` modifier resolving compliance through a pluggable interface, with a mock standing in for the Canton side. Passing CI.
- [woof-software/canton-localnet-poc](https://github.com/woof-software/canton-localnet-poc) — the DAML-side counterpart, built with SDK 3.4.11 and executed on a live Canton ledger: the script creates and reads back an on-ledger compliance decision. Run log and DAR are committed as evidence.

The SDK productionizes these PoCs: the pluggable interface behind `onlyKYC` becomes the authorization verifier described in Layer 1, and the DAML side becomes the authorization producer. The two PoCs validate the two sides separately; the join between them is what Milestone 1 delivers on the permit path, and what the Gateway path adds as Zenith opens it.

### Dependency isolation: the authorization path

We name the proposal's primary platform dependency explicitly. Zenith's cross-VM primitive, `external_call()`, is invoked from Daml workflows, is delivered in Canton's dev protocol version (canton#513, merged July 2026) and is planned for MainNet with Canton 3.6. Zenith's Gateway, which coordinates EVM execution from Canton, is currently Zenith's in-house component without a published path for third-party workflows. Milestone 1 depends on neither: the operator-signed permit path works against the public Zenith EVM testnet and any Canton environment today. The Gateway-coordinated path is designed as an adapter behind the same contract interface, added as Zenith opens it; that it can be added without changes to integrators' contracts is a design goal to be confirmed with Zenith, not a guarantee. Availability of `external_call()` on Zenith's dev or test environments and on Canton MainNet (3.6) are separate questions. We are aligning the exact path with Zenith before fixing the Milestone 2 acceptance criteria that depend on it.

## 3. Architectural Alignment

- **CIP-56 (Token Standard).** This SDK is the EVM consumption surface for CIP-56. Practical adoption of CIP-56 by EVM developers depends on tooling like this existing.
- **`ComplianceGuard` as a public interface.** TokenProof provides the reference DAML implementation and this SDK provides the EVM-facing consumption layer. Compliledger withdrew #231 on 8 September 2026 and resubmitted as [#771](https://github.com/canton-foundation/canton-dev-fund/pull/771) with a different scope, so no funded dependency remains; the interface is public and the SDK also ships its own minimal implementation.
- **Zenith EVM atomic composability.** Zenith coordinates EVM execution from Canton-native workflows through its Gateway and `external_call()`, with both legs committing or failing together. The SDK follows that direction: compliance is checked on Canton, and the EVM contract receives an authorization. Where the Gateway path is available to third-party workflows, the SDK's settlement examples run on it rather than on a bridge.
- **Canton Foundation institutional DeFi narrative.** The Zenith launch release frames the gap directly: "financial applications such as lending protocols, automated market coordination systems, and structured yield strategies are overwhelmingly developed on Ethereum" ([Zenith launch, Mar 2026](https://www.globenewswire.com/news-release/2026/03/09/3252106/0/en/Zenith-launches-as-the-EVM-layer-for-Canton-Network-merging-Ethereum-s-developer-ecosystem-into-Wall-Street-s-blockchain.html)). Zenith brings that EVM logic onto Canton's institutional rails; this SDK is the compliance-enforcement layer that lets it deploy compliantly.
- **2026-2028 roadmap, RFP 12 (RWA Standards), item 12.1.** The Foundation's [2026-2028 Strategic Roadmap](https://github.com/canton-foundation/canton-dev-fund/blob/main/2026-2028-strategic-roadmap.md) asks under Identity, Credentials and KYC Standards for RWA Workflows for work focused on "open standards, interfaces, and reference implementations rather than proprietary identity or compliance services", covering credential classes such as KYC or KYB status, accreditation, jurisdiction, and eligibility to hold or transact in a particular asset. This SDK is that surface on the EVM side: open interfaces and reference implementations through which those credential checks get enforced in Solidity dApps, with no proprietary service anywhere in the stack. Per 12.1, the SDK accounts for work underway in the Identity and Metadata SIG and consumes the credential and party-metadata standards as they land, the same way it consumes CIP-56. Item 12.2 separately lists "Delivery-versus-payment and settlement-flow patterns"; the SDK's atomic ERC-20 to CIP-56 DvP covers the EVM side of that pattern.
- **Review Process priority areas.** The [Development Fund Proposal Review Process](https://github.com/canton-foundation/canton-dev-fund/blob/main/Development%20Fund%20Proposal%20Review%20Process.md) now directs reviewers to weigh alignment with the 2026-2027 Requests for Proposals, and its priority areas still apply: **Security and Resilience** ("monitoring, compliance, and third-party audit capabilities") and **App Building and Developer Experience** ("reduced developer friction", "interoperability across wallets, assets, and dApps"). The SDK delivers compliance capability and removes friction for EVM-background developers, who made up 71% of respondents to the Foundation's 2026 Developer Experience survey (41 respondents). (Quoted phrases are verbatim from those documents.)

- **Durability against standards.** The SDK defines no standard of its own. It is a consumption surface for CIP-56 and for the credential and party-metadata standards as they land, so its scope shrinks rather than conflicts as those standards mature: anything the platform absorbs natively is a dependency the SDK drops, not a feature it defends. What remains under any future standard is the application-layer surface, the dApp's own per-action permissions over objects that are not registry instruments.

## 4. Backward Compatibility

No backward compatibility impact on existing Canton or DAML systems. The SDK deploys new Solidity contracts on Zenith EVM and new DAML templates that consume TokenProof's `ComplianceGuard` interface as a transaction precondition; it does not modify TokenProof. Existing DAML applications, CIP-56 token issuers, and TokenProof PoC users are unaffected.

If the `ComplianceGuard` interface evolves, this SDK tracks it through versioned compatibility releases. Pre-submission coordination with the Compliledger team (see Rationale) is intended to align interface evolution timing.

The SDK does not depend on any third party's funding outcome, which matters now that #231 has been withdrawn. Our DAML-side PoC ([canton-localnet-poc](https://github.com/woof-software/canton-localnet-poc)) already contains a minimal open-source compliance-status template that can serve as the interim on-ledger source of compliance facts while TokenProof's `ComplianceGuard` matures. The Solidity interface in the current EVM PoC is a mock-shaped `isPermitted(address, action)` gate; the production verification interface described in section 2 (a bound authorization with signer, expiry, nonce and revocation semantics) replaces it in M1 and is not interface-compatible with the mock by design. Consumers integrate against the M1 interface, not the PoC.

---

## Dev Fund 2.0 Alignment

**RFP mapping.** RFP 12, RWA Standards, item 12.1: Identity, Credentials and KYC Standards for RWA Workflows. The proposal builds the consumption side of that standard: the open interfaces and reference implementation through which an application acts on a credential or compliance decision that was issued, verified and revoked on Canton. It defines no credential standard of its own and operates no compliance service, and it consumes the credential and party-metadata standards coming out of the Identity and Metadata SIG as they land.

**Ecosystem need and beneficiaries.** Canton holds compliance facts; a Solidity contract on Zenith cannot consult them directly, because Zenith's cross-VM call runs from a Daml workflow to the EVM and not the other way. Today every Zenith EVM team that needs application-level compliance has to design its own bridge between the two, and each design is a separate trust model for reviewers to audit. The direct beneficiaries are Solidity teams building regulated applications on Zenith, and the RWA issuers whose assets they handle; the indirect beneficiary is the ecosystem, which gets one reviewed surface with one set of security properties instead of one per team. Nothing here is operated by Woof as a service, and nothing requires changes to Canton, Daml, Splice or the OSS Wallet.

**Adoption path.** Milestone 1 ships the SDK with a reference integration and a published static-analysis report. Milestone 2 is gated on adoption: a $15,000 tranche is released only when at least two external teams have integrated the SDK in a test environment and published written feedback, and the reported target is at least three external teams publicly committing to evaluate by end of Milestone 2. We are validating the design against concrete issuer workflows before Milestone 1 rather than after; that work is visible in this PR thread.

---

# Milestones and Deliverables

## Milestone 1: Core Compliance Layer + Middleware Foundation

- **Estimated Delivery:** Month 3
- **Focus:** Build, security-review, and publish the core SDK; first end-to-end integration on testnet.
- **Deliverables / Value Metrics:**
  - Solidity contract layer covering the functional areas in Specification §2 (Canton-authorized entry points, authorization producers on the permit path, party↔address resolution) — deployed and verified on Zenith testnet. CIP-56 interaction is delivered through the TypeScript middleware over the #453 facade; contract-level CIP-56 helpers are included only if an EVM representation of a Canton asset is available to third parties on testnet by then.
  - TypeScript middleware published to npm (beta), covering typed compliance access, party resolution, and hybrid transaction assembly.
  - Local testing support (Hardhat-based) with mock compliance responses.
  - Security baseline established: static analysis (Slither) report with zero high-severity findings, published.
  - At least one integration exercising the < 1-hour quickstart — an external developer team if available, otherwise a Woof-built reference integration distinct from the SDK repo.

## Milestone 2: Reference dApps + Documentation

- **Estimated Delivery:** Month 6
- **Focus:** Reference dApps demonstrating common compliant patterns + complete documentation + community handoff.
- **Deliverables / Value Metrics:**
  - A set of open-source reference dApps covering the most-demanded compliant DeFi patterns (e.g. permissioned lending, compliant vault, KYC-gated trading) — deployed and operable on Zenith testnet.
  - Frontend integration building blocks (e.g. React components/hooks) for surfacing compliance state.
  - Documentation site: architecture overview, party-model walk-through, compliance-gated settlement flow, error handling, and a "deploy a compliant dApp" walkthrough.
  - **Target adoption signal:** ≥ 3 external Canton-ecosystem teams publicly committing to evaluate integration (reported as a value metric, not a hard gate).
  - Community review iteration via [forum.canton.network](https://forum.canton.network) with feedback incorporated.

---

# Acceptance Criteria

Milestone 1 is gated on deliverables, since adoption cannot precede a shipped artifact. Milestone 2 is split: the deliverables are gated on the hard criteria below, and a $15,000 tranche of the Milestone 2 budget is gated on external adoption. Where a criterion depends on Zenith mainnet timing, the environment note at the end of this section applies.

**Hard acceptance criteria (within our control):**
- **Time-to-first-integration:** A Solidity developer with no prior Canton experience can deploy a compliance-enabled contract (inheriting from the SDK base layer) and complete a compliance-gated operation in under 1 hour following the quickstart — demonstrated by a Woof-built reference integration (and by an external team's integration where available).
- **Authorization correctness:** On the permit path the contract reverts for every binding the authorization carries, each covered by its own test: wrong signer, wrong chain id, wrong verifying contract, wrong account, wrong recipient or amount, wrong action; expired authorization and one whose validity has not started; a reused nonce; an authorization signed by a rotated-out signer. On the issuing side: a request for a fact that was revoked or archived before issuance is refused, and an unavailable or stale Canton source produces a refusal to issue, never a silent pass. Revocation after issuance is bounded, not eliminated: the deadline the contract checks is signed into the permit and is the earlier of the time of the ledger read plus the validity window and the fact's own timestamp plus the configured maximum age, both published with the milestone, so a permit issued before a revocation stays valid at most until that deadline and never past the point where the fact itself would count as stale; if a later revision adds an on-chain revocation signal, its test asserts rejection only after that signal has been delivered, never an instant synchronisation of the two systems. One liveness case is covered as well: a failed business call after a valid authorization leaves no consumed nonce behind. The suite runs on Zenith testnet and the results are published with the milestone. The permit-path suite is the paid acceptance criterion for this milestone. Criteria for the Gateway-coordinated path will be added by a separate revision once that path is agreed with Zenith; they do not replace or relax the permit-path criteria.
- **Security posture:** Solidity contracts pass Slither with zero high-severity findings. DAML helpers (if any) pass Certora Daml Package Analyzer with zero high-severity findings. (A third-party external audit is not budgeted within this grant; the security baseline is static analysis plus internal manual review, and the SDK is deliberately scoped as a thin, reviewable layer.)
- **Boundary acknowledgement:** obtained pre-submission — the Compliledger team confirmed the efforts are complementary ([reply of 22 July 2026](https://github.com/canton-foundation/canton-dev-fund/pull/231#issuecomment-4718622123), on the since-withdrawn #231). Ongoing coordination on `ComplianceGuard` interface stability continues in the PR threads as both efforts mature.

**Milestone 2 adoption gate (gates the $15,000 adoption tranche, see Funding):**
- **≥ 2 external teams** have integrated the SDK in a test environment and provided written feedback on the integration surface, published in the PR thread or on the Canton forum.

**Adoption targets (reported, not gated):**
- ≥ 3 external teams publicly committing to evaluate integration by end of M2.
- At least one integration exercised end-to-end in a production-track deployment.

**Environment note.** Where these criteria reference "Zenith testnet", an equivalent public EVM test environment may be substituted if public Zenith testnet access is not yet available at execution time; the permit path does not depend on Zenith-specific features, so all functional criteria remain verifiable independently of Zenith availability.

---

# Funding

**Total Funding Request:** $120,000 USD, denominated in Canton Coin at the prevailing USD/CC rate at each milestone's acceptance.

## Payment Breakdown by Milestone

- **Milestone 1** (Core Compliance Layer + Middleware Foundation): $60,000 USD in CC upon committee acceptance.
- **Milestone 2** (Reference dApps + Documentation): $60,000 USD in CC, split into two tranches:
  - **$45,000 USD in CC** upon final release and acceptance of the deliverables.
  - **$15,000 USD in CC** upon the Milestone 2 adoption gate being met (see Acceptance Criteria). This tranche remains claimable for three months after Milestone 2 delivery, and is forfeited if the gate is not met in that window.

The adoption tranche is deliberate. Delivering the code is within our control and is paid on delivery; external teams choosing to integrate is not fully within our control, so that portion of the budget is placed at risk rather than asserted as a target.

## Volatility Stipulation

[CIP-0100](https://github.com/canton-foundation/cips/blob/main/cip-0100/cip-0100.md) requires proposals with milestones at or beyond six months to state explicitly how CC price volatility is handled. This proposal handles it by denominating the engineering budget in USD and converting to Canton Coin at the prevailing USD/CC rate on the date each milestone is accepted. Payment is made in CC; no CC amount is fixed in advance, so neither party carries an open-ended exposure to rate movement between approval and delivery.

The rationale is that the cost of the work is a USD cost, and Canton Coin has moved across a wide range in its short trading history. Fixing a CC quantity at submission would make the real value of delivery a function of the rate on an arbitrary date rather than of the work performed. Should the project timeline extend beyond 6 months due to Committee-requested scope changes, any remaining milestones are renegotiated on the same basis.

---

# Co-Marketing

Upon release, Woof will collaborate with the Canton Foundation on:

- **Announcement coordination** — joint blog post or press release at v1.0 launch.
- **Technical blog** — engineering deep-dive on the EVM ↔ DAML compliance integration pattern, published on Canton Foundation channels.
- **Developer outreach** — 1 conference talk or workshop session at a Canton ecosystem event during the milestone window (in-person or virtual).
- **Reference dApp showcase** — Foundation may feature the reference implementations in ecosystem marketing materials.

---

# Distribution & Go-to-Market

How developers discover, adopt, and depend on the SDK:

- **Distribution channels.** Published as a versioned npm package and an open-source Solidity package verified on the Zenith block explorer, with a public GitHub repository. Standard `npm install` / inherit-and-import flow — zero bespoke setup, the path EVM developers already expect.
- **Discovery.** A documentation site with a "deploy a compliant dApp" quickstart; launch announcement on forum.canton.network and Canton ecosystem channels; a technical blog post; and a short walkthrough video. Inclusion in the Canton ecosystem tooling listings (e.g. CCTools directory) once live.
- **Onboarding funnel.** The reference dApps (e.g. permissioned lending, compliant vault, KYC-gated trading) double as copy-paste starting points — a developer forks the closest pattern and adapts it, rather than starting from a blank file. The < 1-hour quickstart is the conversion metric.
- **Targeted adoption.** Direct outreach to the EVM-background teams entering via Zenith (71% of respondents in the 2026 Developer Experience survey), plus the TradFi/hybrid teams (83% of surveyed projects) for whom compliance is mandatory. Adoption is an explicit acceptance criterion for Milestone 2, not a reported metric (see Acceptance Criteria).
- **Ecosystem pull.** The SDK rides the same demand curve as CIP-56 adoption — every team that needs on-ledger compliance is a candidate consumer of the Solidity surface.

---

# Rationale

## Why this approach

**Solidity-side wrappers are the lowest-friction surface for EVM developers.** Inherit-and-extend with familiar modifiers (`onlyKYC`, `onlyParty`) maps to standard EVM patterns; developers do not need to learn DAML or Canton's party model to add compliance gating. The cost of building this once as security-reviewed infrastructure is recovered many times over across the ecosystem.

**Two-layer architecture (Solidity + TypeScript) follows Ethereum precedent.** OpenZeppelin Contracts + Ethers.js / Viem is the canonical model that EVM developers expect. This SDK adopts that mental model for Zenith.

## Alternatives considered

- **Extend TokenProof to include Solidity bindings ourselves, contributing to that repo.** Considered. The Compliledger team is staffed for DAML protocol work; Solidity contract authorship and EVM-dApp reference implementations sit outside their scope per their own PR statement. Splitting ownership cleanly (DAML primitive vs EVM consumption layer) is healthier than concentrating both layers in one team's roadmap.
- **Build a higher-level abstraction (single-call "do everything compliant" wrapper) instead of base contracts.** Rejected: prevents composition, locks developers into our opinions. Base contracts + interfaces match Solidity ergonomics and let developers compose with their existing inheritance hierarchies.
- **Defer Solidity work until OpenZeppelin Canton Stack ships its Reference Implementations.** Rejected: OpenZeppelin's stack is Daml-centric per their own scope description; their Reference Implementations are not Solidity contracts on Zenith. There is no overlap.
- **Leave the EVM-side compliance surface to the platform's own roadmap.** Considered. Digital Asset's Registry App scopes itself to asset-model authorization (see the boundary in Motivation), and no public roadmap covers an application-layer compliance surface for Solidity developers on Zenith. The Dev Fund exists precisely to fund open, MIT-licensed infrastructure that composes with the platform's products. If the platform later ships native equivalents, we would align interfaces rather than compete, and the reference dApps retain their value as open app-layer patterns.

- **Build on Chainlink ACE's policy engine.** Considered. ACE's `PolicyProtected` and `runPolicy` surface is the closest EVM-generic prior art for modifier-based gating. It is licensed under BUSL 1.1; the repository's additional-use grant lists no permitted production purposes, so a production right for a use like ours is not confirmed without a separate license, and each version converts to MIT only on its own change date (October 2029 for the current one). The SDK keeps its own minimal authorization verifier under MIT and stays interface-compatible where practical, so a team already licensing ACE can wrap the Canton authorization as an ACE policy.

## Fit with existing ecosystem tooling

- Implements against TokenProof's public `ComplianceGuard` interface — coordinated with Compliledger before submission; their funding proposal #231 has since been withdrawn.
- Uses [Daml Package Analyzer #130 (Certora)](https://github.com/canton-foundation/canton-dev-fund/pull/130) as audit baseline for any DAML helpers.
- Coexists with [OpenZeppelin Canton Stack #262](https://github.com/canton-foundation/canton-dev-fund/pull/262) — different layers (Daml libraries vs EVM contracts). Their [Milestone 2 acceptance criteria](https://github.com/canton-foundation/canton-dev-fund/issues/565) commit to architecture documentation that "includes integration patterns for custody, compliance, and risk management systems", so integration with external compliance systems is an explicit part of their funded scope; this SDK is one such system on the EVM side.
- Independent of [LynoBridge #147](https://github.com/canton-foundation/canton-dev-fund/pull/147) — bridge is token-transfer; this SDK is compliance enforcement.
- Complementary to [CIP-56/ERC-20 Middleware #453](https://github.com/canton-foundation/canton-dev-fund/pull/453) — that is a JSON-RPC/wallet transport layer over CIP-56; this SDK is contract-level compliance logic. Composable, not overlapping.
- Complementary to [BlockTravel #190](https://github.com/canton-foundation/canton-dev-fund/pull/190) — an off-ledger compliance decisioning service that can act as one backend behind our `ICantonKYC` interface.
- Complementary to native registrar controls in [Digital Asset's Registry App](https://docs.digitalasset.com/registry/guides/blocklist) — party blocklisting (`checkBlocklist`, since v0.13) and credential-based instrument [allowlisting](https://docs.digitalasset.com/registry/features/allowlist) are issuer-side **asset-model** controls; this SDK is the **application-layer** counterpart on the EVM side (full boundary in Motivation). The SDK reimplements neither.

## Pre-submission coordination

We coordinated with the Compliledger team (@Mharris40) via PR #231 before opening this proposal; that proposal was withdrawn on 8 September 2026 and resubmitted as #771 with a different scope. The TokenProof team reviewed our scope and **confirmed the two efforts are complementary, not overlapping**, and is open to coordinating on `ComplianceGuard` interface stability as both efforts progress ([their reply](https://github.com/canton-foundation/canton-dev-fund/pull/231#issuecomment-4718622123), in response to [our note](https://github.com/canton-foundation/canton-dev-fund/pull/231#issuecomment-4718506264)). We will keep our Solidity wrappers adaptable to their interface as it firms up. If their design later expands to cover the Solidity side directly, we will revise our scope rather than duplicate, and would offer to contribute to their milestones.

---

# Sustainability

Who operates and maintains the SDK after the grant period:

- **Primary steward: Woof.** As an active Compound DAO contractor team building EVM infrastructure long-term, Woof commits to maintaining the SDK as part of its ongoing open-source footprint — minimum 6 months of bug fixes, CIP-56 spec compatibility, and Zenith version updates post-delivery, with security disclosures addressed within 48 hours.
- **Low maintenance surface by design.** The SDK is a thin, security-reviewed layer over a `ComplianceGuard`-shaped interface; most spec evolution is absorbed on the DAML side, limiting our long-term maintenance burden.
- **Community handoff path.** All code is MIT/Apache-2.0 with a public issue tracker and contribution guide. As the Token Standards / Regulatory Compliance SIG matures, stewardship can transition to or be shared with the relevant SIG, so the SDK does not depend on a single vendor indefinitely.
- **No ongoing protocol fees or hosted dependency** — the SDK is a library consumed by other teams' deployments, so it does not require Woof to run infrastructure to keep functioning.

---

# Open Source

- Solidity contracts: MIT.
- TypeScript middleware: MIT.
- Reference implementations: Apache-2.0.
- Documentation: CC-BY-4.0.
- Repositories: `github.com/woof-software/canton-compliance` (TypeScript middleware) and `github.com/woof-software/canton-compliance-contracts` (Solidity contracts).

---

# Team

**Woof** ([woof.software](https://woof.software)) — senior EVM engineers for DeFi. Active Compound DAO contractor team. Public artifacts directly relevant to this proposal:

- **[`comet`](https://github.com/woof-software)** — Compound v3 core contributions.
- **[`comet-wrapper`](https://github.com/woof-software)** — production Solidity ERC-4626 + ERC-7246 wrapper for Compound III; precedent for the Compliant Token Vault reference implementation.
- **[`compound-multiplier`](https://github.com/woof-software)** — risk-adjusted DeFi engineering on EVM.
- **[`migrator-v2`](https://github.com/woof-software)** — cross-protocol position migration; precedent for hybrid (multi-protocol) transaction building.

Operational track record: delivering the deployments behind Compound's market expansion to Optimism, Mantle, and additional networks. Solidity, security review (Slither, manual review), and audit-grade engineering are daily practice.

We do not present ourselves as DAML protocol experts. This proposal is scoped accordingly — the EVM / Solidity side specifically, with the DAML side kept to the minimum the authorization path requires.

---

# References

- [TokenProof](https://github.com/Compliledger/canton_tokenproof) — reference DAML implementation of the `ComplianceGuard` interface (funding proposal [#231](https://github.com/canton-foundation/canton-dev-fund/pull/231) withdrawn 8 September 2026, resubmitted as [#771](https://github.com/canton-foundation/canton-dev-fund/pull/771) with a different scope)
- [PR #130 Daml Package Analyzer (Certora)](https://github.com/canton-foundation/canton-dev-fund/pull/130) — audit baseline
- [PR #262 OpenZeppelin Canton Stack](https://github.com/canton-foundation/canton-dev-fund/pull/262) — adjacent Daml library work
- [PR #147 LynoBridge](https://github.com/canton-foundation/canton-dev-fund/pull/147) — bridging context
- [PR #453 CIP-56/ERC-20 Middleware](https://github.com/canton-foundation/canton-dev-fund/pull/453) — adjacent JSON-RPC/wallet transport layer (complementary)
- [PR #190 BlockTravel](https://github.com/canton-foundation/canton-dev-fund/pull/190) — adjacent off-ledger compliance decisioning service
- [CIP-56 Token Standard](https://github.com/global-synchronizer-foundation/cips/blob/main/cip-0056/cip-0056.md)
- [Canton DevEx Survey 2026](https://forum.canton.network/t/canton-network-developer-experience-and-tooling-survey-analysis-2026/8412)
- [CIP-0100 Dev Fund governance](https://github.com/canton-foundation/cips/blob/main/cip-0100/cip-0100.md)
- [Zenith atomic transactions (The Block, March 2026)](https://www.theblock.co/post/394288)
