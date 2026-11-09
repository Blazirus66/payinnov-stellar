# Payinnov on Stellar: milestones and budget

Three tranches from 5 December 2026 to 21 March 2027. Total budget: $125,000 (tranche 1: $48,900; tranche 2: $47,800; tranche 3: $28,300). Components and building blocks carry the names used in the architecture.

---

## Tranche 1 — The settlement contract and Stellar acceptance: Soroban, Stellar Wallets Kit, Privy and Soroswap implemented ($48,900)

### 1.1 MerchantRegistry and PaymentSplitter

**Description.** The two Soroban contracts of the architecture, in Rust on `soroban-sdk` 29 and the OpenZeppelin `stellar-contracts` 0.8.0 libraries (roles, pause, upgrade with migration). MerchantRegistry: per-merchant configuration (Stellar destination, per-asset Bridge addresses confirmed by the merchant, settlement asset EURC or USDC, kept share, payout threshold, commission), 5% commission cap hard-coded, fee changes delayed by 24 hours with an event, every destination reserved to the merchant's signature. PaymentSplitter: `settle_sale` signed by the customer with an exact amount, `receive_and_settle` under the attribution role (receipt of the Circle message and settlement in the same invocation), `settle_received` for a message presented by a third party, split between commission, accrued local share and kept share, threshold-based `payout` callable by anyone, `PaymentSettled` and `LocalPaidOut` events, replays rejected by `sale_ref`, pause with no withdrawal function, upgrades under a 48-hour delay. Unit tests with mainnet limits, authorisations asserted, fuzzing on the split invariant, mutation testing, continuous integration with `stellar contract build`, `clippy` and Scout static analysis.

**How to measure completion.**
- Both contracts are deployed on testnet, identifiers published in the repository with the WASM hash and the matching commit.
- A testnet `settle_sale` of 8 EURC splits 8 EURC between commission, local share and kept share according to the merchant's configuration, `PaymentSettled` event published with its hash; the same invocation replayed with the same `sale_ref` fails (public tests).
- `propose_fee` at 501 basis points is rejected; at 300 basis points it has no effect for 17,280 ledgers and then applies (public tests); `set_destinations` and `set_local_destinations` signed by any account other than the merchant's destination are rejected (public tests); `payout` below the threshold transfers nothing, at the threshold it transfers the accrued share to the Bridge address for the settlement asset (hashes published).
- The test suite, the fuzzing of the invariant commission + kept share + local share = gross and the mutation report run in continuous integration, output published.

**Budget.** $17,600

**Delivery date.** 20 December 2026

### 1.2 Stellar payment page and Stellar Wallets Kit

**Description.** The payment page gains the "Pay with a Stellar wallet" button: Stellar Wallets Kit v2 (xBull, Freighter, Hana, Albedo, Lobstr via WalletConnect 2.0), construction of the `settle_sale` invocation with an exact amount in EURC or USDC, simulation through Stellar RPC, checks of the network passphrase, the contract address and the customer's trustline before signing, submission and tracking up to the event. Till QR code that opens the page on the customer's phone with the sale identifier.

**How to measure completion.**
- A complete testnet payment run from the page with at least two wallets from the kit, including Lobstr via WalletConnect from a phone, invocation hash published.
- An invocation prepared against a wrong passphrase or another contract is blocked in the page before signing; a customer without a trustline on the chosen asset sees the message and cannot sign (public tests).
- A till QR code scanned with a phone camera opens the page on the right sale (short video published); an invocation signed by Lobstr through WalletConnect from the phone is accepted by the contract (hash published).

**Budget.** $8,500

**Delivery date.** 27 December 2026

### 1.3 Merchant accounts: Privy, own wallet, sponsored reserves, OpenZeppelin Relayer

