---
name: agentbank-pay
description: Use the installed AgentBank MCP across setup, identity, payment, wallet, recipient, tracking, and recovery workflows; check readiness, create direct or two-hop payments, use the shared Privy wallet, and recover durable payments safely.
---

# AgentBank Pay

## Use the AgentBank MCP

Before AgentBank work, make sure the AgentBank MCP is installed and loaded by checking whether the
current client exposes `whoami`, `get_instructions`, and `begin_agent_onboarding`.

If these tools are absent, stop and tell the human that the AgentBank MCP must be made available
before the workflow can continue. Do not install, configure, reload, or provide installation
commands from this skill.

When the tools are available, preserve the original task and continue immediately. Call `whoami`
without waiting for another user message, then use the relevant AgentBank MCP tools across setup,
payment, tracking, recovery, recipient, and wallet workflows.

Do not replace AgentBank MCP calls with direct HTTP requests, locally constructed protocol payloads,
or unrelated payment tools. Treat AgentBank Core as the authority for approvals, locked routes,
payment instructions, transaction verification, and terminal payment state.

Do not use or ask for legacy intent, route-agreement, approval, settlement, partner, raw-swap, or
progress-reporting mutations. They are not exposed by the production server.

## Safety rules

- Never request or expose private keys, seed phrases, AgentBank JWTs, Privy tokens, authorization keys, or World ID proofs.
- Never infer a wallet address, bank account, recipient, token contract, chain, decimals, amount, or calldata from weak context.
- Use decimal strings for human amounts.
- Build a complete recipient, source amount, destination amount, fees with fee currencies, route, and expiry summary before `create_payment`.
- Set `confirmed_by_user=true` only when an explicit human confirmation or the account's active platform standing authorization permits the payment. If the runtime does not expose a valid standing authorization, ask the human to confirm.
- Use only a current server-generated payment instruction to move funds.
- A transaction hash or successful receipt is not payment completion. Trust
  `get_payment`.
- Do not expose partner identity. Current quote tools intentionally return
  anonymous offers.

## Runtime source of truth

Call `get_instructions` with the relevant journey when starting an unfamiliar
flow or recovering after an interruption:

```text
setup
pay
track
recover
manage_recipients
manage_wallets
```

The MCP also exposes `agentbank://guides/routing` and
`agentbank://instructions/{journey}` as resources. Follow newer runtime guidance
when it does not conflict with these safety rules.

## Request IDs

Generate a stable `request_id` for each logical create, continue, execute,
approve, cancel, recipient-correction, recipient-creation, or
recipient-replacement mutation.

Reuse the same ID only when retrying the same tool call with the same payload.
Use a different ID for a changed payload or a different on-chain transaction.
Do not treat a wallet transaction request ID as a payment ID.

## Asset and amount format

Use canonical assets:

```json
{ "type": "crypto", "ticker": "USDC", "chain": "worldchain" }
```

```json
{ "type": "fiat", "symbol": "VND" }
```

For `exact_source`, put the exact amount in `source.amount`. For
`exact_destination`, put it in `destination.amount`.

Use `list_currencies` whenever a ticker, fiat code, chain, token address, or
decimals need verification. Always pass the complete structured asset object;
compound asset strings are invalid.

## 1. Set up the agent

As soon as the MCP tools are available, call `whoami` without waiting for
another user message.

If it succeeds, call `get_account_status` and continue with the existing
installation.

If it returns `MISSING_CREDENTIAL`:

1. Call `begin_agent_onboarding` once immediately. This creates or resumes the
   local installation and Privy device flow.
2. Show `authorization_url` and explain that its one browser approval claims
   the agent and grants shared Privy wallet access.
3. Call `wait_for_agent_onboarding` immediately with the returned
   `enrollment_id`; it polls while the human approves. If it times out while the
   enrollment remains pending, call it again with the same enrollment ID.
4. Verify `agentbank_claimed`, `privy_authorized`, `wallet_bound`, and
   `authenticated`.
5. Call `whoami`, `check_my_scopes`, `get_account_status`, and `list_wallets`.

Browser approval is the only required human step. Do not ask the human to
repeat the setup request after MCP availability is confirmed. Do not call a legacy registration
alias or start a second onboarding flow while one is pending.

When the human asks to log out or reset this local agent:

1. Explain that revocation invalidates the installation, sessions, and bound
   wallet authorization for this agent and clears local credentials.
2. Obtain explicit confirmation.
3. Call `revoke_agent({"confirm":true})`.

## 2. Handle identity requirements

Use `check_verification_status` when KYC or badges matter. Its `markets` result is provider-agnostic
KYC readiness by country, not live corridor availability; use the quote-book tools for live pairs and rates.

If KYC is missing:

1. Call `do_kyc`.
2. If it returns `already_verified`, continue.
3. If it returns `kyc_url`, show the URL to the human.
4. Ask the human to complete Didit verification.
5. Call `check_verification_status` again before retrying a gated action.

If `create_payment` returns `status=need_review` with `reason=rail_not_ready`, no payment was created
and no funds moved. Show its KYC state, badges, and `markets` readiness. Wait until the market is
`APPROVED`, then retry `create_payment` with the same `request_id`; refresh and reconfirm if the quote
has expired. Do not call `continue_payment`, create a replacement payment, or ask for World ID approval
while the rail is not ready.

Use `get_verification_guidance` for first-party profile or World ID badge guidance. Never ask the human
to paste identity proofs into chat.

Payment World ID approval is a separate per-payment action returned by
`create_payment`. Do not replace it with a badge check.

AgentKit wallet verification is separate from AgentBank KYC, WORLDID badges, and
payment approval. When requested:

1. Call `verify_agent_kit` without wallet IDs or addresses.
2. Show `verification_url` and ask the human to open or scan it in World App.
3. After the human finishes, call `verify_agent_kit` again until it returns
   `status=verified`.

Do not run the AgentKit CLI manually or request a World ID proof. The tool uses
World's pinned official flow, refreshes Core, and does not move funds.

## 3. Understand the payment request

Collect:

- source asset and chain, or source fiat currency;
- exact source or exact destination amount;
- destination asset/currency and country;
- recipient or destination wallet;
- optional routing preference.

Routing preferences are:

```text
balanced
lowest_total_cost
fastest
highest_success_rate
```

Use `balanced` when the user gives no preference.

Do not silently split an amount, switch chains, change exactness, or choose a
different recipient.

## 4. Resolve the recipient

For a saved destination:

1. Call `list_recipients`.
2. Reuse a record only when the user request clearly matches its rail and
   canonical fields.
3. Call `get_recipient` when full fields are needed.
4. Ask the human to choose when multiple records match.

Do not describe a recipient as invalid solely because `verified` is false.
That flag means verified holder metadata has not been established; route and
partner validation remain authoritative.

For a new fiat or crypto recipient, call `create_recipient` with one or more of:

- canonical `fields`;
- structured `bank_info`;
- labeled `pasted_text`;
- raw `qr_content`;
- a QR image.

When the human sends recipient information through chat as an image, raw QR
payload, pasted bank text, account/holder details, or structured bank data,
call `create_recipient` before `estimate_payment` or `create_payment`. Use the
returned `recipient_id` or canonical `recipient_fields`; do not manually copy
unvalidated image/QR fields directly into a payment request.

For local stdio, an image may use an absolute `image.path`. Remote clients use
`image.data_base64`. The image must contain a readable QR. If it is a text-only
screenshot, pass the visible details as `pasted_text` or `bank_info`; OCR is not
implemented.

If `create_recipient` returns `information_required`, ask only for the listed
missing or invalid fields and retry with the same request ID only if the payload
is unchanged. Use a new request ID after adding or changing fields.

On success, use either the returned `recipient_id` or canonical
`recipient_fields` in payment tools.

Use `update_recipient` only after the human confirms the replacement fields.
It creates a replacement record; it does not edit or revoke the old record.

For an on-ramp into the shared wallet, call `list_wallets` and use the active
Worldchain wallet address as the crypto recipient. Never ask for its private
key.

## 5. Discover a supported route

Call `list_quote_book_pairs` to inspect live direct on/off-ramp corridors. Use a
matching live pair directly in `estimate_payment`.

For fiat-to-fiat or source-token-to-fiat routing:

1. List relevant on-ramp and off-ramp pairs.
2. Find a common active crypto asset on one supported chain.
3. Call `estimate_payment` with `route.intermediate_asset` set explicitly.
4. Prefer the requested route; otherwise compare executable outcomes including
   fees instead of comparing raw quote-book rates alone.

There is no automatic Core route planner. Do not ask Core to invent a two-hop
route.

Use `browse_quote_book` only for anonymous rough-rate or band discovery. Its
`rate` is raw. Read `fee_pct`, `flat_fee`, and `fee_ccy` together.

Use `get_ramp_quote` only for a direct `on_ramp` or `off_ramp`. Never pass
`on_chain_swap` to it.

## 6. Estimate the complete payment

Call `estimate_payment` for every supported flow:

- direct on-ramp;
- direct off-ramp;
- pure same-chain crypto swap;
- explicit fiat-to-fiat two-hop;
- explicit crypto-token-to-fiat two-hop.

For two hops, pass `route.intermediate_asset`. A concrete final recipient is
required.

Treat the result as an ephemeral review preview:

- it has no estimate ID;
- it is not durable;
- it may create quote intents;
- it never creates approval, settlement, or execution calldata;
- `create_payment` validates the route independently;
- its quote references can expire.

