# Quartermaster eSIM

**A store built for AI agents to shop in, on behalf of the people they work for.** Quartermaster sells prepaid travel eSIM data plans to AI agents over MCP. Your agent searches plans by country, checks the store's machine-readable policies, gets a quote, and buys within limits its human set in advance, then hands back the QR / LPA activation code.

**The human stays in control.** A person registers the agent with a spending mandate (per purchase, per day, per month, expiry) and funds a prepaid wallet themselves through a hosted payment page. The agent can never see card details, can never spend beyond the mandate or the wallet balance, and every purchase produces a signed receipt the human can verify. Within those limits the agent shops on its own: there is no storefront for people, because people are not the ones browsing.

- MCP endpoint (Streamable HTTP): `https://api.quartermasterhq.ai/mcp`
- Docs for agents: https://api.quartermasterhq.ai/docs
- Discovery: [`/.well-known/mcp.json`](https://api.quartermasterhq.ai/.well-known/mcp.json), [`/llms.txt`](https://api.quartermasterhq.ai/llms.txt), [`/policies`](https://api.quartermasterhq.ai/policies)
- Claude skill: [`skills/quartermaster-esim/SKILL.md`](skills/quartermaster-esim/SKILL.md)

> Status: **public sandbox**. Search, quotes, wallet and purchase flows are live against a sandbox supplier; real eSIM fulfilment is being connected. Sandbox purchases return test activation codes.

## Connect

**Claude Code**
```sh
claude mcp add --transport http quartermaster https://api.quartermasterhq.ai/mcp
```

**Claude Desktop / other stdio clients** (via `mcp-remote`)
```json
{
  "mcpServers": {
    "quartermaster": { "command": "npx", "args": ["-y", "mcp-remote", "https://api.quartermasterhq.ai/mcp"] }
  }
}
```

**Raw HTTP**
```sh
curl -s https://api.quartermasterhq.ai/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

## Tools

| Tool | What it does |
| --- | --- |
| `register_agent` | Register an agent for a principal with a spending mandate (per transaction, day, month, expiry). Returns `agent_key` and, for a new principal, `principal_key`. |
| `get_policies` | Machine-readable refund, dispute, mandate and wallet rules, so an agent can reason about terms before buying. |
| `search_plans` | Search eSIM plans by country, region, data and validity. |
| `get_quote` | Lock a price for a plan. |
| `purchase` | Buy with an idempotency key; enforces mandate and wallet; returns activation code and a signed receipt. |
| `get_wallet` / `get_topup_link` | Principal's prepaid wallet; hosted top-up link for the human to pay. |
| `order_status`, `regenerate_activation`, `request_refund`, `verify_receipt` | After-sale: status, new activation code, refunds and disputes by policy, receipt verification. |
| `rotate_principal_key` | Rotate the principal's ownership key. |

Every decision comes back as `{allowed, reason_code, human_readable}`; refunds and disputes are resolved by written rules, not by a support desk.

## Why agent-only

Other eSIM MCP servers help an agent compare plans. Quartermaster lets it finish the job: buy within a signed mandate, deliver the activation code, and keep a verifiable receipt.

Agents don't need a homepage. They need complete structured data, a stable API, explicit authority from their principal, and terms they can evaluate before acting. Quartermaster is designed around exactly that: one catalog behind every protocol, policies as data, and a hash-chained ledger with signed receipts for every transaction.

## Contact

Suppliers and agent builders: dvirmimon123@gmail.com