**Description.** Merchant onboarding in the dashboard: creation of a Stellar account behind their login through Privy, or connection of their own wallet through the Stellar Wallets Kit; account creation and EURC and USDC trustlines through sponsored reserves from Payinnov's sponsoring account; registration in the MerchantRegistry signed by the merchant's destination. Self-hosted OpenZeppelin Relayer: fee-bumps for the merchant's transactions (slider, destinations) and submission of the orchestrator's transactions with sequence channels. Payinnov's accounts (commission, sponsoring, attribution, onboarding, pricing, pauser, upgrader) created, sensitive roles on multisignature, keys in a secrets manager.

**How to measure completion.**
- On testnet, a merchant onboarded through Privy and a merchant onboarded with Freighter each have an account with the two trustlines, sponsored by Payinnov's account (accounts and sponsoring hashes published), and their configuration in the MerchantRegistry.
- The Privy merchant changes their kept share from the dashboard without holding any XLM: the transaction is fee-bumped by the relayer (hash published); the same operation attempted by a Payinnov account is rejected (public test).
- Payinnov's seven accounts are listed with their roles; `Upgrader`, `Pauser` and `Pricing` require two signatures (account configuration published).

**Budget.** $11,400

**Delivery date.** 6 January 2027

### 1.4 Soroswap conversion inside the PaymentSplitter and fallback

**Description.** Call to the Soroswap Aggregator from the PaymentSplitter: `swap_tokens_for_exact_tokens` for the exact amount, in the merchant's settlement asset, of a sale paid in the other asset by a customer (surplus returned), `swap_exact_tokens_for_tokens` with a minimum amount for a sale arriving through CCTP; distribution and price taken from the Soroswap API by the payment page or the orchestrator; aggregator address and slippage cap in the contract instance; fallback: call without a route, pass-through in USDC, `converted = false` event. Tests on a network snapshot against the real WASM of the aggregator and of the USDC and EURC SACs.

**How to measure completion.**
- On testnet, a `settle_sale` paid in test USDC delivers exactly the amount of test EURC requested and returns the surplus to the customer (hash published); with an unreachable minimum amount, the invocation fails as a whole and no balance moves (public test).
- A `receive_and_settle` without a route splits in USDC and emits `converted = false`; with a route, it splits in EURC and emits `converted = true` (hashes published).
- The network-snapshot test against the deployed aggregator WASM passes in continuous integration, output published.

**Budget.** $11,400

**Delivery date.** 15 January 2027

---

## Tranche 2 — From the other chains and out to the bank: CCTP V2, Bridge, Mercuryo, Alfred Pay and DeFindex implemented ($47,800)

### 2.1 EVM contract in Stellar settlement mode, CCTP V2 and orchestrator

**Description.** Per-merchant "Stellar settlement" mode on the EVM split contract: `settleToStellar(saleRef, minUsdcOut)` function callable by the operator or the merchant, swap of the asset received into native USDC on the chain's DEX with a minimum amount bounded by the merchant's tolerance, then `depositForBurnWithHook` to domain 27 with `mintRecipient` and `destinationCaller` set to the `CctpForwarder` and the PaymentSplitter as recipient in the hook data, sale identifier as payload. Orchestrator: tracking of EVM sales, call to `settleToStellar`, polling of Circle's attestation, `receive_and_settle` on the PaymentSplitter (bridge receipt and settlement in one invocation), `settle_received` when the message was presented by a third party, six-to-seven decimal conversion, recovery after incidents, per-sale log. Ethereum, Base, Polygon and Sonic.

**How to measure completion.**
- A testnet sale paid in test USDC on Base Sepolia and one paid in test ETH on Ethereum Sepolia are each swapped, burned, attested, minted and settled on Stellar testnet by a single `receive_and_settle` invocation, hashes on both chains and `PaymentSettled` event published.
- The EVM contract rejects `settleToStellar` for an already-settled sale, for a merchant not in Stellar mode and for a `minUsdcOut` below the tolerance (public tests); no code path allows a `mintRecipient` other than the `CctpForwarder` (public test); a message presented to the `CctpForwarder` by a third-party account is then settled through `settle_received` (public test, hashes published).
- A testnet transfer of 1.234567 test USDC yields exactly 1.2345670 on Stellar testnet (message and balance amounts published).
- The orchestrator stopped between burn and mint, then restarted, completes the sale without counting it twice (public test, log published).