Require `status=estimate_ready`. Read:

- `source_amount`;
- `destination_amount`;
- fee and fee currency for every leg;
- effective request-specific amounts;
- expiry;
- route and intermediate amount;
- recipient validation;
- returned `hops`.

If no executable estimate is available, explain the blocking requirement and
stop or select another live route with the user's approval.

## 7. Apply authorization once

Show one complete summary before creating the payment:

```text
Recipient: [name/rail and sufficient destination details]
You send: [amount and asset]
Recipient receives: [amount and asset]
Fees: [each amount and its currency]
Route: [direct, swap, or source -> intermediate -> destination]
Estimate expires: [time]
Expected duration: [when available]
Material warnings: [only relevant warnings]
```

Do not show a raw rate as the effective customer rate when fees change the
actual source/destination amounts.

Apply the account owner's automatic transaction threshold configured at
`https://app.agentbank.world`:

- below-threshold payments may continue automatically only when the current
  runtime exposes valid standing authorization for the connected agent;
- above-threshold payments require World ID payment authorization;
- if standing authorization is absent or unclear, ask the human to confirm the
  complete payment.

After any material change to recipient, source amount, destination amount, fee,
route, or expiry, reevaluate authorization and reconfirm when required. Never
infer the threshold, its currency conversion, or whether a payment qualifies.

## 8. Create and approve the durable payment

After the applicable explicit or standing authorization is satisfied, call
`create_payment` with:

- a new stable `request_id`;
- `confirmed_by_user=true`;
- the reviewed source, destination, amount mode, and routing preference;
- top-level `intermediate_asset` for two hops;
- the exact `hops` returned by the current estimate.

Do not pass an estimate ID. None exists.

For two-hop payments, preserve the returned structure:

- hop 0 is `on_ramp` or `on_chain_swap`;
- hop 0 has `recipient_ref:{"hop_index":1}`;
- hop 1 is `off_ramp`;
- hop 1 has concrete `recipient_fields`.

When the payment returns `approval_required`:

1. Show `approval.approval_url`, the first-party action page.
2. State that the human must sign in as the payment owner before the page reveals the World ID request.
3. State the approval expiry.
4. Ask the human to approve in World App.
5. Never request, print, reconstruct, or transmit a raw World ID QR, verification URL, or proof.

Payments below the configured platform threshold may return `approval_ready`
with `approval:null`. Do not ask for World ID authorization in that case. Call
`continue_payment` with a new continuation request ID. When the payment returns
`approval_required`, call `get_payment` after the human authorizes and continue
when it returns `approval_ready`.

## 9. Follow the payment instruction

Read `payment_instruction` and `next_action` exactly.

### Two-hop funding invariant

For a linked two-hop payment, the agent acts only on the first/source hop and
then tracks the aggregate:

- API `hop_index:0` is the first upstream `on_ramp` or `on_chain_swap` hop.
- API `hop_index:1` is the downstream `off_ramp` hop.
- Core opens hop 1 first only to obtain its crypto deposit destination.
- Hop 0 is already bound to that destination through
  `recipient_ref:{"hop_index":1}`.
- Funding or executing hop 0 therefore delivers the intermediate crypto
  directly into hop 1. No second wallet transfer is required.

For fiat-to-fiat, ask the human to pay the source on-ramp instruction, then poll
`get_payment`. For crypto-token-to-fiat, execute the source swap instruction;
its output recipient is already the downstream off-ramp deposit, then poll
`get_payment`.

Never separately fund hop 1 after hop 0 is paid or executed. If a later payment
view exposes hop 1's crypto-deposit details while the linked payment is still
processing, treat them as internal routing/tracking context, not a new funding
request. A second manual transfer would duplicate funding. This rule overrides
the generic crypto-deposit instructions below for linked two-hop payments.

If `next_action.action_url` or `payment_instruction.presentation_url` exists,
use it for human-executed fiat funding such as bank transfer, QR, payment-link,
or mobile money. For `crypto_deposit`, treat the page as optional presentation;
execute the exact instruction through the shared Privy wallet in chat after
confirmation.

### Fiat funding

For on-ramp bank, QR, payment-link, or mobile-money instructions:

1. Show the action URL and exact amount/expiry.
2. Ask the human to complete the fiat payment.
3. Do not call `execute_payment_instruction` for fiat funding.
4. Poll `get_payment` after the human pays.

### Crypto deposit funding

For a direct one-hop off-ramp crypto deposit:

1. Show the exact chain, asset, amount, full destination, memo/reference, and
   expiry.
2. Do not ask the human to open a frontend payment page. The agent controls the
   onboarding-bound Privy wallet and should execute the current instruction in
   chat after the applicable explicit or standing authorization.
