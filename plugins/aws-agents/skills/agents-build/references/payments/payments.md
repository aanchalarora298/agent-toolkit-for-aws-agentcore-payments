# AgentCore Payments

## Overview

Add AgentCore Payments to your agent — the managed service that enables microtransaction payments in AI agents to access paid APIs, MCP servers, and content via the x402 protocol.

The AWS MCP server is recommended for executing AWS commands (sandboxed execution, audit logging, observability), but is not required. If the MCP server is not available, use AWS CLI or boto3 scripts instead.

## When to Use

- Your agent encounters HTTP 402 Payment Required responses from paid endpoints
- You want your agent to autonomously pay for x402-protected content (APIs, MCP tools, paywalled sites)
- You want to establish granular budget controls at user and agent levels
- You need to set up AgentCore Payments resources from scratch
- You already have payments configured but need to wire the plugin into agent code
- Payment processing is not working as expected

Do NOT use for:

- General agent scaffolding or project creation
- Connecting to external APIs via Gateway (OpenAPI specs, Lambda, MCP servers)
- Agent deployment or infrastructure
- Non-payment related agent capabilities (memory, VPC, multi-agent)

## Process

### Step 1: Read the project context

Read the agent's entrypoint file (e.g., `main.py`, `app.py`). Detect the framework:

- `from strands import Agent` → **Strands**
- `from langgraph` or `from langchain` → **LangGraph**
- `from agents import Agent` → **OpenAI Agents SDK**
- No recognizable framework → default to the **custom tool pattern**

### Step 2: Determine the situation

**Case A — No payments configured yet**
No Payment Manager exists. Proceed to Step 3 (prerequisites) then Step 4 (resource creation).

**Case B — Payments resources exist, needs wiring**
The developer already has a Payment Manager. Skip to Step 5 (generate wiring code). Ask for their Payment Manager ARN, Instrument ID, and Session ID.

**Case C — Payments configured and wired, debugging**
Ask: "What's happening? Is the agent seeing 402 but not paying? Is ProcessPayment failing? What error do you see?"
Then diagnose using the Debugging section below.

**Case D — Developer asking about payments without a project**
Answer directly. For architecture questions, explain the x402 flow. For code questions, show the custom tool pattern.

### Step 3: Collect inputs from the developer

Before setting up payments, collect these inputs:

1. **Which payment provider?** — Coinbase CDP or Stripe Privy
2. **Which AWS region?** — must be one of: us-east-1, us-west-2, eu-central-1, ap-southeast-2
3. **AWS account ID** — the account where resources will be created
4. **End user email** — the email of the person whose wallet the agent will spend from