**Budget.** $16,500

**Delivery date.** 29 January 2027

### 2.2 Bridge: merchant verification, liquidation addresses, onboarding states

**Description.** Payinnov as a Bridge platform customer; for each merchant: creation of the Bridge customer, hosted verification link, registration of the bank account (IBAN and BIC, or Pix key), creation of the liquidation addresses, one per asset (source Stellar, EURC and USDC, destination SEPA, Pix, SPEI, pounds or Colombian pesos), return address = the merchant's Stellar account; confirmation by the merchant in the dashboard, which writes account and muxed id into the MerchantRegistry with their signature; drain webhooks (`funds_received`, `payment_submitted`, `payment_processed`, `returned`) reconciled with the `PaymentSettled` event and shown in the dashboard; onboarding states `registered`, `Bridge verification in progress`, `payout active`, `alternative payout`; screening of refused activities at onboarding. Against the Bridge API in sandbox.

**How to measure completion.**
- In the Bridge sandbox, a test merchant completes the verification link, obtains their EURC and USDC liquidation addresses, and confirms them from the dashboard with their signature in the testnet MerchantRegistry (Bridge identifiers and hash published).
- A testnet `settle_sale` to that merchant followed by a `payout` sends their local share to the muxed address for their asset, and the matching drain appears in the dashboard next to the `LocalPaidOut` event (screenshots and identifiers published).
- A merchant whose verification is not complete has their sale settled entirely to their Stellar account with the `LocalPending` event; a webhook with an invalid or replayed signature is rejected (public tests).

**Budget.** $11,400

**Delivery date.** 8 February 2027

### 2.3 Mercuryo, Alfred Pay and DeFindex from the dashboard

**Description.** Three dashboard actions signed by the merchant from their account (Privy or own wallet): sale of USDC to card through the Mercuryo widget (iframe, off-ramp enabled, merchant KYC at Mercuryo); withdrawal in reais via Pix or pesos via SPEI through Alfred Pay (merchant verification at Alfred, order created by the dashboard, transfer to the order's deposit address, tracking up to payout); deposit and withdrawal of the kept share into a DeFindex vault (`depositToVault`, `withdrawFromVault`, vault shares on the merchant's account, balance displayed). Common payout adapter (`quote`, `initiate`, `status`), one implementation per partner.

**How to measure completion.**
- A DeFindex deposit and withdrawal run on testnet from the dashboard by a Privy merchant, vault shares visible on their account, hashes published.
- A test Alfred Pay order created from the dashboard, funded by a Stellar USDC transfer signed by the merchant, tracked to its final state in Alfred's test environment (identifiers published).
- The Mercuryo widget opens from the dashboard in sandbox mode with Stellar USDC preselected for sale, and a transfer to the address supplied by the widget is signed by the merchant (screenshot and hash published).
- A withdrawal attempted by a Payinnov account on a merchant's account is rejected (public test).

**Budget.** $11,400

**Delivery date.** 18 February 2027

### 2.4 Detection, FEC export, threat model and monitoring plan

**Description.** Reading of `PaymentSettled` and `LocalPending` events through Stellar RPC with a persistent cursor, catch-up through Horizon after a stop, sale confirmation to the till and the plugin, reconciliation of `LocalPaidOut` payouts with Bridge drains, exceptions in the dashboard (sale without event, payout without drain, drain in error), FEC export fed by the same stream. Threat model and monitoring plan published: threats by category with their mitigation (Circle message, commission cap, destinations, replay, degraded swap, CCTP recipient, decimals, role keys, code upgrade), OpenZeppelin Monitor configured on both contracts with the architecture's alerts.

**How to measure completion.**
- A testnet sale is confirmed at the test till on receipt of the event, and appears in the test FEC export with gross, commission, kept share and local share (file published).
- The event reader stopped during a testnet sale then restarted catches the sale up without counting it twice (public test, log published).
- The threat model and the monitoring plan are public in the repository, each threat linked to a mitigation or an accepted residual risk.
- An injected event (Circle message not presented for more than an hour, USDC received outside `receive_and_settle`, `LocalPending`, `FeeProposed`, failed transfer to a destination) triggers its alert in OpenZeppelin Monitor (public tests).