3. Call `get_wallet_balances` for the required asset.
4. Obtain explicit confirmation before sending unless the runtime confirms that
   the account's standing authorization permits this below-threshold execution.
5. Call `execute_payment_instruction` with the payment ID, current instruction
   ID, stable request ID, and `confirmed_by_user=true`.
6. If execution is still pending, call `execute_payment_instruction` again with
   the same request ID. Do not construct ERC-20 calldata or submit a replacement
   wallet transaction.
7. Continue polling `get_payment`; Core tracks the deposit independently.

Never send a rounded amount when the instruction requires an exact amount.

Core requests gas sponsorship only when the bound wallet is AgentKit verified.
Otherwise `execute_payment_instruction` automatically submits through the same
Privy EOA without sponsorship. Check the native balance along with the payment
asset balance because the wallet must pay gas in that mode.

### Pure swap or swap hop

For `payment_instruction.type=swap_execution`:

1. Read only the fresh execution returned by `continue_payment`/`get_payment`.
2. Show the authorized source ceiling, destination amount, asset, chain, and
   recipient. Do not expose or ask the human to validate raw calldata.
3. Call `execute_payment_instruction` with the payment ID, current instruction
   ID, stable request ID, and `confirmed_by_user=true`.
4. Core checks the current allowance and submits an exact approval only when
   needed, immediately rechecks allowance before the swap, executes the pinned
   swap, and submits the final hash for verification.
5. If execution is pending, call `execute_payment_instruction` again with the
   same request ID. Never rotate the request ID after an ambiguous submission.
6. Poll `get_payment` while Core verifies the transaction.

Do not invent calldata, allowance targets, token contracts, or amount ceilings.
For exact destination, never spend more than the confirmed source ceiling.

## 10. Track the payment

Use `get_payment` as the authoritative state. Follow
`next_action.poll_after_seconds` when present.

Current statuses are `approval_required`, `approval_ready`, `funding_required`,
`funding_detecting`, `need_review`, `recipient_correction_required`,
`completed`, `cancelled`, and `failed`.

Do not report a two-hop payment complete until the aggregate is `completed`.
One completed hop is not complete payment delivery.

When a hop has `receipt.crypto_tx_hash`, report it as that hop's chain
reference. The receipt also contains the settled input/output assets and
amounts plus `completed_at`. A receipt is useful evidence, but `get_payment`
remains the authority for aggregate payment completion.

When status is `need_review`, the provider is still reviewing KYC or another
temporary settlement condition. Do not call `continue_payment` or
`correct_payment_recipient`. Wait for `next_action.poll_after_seconds`, then
call `get_payment` again. This state does not mean the saved recipient is
invalid.

Use `list_payments` to find historical payments for the current installation.
When the user refers to an earlier payment ambiguously, compare source,
destination, amount, status, and time, then ask which one they mean if multiple
records fit.

## 11. Recover safely

Always call `get_payment` before retrying a mutation.

### Recipient correction

When status is `recipient_correction_required`:

1. Read the failure and current destination context.
2. Ask for corrected fields.
3. Show the change and obtain confirmation.
4. Call `correct_payment_recipient` with a new request ID and
   `confirmed_by_user=true`.
5. Resume polling.

### Cancellation

Call `cancel_payment` only after the human confirms and only before funds move.
Cancellation also closes a payment whose hops have not opened yet. There is no
separate approval-cancel tool, so an associated World ID challenge may remain
visible until it expires, but completing it cannot resume the cancelled payment.

If `funds_moved=true`, do not promise cancellation, duplicate funding, or create
a replacement payment automatically.

### Failure

Read `failure.code`, `failure.stage`, `failure.message`, `retryable`, and
`funds_moved`.

- If funds did not move and the route/approval expired, create a fresh estimate
  and obtain fresh confirmation before a new payment.
- If funds moved, explain the state and continue tracking or escalate. Current
  Core has no automatic partial two-hop recovery action.
- Never convert a failed payment into success based on a wallet receipt alone.

## 12. Tool selection reference

```text
Setup: begin_agent_onboarding, wait_for_agent_onboarding, get_installation_status, revoke_agent
Identity: whoami, get_account_status, check_my_scopes, check_verification_status, do_kyc, get_verification_guidance
Discovery: list_currencies, list_quote_book_pairs, browse_quote_book, get_ramp_quote, estimate_payment
Payments: create_payment, continue_payment, execute_payment_instruction, get_payment, list_payments, cancel_payment, correct_payment_recipient
Recipients: list_recipients, get_recipient, create_recipient, update_recipient
Wallet: list_wallets, verify_agent_kit, get_wallet_balances, get_token_allowance, approve_token, get_transaction_receipt
Guidance: get_instructions
```