Once you have answers 1-4, show the provider-specific `.env.payments` template and ask the developer to create the file and run `source .env.payments`:

   For **Coinbase CDP** (get credentials from https://portal.cdp.coinbase.com/):
   ```bash
   # .env.payments — DO NOT COMMIT THIS FILE
   COINBASE_API_KEY_ID=your-api-key-id-uuid-here
   COINBASE_API_KEY_SECRET=your-base64-encoded-api-key-secret-here
   COINBASE_WALLET_SECRET=your-base64-encoded-wallet-secret-here
   ```

   For **Stripe Privy** (get credentials from https://dashboard.privy.io/):
   ```bash
   # .env.payments — DO NOT COMMIT THIS FILE
   AUTH_PRIVATE_KEY=your-base64-encoded-ec-private-key-here
   AUTH_ID=your-hex-auth-id-here
   PRIVY_APP_ID=your-privy-app-id-here
   PRIVY_APP_SECRET=privy_app_secret_your-secret-here
   ```

After they confirm the file exists and have run `source .env.payments`, add `.env.payments` to `.gitignore`.

> **Security:** Do NOT paste credentials directly in chat or ask the agent to read
> the `.env.payments` file. Instead, run `source .env.payments` in your terminal
> to expose the values as environment variables locally. The setup script reads
> from environment variables, not the file directly.

> **Production:** If needed to be stored outside of AgentCore Identity ever, 
> store credentials in AWS Secrets Manager or SSM Parameter Store
> (SecureString) and retrieve them at runtime. The `.env.payments` file is for
> local development only.

### Step 4: Generate and execute the setup script

Read [setup-script.md](setup-script.md) for the full script template. Substitute the developer's inputs and execute it.

The script creates:
1. Payment Credential Provider (stores provider creds in AgentCore Identity)
2. IAM execution role with trust policy and permissions
3. Payment Manager (waits for READY status)
4. Payment Connector
5. Payment Instrument (wallet)
6. Payment Session

### Step 5: Wire the x402 tool into the agent

Read [wiring.md](wiring.md) for framework-specific tool code. Use the pattern matching the detected framework from Step 1.

The `x402_fetch` tool:
1. Makes an HTTP request to the target URL
2. If 402, extracts the x402 challenge from body or `payment-required` header
3. Calls `ProcessPayment` to get a signed payment proof
4. Retries with the `X-PAYMENT` header (fresh HTTP client to avoid cookie contamination)
5. Returns the paid content

### Step 6: Test the integration

Set environment variables (printed by setup script) and run the agent:

```bash
export PAYMENT_MANAGER_ARN="..."
export PAYMENT_INSTRUMENT_ID="..."
export PAYMENT_SESSION_ID="..."
export PAYMENT_USER_ID="..."
export AWS_REGION="..."
```

Test with:
```
Fetch the content from https://sandbox.node4all.com/v1/x402-test and tell me what you find.
```

Expected: Agent calls x402_fetch → gets 402 → ProcessPayment → retries with proof → returns content.

## Security Considerations

- **Credential rotation**: Rotate payment provider credentials periodically. Recreate the credential provider with updated values.
- **Budget/spend limits**: Use Payment Session `expiryTimeInMinutes` and per-session budget controls to prevent runaway payments.
- **Audit logging**: Verify CloudTrail is logging all `bedrock-agentcore` API calls, especially `ProcessPayment`. For production, set up a CloudWatch alarm for failed payment attempts as a potential abuse indicator.
- **SSRF mitigation**: The `x402_fetch` tool enforces HTTPS-only and blocks private IP ranges to prevent fetching internal endpoints.
- **Least privilege**: The IAM service role should only have the minimum permissions required (token-vault, workload-identity, secrets access).
- **Session expiry**: Keep payment sessions short-lived (60 minutes or less). Create fresh sessions per user interaction rather than reusing long-lived ones.
- **Encryption in transit**: All payment requests must use HTTPS. The `x402_fetch` tool rejects non-HTTPS URLs.

For comprehensive security guidance, see the [AgentCore Security documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/security.html).

## How x402 Payment Works (End-to-End)

```
Agent calls x402_fetch("https://paid-api.example.com/data")
  │
  ├─ 1. HTTP GET → 402 Payment Required
  │     Body: {"x402Version": 1, "accepts": [{"scheme": "exact", "network": "base-sepolia", ...}]}
  │
  ├─ 2. Extract x402 challenge
  │
  ├─ 3. ProcessPayment(paymentManagerArn, instrumentId, sessionId, challenge)
  │     → Returns signed proof (signature + authorization)
  │
  ├─ 4. Build X-PAYMENT header (base64-encoded proof)
  │
  ├─ 5. Retry with X-PAYMENT header (fresh HTTP client, no cookies)
  │     → 200 OK + paid content
  │
  └─ 6. Return content to agent
```

## Supported Networks

| Network | Instrument Value | Chains | Providers |
|---|---|---|---|
| Ethereum | `ETHEREUM` | Base Sepolia, Base, Ethereum Mainnet | Coinbase, Stripe |
| Solana | `SOLANA` | Solana Mainnet, Solana Devnet | Coinbase, Stripe |

For testing, start with **Base Sepolia** — free testnet tokens from https://faucet.circle.com/.

## Debugging payments

**Agent sees 402 but does not pay:**

1. Verify `PAYMENT_MANAGER_ARN` env var is set and not None
2. Check that the agent is using `x402_fetch` tool (not a generic `http_request`)
3. Verify the x402 challenge is present in either the response body (`x402Version` + `accepts` fields) or the `payment-required` header

**ProcessPayment fails with "Failed to obtain resource payment token":**

- The IAM service role is missing permissions. Ensure it has `GetResourcePaymentToken` on the token-vault and `secretsmanager:GetSecretValue` on the secrets.
- Wait 15+ seconds after creating the role before calling ProcessPayment (IAM propagation).

**ProcessPayment fails with "Failed to obtain workload access token":**

- The service role is missing `GetWorkloadAccessToken` permission on the workload-identity-directory resources.

**ProcessPayment fails with "Failed to assume payment execution role":**

- The service role's trust policy is incorrect. Ensure it trusts `bedrock-agentcore.amazonaws.com` with the correct `aws:SourceAccount` condition.
- Verify the role ARN passed to the Payment Manager matches the actual role.

**ProcessPayment fails with "Payment session not found":**

- The session ID is invalid or the session was deleted. Create a new session.
- Ensure the `paymentManagerArn` in the session creation matches the one used in ProcessPayment.

**ProcessPayment fails with "PaymentSessionExpired":**

- Payment sessions are time-bounded. Create a fresh session:
```bash
export PAYMENT_SESSION_ID=$(aws bedrock-agentcore create-payment-session \
  --payment-manager-arn "$PAYMENT_MANAGER_ARN" \
  --user-id "$PAYMENT_USER_ID" \
  --expiry-time-in-minutes 60 \
  --region "$AWS_REGION" \
  --query 'paymentSession.paymentSessionId' --output text)
```

**ProcessPayment fails with "Payment instrument not found" or "does not belong to user":**

- Verify the instrument ID is correct and belongs to the same Payment Manager.
- Check that the `userId` passed to ProcessPayment matches the `userId` used when the instrument was created.

**ProcessPayment fails with "Network mismatch":**

- The x402 challenge specifies a network that does not match the instrument's network.
- Instruments with `network: "ETHEREUM"` support Base, Base Sepolia, and Ethereum chains.
- Instruments with `network: "SOLANA"` support Solana and Solana Devnet chains.

**ProcessPayment fails with "Wallet does not have a USDC balance":**

- Fund via Circle faucet (testnet): https://faucet.circle.com/
- For mainnet: the end user must fund the wallet directly.

**Payment succeeds but merchant still returns 402:**

- **Cookie contamination**: The retry is sending cookies from the initial 402 request. Ensure you use a fresh httpx client: `httpx.Client(cookies=None).request(...)` — do NOT reuse the same client/session.
- **Proof format mismatch**: Check that the `network` field in the proof matches what the merchant expects (e.g., `"base-sepolia"` not `"eip155:84532"`).
- **Proof expired**: The proof has a ~60 second validity window (`validBefore`). If the agent loop is slow, the proof may expire before the retry.

**Coinbase: "Delegated signing grant is not active":**

- The end user has not completed the delegation step. Redirect them to the `redirectUrl` returned during instrument creation. They must log in and grant permissions to the wallet.

**Coinbase: "Delegated signing is not enabled":**

- Go to portal.cdp.coinbase.com > Project > Wallet > Embedded Wallets > Policies > Enable Delegated signing.

**Stripe Privy: "Privy signing key is invalid or expired":**

- The Authorization Private Key or Authorization ID is invalid. Generate a new P-256 key pair in Privy Dashboard > Wallet Infrastructure > Authorization. Strip the `wallet-auth:` prefix from the private key. Update the credential provider.

**Stripe Privy: "Wallet policy denied the transaction":**

- A wallet policy in Privy is blocking the transaction. Review wallet policy settings in Privy Dashboard.