**Budget.** $8,500

**Delivery date.** 25 February 2027

---

## Tranche 3 — Production: Soroban, Stellar Wallets Kit, Privy, Soroswap, CCTP V2, Bridge, Mercuryo, Alfred Pay and DeFindex live ($28,300)

### 3.1 Contracts and payment page in production, first merchants settled on Stellar

**Description.** MerchantRegistry and PaymentSplitter deployed on mainnet with a SEP-55 verified build (metadata, GitHub attestation, release WASM); Payinnov's accounts and roles in production, multisignature on sensitive roles; payment page in mainnet configuration; USDC/EURC pool seeded on Soroswap; the current merchants who wish to are onboarded (Stellar account, destinations) and switched to Stellar settlement for native Stellar payments.

**How to measure completion.**
- Both mainnet contracts are published with their SEP-55 attestation and the hash matches the release WASM (identifiers and attestation link published).
- A `settle_sale` of a few euros run by the team on mainnet from the payment page, with Soroswap conversion, splits and emits `PaymentSettled` (hash published).
- The role accounts and their multisignature configuration are published; `propose_fee` above the cap is rejected on mainnet (public test).
- Adoption, counted from go-live to the submission of this tranche: at least 5 distinct merchant wallets (excluding team accounts) have received a sale settled by the PaymentSplitter, footprint published (contract identifiers, sponsoring account, list of merchant accounts).
- Adoption, same window: at least 300 settled transactions (successful `settle_sale` or `receive_and_settle` invocations) and a cumulative settled volume of at least EUR 20,000 equivalent, list of hashes published; any shortfall is reported as is with the tranche.

**Budget.** $11,400

**Delivery date.** 7 March 2027

### 3.2 CCTP, Bridge, Mercuryo, Alfred Pay and DeFindex in production

**Description.** "Stellar settlement" mode of the EVM contract deployed on Ethereum, Base, Polygon and Sonic, orchestrator in production with Circle's mainnet addresses; Payinnov verified by Bridge in production, first merchants verified, SEPA and Pix liquidation addresses written into the registry; Mercuryo widget and Alfred Pay withdrawals in production; DeFindex vault in production; OpenZeppelin Monitor alerts active and alert channel documented.

**How to measure completion.**
- A sale of a few euros paid by the team in USDC on Base is burned, attested, minted and settled on Stellar mainnet by a single `receive_and_settle` invocation, hashes on both chains published.
- A `payout` of a few euros from the PaymentSplitter to the muxed Bridge address of a team merchant produces a `payment_processed` drain (drain identifier and Stellar hash published).
- A DeFindex deposit of a few units and an Alfred Pay order of a few units run by the team on mainnet, hashes and identifiers published; the Mercuryo widget in production opens a sale of Stellar USDC (screenshot published).
- Monitors active: a test alert triggered and received on the documented channel (screenshot published).

**Budget.** $8,500

**Delivery date.** 14 March 2027

### 3.3 E-commerce plugins and till app released with Stellar payment

**Description.** Released versions of the WooCommerce, PrestaShop and Magento plugins and of the till app with Stellar payment switchable on by the merchant: Stellar button on the payment page, till QR code to the page, confirmation on the event, display of the kept share and payouts; public integration guide for a merchant (onboarding, Bridge verification, slider, withdrawals) and for a developer (API, confirmation webhooks, `PaymentSettled` event).

**How to measure completion.**
- The three plugins and the till app are released in a version carrying Stellar payment, release notes published; a payment of a few euros run by the team from each of the three plugins and from the till on mainnet, hashes published.
- The integration guide is public and a developer from outside the team completes a full testnet payment by following it (write-up published).
- The production FEC export of a team merchant contains their Stellar sales for the month with gross, commission and shares (anonymised file published).

**Budget.** $8,400

**Delivery date.** 21 March 2027
