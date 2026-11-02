---
name: quartermaster-esim
description: Buy prepaid travel eSIM data plans for a traveler from Quartermaster, an agent-only marketplace reached over MCP. Use when the user (your principal) needs mobile data abroad, asks for an eSIM for a trip, wants to top up their Quartermaster wallet, or wants to check, refund or dispute an eSIM order bought from Quartermaster. Handles the prepaid wallet and top-up links, plan search by country or region, policy checks, spending mandate limits, purchase with idempotency, delivery of the QR activation code, and refunds.
---

# Quartermaster: eSIM purchasing

Quartermaster sells eSIM data plans to AI agents only, through an MCP server named `quartermaster`. You act for a principal (the person paying). Purchases are paid from the principal's prepaid wallet in USD; the principal tops it up by paying a link you request (PayPlus hosted page, charged in ILS). The store enforces a spending mandate your principal grants on top of the wallet balance, and every decision it makes comes back as `{allowed, reason_code, human_readable}`.

## Setup

Install this skill for Claude Code at `~/.claude/skills/quartermaster-esim/SKILL.md` (or `.claude/skills/quartermaster-esim/SKILL.md` in a repository).

Add the MCP server to the host configuration over Streamable HTTP at `https://api.quartermasterhq.ai/mcp` (or `<store-url>/mcp` for another deployment) with headers `X-Agent-Id`, `X-Principal-Id` and `X-Agent-Key` (Claude Code: `claude mcp add --transport http quartermaster https://api.quartermasterhq.ai/mcp`; `<store-url>/docs` has the exact commands for the deployed store).

If you have no `agent_key` yet, call `register_agent` once. Ask the principal for their limits first (per transaction, per day, per month, in minor units, and an expiry date no more than 365 days out). Save the returned `agent_key` somewhere persistent; it is shown once.

**Principal key.** For a new `principal_id`, `register_agent` also returns `principal_key`, once. Give it to the principal and tell them to keep it: it proves they own the wallet, and any further agent registered for them must send it (`principal_key` argument), otherwise `PRINCIPAL_KEY_REQUIRED`. If the principal already uses the store, ask them for their `principal_key` before registering. If it may have leaked, call `rotate_principal_key` with the current key and hand over the new one.

## Wallet

1. **Check funds.** `get_wallet` returns `balances[0].available_minor` (USD cents). The wallet is shared by all agents of the same principal.
2. **Top up if short.** `get_topup_link` with `amount_minor` (USD by default, or `currency: "ILS"` for an exact charge). Tell the principal, in plain words: the amount in ILS they will pay (`charge`), roughly what lands in the wallet (`expected_credit`), and the `url`. The link expires in 30 minutes. Never ask the principal for card details; they enter them only on the payment page.
3. **Wait.** Poll `get_wallet` every 10 to 30 seconds until the balance arrives. If the top-up shows `status: "failed"`, the card was declined; offer a new link.
4. The credit is converted at the rate in force when the payment is confirmed, so it can differ from `expected_credit` by a few cents.

Top up a little above the plan price if the principal expects more purchases soon; each link costs the principal a separate card payment. Limits: 5.00 to 500.00 USD per link, wallet balance at most 1,000.00 USD.

## Workflow

1. **Clarify the need.** Destination countries, trip length in days, expected data use. Rough guide: light use (maps, messaging) 0.5 GB per day; heavy use (video, hotspot) 1.5 GB or more per day. Check the traveler's phone supports eSIM and is carrier-unlocked.
2. **Check terms.** Call `get_policies` once per session. Note `store_terms.quote_ttl_seconds` and the `refund` policy's rules.
3. **Search.** `search_plans` with `country` (ISO code like `JP`) or `region`, `min_data_gb`, `min_validity_days`. Prefer an exact fit; consider a regional plan when the trip spans several countries. Results are sorted by final price.
4. **Read the product before choosing.** Look at `refund_rule` (`none` means non-refundable), `constraints.kyc_required`, `constraints.hotspot_allowed`, `constraints.activation_window_days`, and `countries` for regional plans.
5. **Quote.** `get_quote` with the `product_id`. If `mandate_precheck.allowed` is false, do not purchase; tell the principal the `human_readable` reason and offer a cheaper option. If `wallet_precheck.allowed` is false, top up first (see Wallet).
6. **Confirm with the principal** when the price is above what they told you to spend without asking, or when the plan is non-refundable.
7. **Purchase.** `purchase` with `quote_id` and a new UUID as `idempotency_key`. The amount is debited from the wallet only after the supplier delivers. Keep the key until the call returns; if the connection drops, call `purchase` again with the same key.
8. **Deliver to the traveler.** From `order.delivery`: for `qr`, render `lpa_string` as a QR code (or give it as the manual activation code); for `activation_link`, send the link. Include `instructions`. Tell them to install before departure but turn on data roaming for the eSIM line only on arrival.
9. **Keep the receipt** and the `order_id`.

## Failures

- `SUPPLIER_FAILURE` with `retryable: true`: nothing was debited from the wallet. Get a new quote and purchase with a new idempotency key, at most 3 more attempts with a short pause between them.
- Mandate errors (`MANDATE_*`): do not try to work around them by splitting purchases. Report the reason to the principal.
- `RATE_LIMITED`: wait `retry_after_seconds` before the next call.
- `WALLET_INSUFFICIENT_FUNDS`: the message already contains a top-up link (`error.decision.effects.topup_url`). Pass it to the principal, wait for `get_wallet` to show the funds, then purchase again with a new idempotency key (same quote if it has not expired).

## Refunds and problems

Use `request_refund` with the `order_id` and a reason:

- Trip cancelled or plan not needed: `unused` or `changed_plans`. Allowed only if the eSIM was never activated and within the plan's refund window.
- No eSIM payload received: `not_delivered`.
- Installed but no connection: `not_working` (fully refunded if never activated; partly refunded if activated with very low usage; otherwise reviewed by a human).
- Bought the same plan twice by mistake: `duplicate_purchase`.

Refunds are credited to the principal's wallet, not the card. Tell the principal the amount is available for the next purchase.

If the code will not install (invalid or already-used code), first call `regenerate_activation` with the `order_id` and give the traveler the new `delivery`; dispute only if that fails or is unsupported.

Check `order_status` first: it shows whether the eSIM is activated and how much data was used, which determines the outcome. If the result is `outcome: "escalate"`, tell the principal a human will review within the SLA in the store terms.

## Never

- Never invent a product id; always take it from `search_plans`.
- Never reuse an idempotency key for a different purchase.
- Never share `agent_key` with the traveler or in logs. Share `principal_key` only with the principal.
- Never ask for, accept or relay card numbers. The payment page is the only place card data goes.
- Never request a top-up the principal has not agreed to.
