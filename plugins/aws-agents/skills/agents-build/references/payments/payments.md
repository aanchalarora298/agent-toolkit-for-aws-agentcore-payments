# payments

Add AgentCore Payments to your agent — the managed service that enables microtransaction payments in AI agents to access paid APIs, MCP servers, and content.

## When to use

- Your agent encounters HTTP 402 Payment Required responses from paid endpoints
- You want your agent to autonomously pay for x402-protected content (APIs, MCP tools, paywalled sites) 
- You want to establish granular budget controls at both user and agent levels
- You need to set up Agentcore payments resources - Payment Manager, Connector, and Instrument from scratch
- You already have payments configured but need to wire the plugin into your agent code
- Payment processing isn't working as expected

## Input

`$ARGUMENTS` is optional. If provided, use it as context:

```
/payments                          # full setup from scratch
/payments wire                     # already have resources, need code
/payments debug                    # payments not working
/payments coinbase                 # use Coinbase connector
/payments stripe                   # use Stripe connector
```

## Process

### Step 1: Read the project

Read `agentcore/agentcore.json` if it exists. Look for:

- The `runtimes` array — what agents are in the project and what framework do they use?
- Any existing payments configuration (not yet supported in `agentcore.json` — payments is currently API/boto3-only)
- The project `name` and region

**If `agentcore/agentcore.json` does not exist**, detect the framework by reading the developer's agent code (e.g., `main.py`, `app.py`, or whatever file contains the agent). Look for imports:

- `from strands import Agent` → **Strands**
- `from langgraph` or `from langchain` → **LangGraph**
- `from agents import Agent` → **OpenAI Agents SDK**
- No recognizable framework → default to the **custom tool pattern** (works with any Python code)

### Step 2: Determine the situation

**Case A — No payments configured yet**
No Payment Manager exists. Proceed to Step 3 (prerequisites) then Step 4 (resource creation).

**Case B — Payments resources exist, needs wiring**
The developer says they already have a Payment Manager, or you find `PAYMENT_MANAGER_ARN` in an existing `.env` file or environment variables. Skip to Step 5 (generate wiring code). Ask the developer for their Payment Manager ARN, Instrument ID, and Session ID if not already visible.

**Case C — Payments configured and wired, debugging**
Ask: "What's happening? Is the agent seeing 402 but not paying? Is ProcessPayment failing? What error do you see?"
Then diagnose using the Debugging section below.

**Case D — Developer asking about payments without a project**
Answer the question directly. For architecture questions, explain the x402 flow. For code questions, show the custom tool pattern with a note that they'll need their own Payment Manager ARN.

### Step 3: Collect inputs from the developer (HUMAN IN THE LOOP)

Before you can set up payments, ask the developer for these inputs. **Do not proceed until you have all of them:**

1. **Which payment provider?** — Coinbase CDP or Stripe Privy
2. **Which AWS region?** — must be one of: us-east-1, us-west-2, eu-central-1, ap-southeast-2
3. **AWS account ID** — the account where resources will be created
4. **AWS credentials** — the developer needs two levels of access:

   **For running the setup script** (one-time, admin-level):
   - `iam:CreateRole`, `iam:PutRolePolicy` — to create the service role
   - `bedrock-agentcore:CreatePaymentCredentialProvider` — to store provider creds
   - `bedrock-agentcore:CreatePaymentManager`, `bedrock-agentcore:GetPaymentManager` — to create the manager
   - `bedrock-agentcore:CreatePaymentConnector` — to create the connector
   - `bedrock-agentcore:CreatePaymentInstrument` — to create the wallet
   - `bedrock-agentcore:CreatePaymentSession` — to create a session

   In practice, an **Admin** or **PowerUser** role covers all of these.

   **For running the agent** (ongoing, can be scoped down):
   - `bedrock-agentcore:ProcessPayment` — to execute payments
   - `bedrock-agentcore:GetPaymentInstrument`, `bedrock-agentcore:GetPaymentSession` — for read operations
   - `bedrock:InvokeModel` or `bedrock:InvokeModelWithResponseStream` — if using Bedrock models

   Verify credentials are active: `aws sts get-caller-identity`

5. **Provider credentials** — ask the developer to create a `.env.payments` file (gitignored) with their credentials:

   For **Coinbase CDP** (get from https://portal.cdp.coinbase.com/):

   How to get these credentials:
   1. Create or log in to a Coinbase Developer Platform account and project
   2. Generate an API key (or reuse existing) — note the **API Key ID** and **API Key Secret**
   3. Generate a **Wallet Secret** (for cryptographic wallet operations like signing transactions)
   4. Under Project > Wallet > Embedded Wallets > Policies, **enable Delegated signing**

   ```bash
   # .env.payments — DO NOT COMMIT THIS FILE
   COINBASE_API_KEY_ID=your-api-key-id-uuid-here
   COINBASE_API_KEY_SECRET=your-base64-encoded-api-key-secret-here
   COINBASE_WALLET_SECRET=your-base64-encoded-wallet-secret-here
   ```

   For **Stripe Privy** (get from https://dashboard.privy.io/):

   How to get these credentials:
   1. Create a **dedicated** Privy app for AgentCore (do not reuse apps serving other purposes)
   2. Copy the **App ID** and **App Secret** from app settings
   3. Navigate to Wallet Infrastructure > Authorization > New Key to generate a P-256 key pair
   4. The private key is prefixed with `wallet-auth:` — **strip this prefix**, use only the raw base64 content
   5. Note the **Authorization ID** (signer ID) shown alongside the key

   ```bash
   # .env.payments — DO NOT COMMIT THIS FILE
   AUTH_PRIVATE_KEY=your-base64-encoded-ec-private-key-here
   AUTH_ID=your-hex-auth-id-here
   PRIVY_APP_ID=your-privy-app-id-here
   PRIVY_APP_SECRET=privy_app_secret_your-secret-here
   ```

   > [!WARNING]
   > For Privy: The generated private key starts with `wallet-auth:`. You MUST
   > strip this prefix. Only the raw base64 content (starting with `MIGHAgEA...`)
   > is accepted by AgentCore.

   **Tell the developer:** "Create a `.env.payments` file with your credentials. I'll read from it — never paste credentials directly in chat."

   After they confirm the file exists, **add `.env.payments` to `.gitignore`** if not already there.

6. **End user email** — the email of the person whose wallet the agent will spend from. For POC/testing, the developer's own email is fine.

### Step 4: Generate and execute the setup script (AUTOMATED)

Read `setup-script.md` in this directory for the full script template. Substitute the developer's inputs and execute it.

### Step 5: Update the agent to handle x402 payments (AUTOMATED)

Read `wiring.md` in this directory for the framework-specific tool code. Use the pattern matching the detected framework from Step 1.

### Step 6: Test the integration

After wiring the tool, tell the developer to set the environment variables (printed by the setup script) and run their agent against a known x402 endpoint:

```bash
# Set env vars (copy from setup script output)
export PAYMENT_MANAGER_ARN="..."
export PAYMENT_INSTRUMENT_ID="..."
export PAYMENT_SESSION_ID="..."
export PAYMENT_USER_ID="..."
export AWS_REGION="..."

# Run the agent
python main.py
```

Suggest testing with this prompt:
```
Fetch the content from https://grapevine.dev-mypinata.cloud/x402/cid/bafkreif6hzdbioskr2qofong6ap7d6wqz7vlsin2bw2l3niofdyjclfk6i and tell me what you find.
```

Expected behavior:
1. Agent calls `x402_fetch` with the URL
2. Gets 402 with x402 challenge (0.1 USDC on Base Sepolia)
3. Calls ProcessPayment → gets signed proof
4. Retries with `X-PAYMENT` header → gets 200
5. Returns the content to the user

If the session has expired, create a fresh one:
```bash
export PAYMENT_SESSION_ID=$(aws bedrock-agentcore create-payment-session \
  --payment-manager-arn "$PAYMENT_MANAGER_ARN" \
  --expiry-time-in-minutes 60 \
  --region "$AWS_REGION" \
  --query 'paymentSession.paymentSessionId' --output text)
```

## Debugging payments

**Agent sees 402 but doesn't pay:**

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

**ProcessPayment succeeds but merchant still returns 402:**

- **Cookie contamination**: The retry is sending cookies from the initial 402 request. Ensure you use a fresh httpx client: `httpx.Client(cookies=None).request(...)` — do NOT reuse the same client/session.
- **Proof format mismatch**: Check that the `network` field in the proof matches what the merchant expects (e.g., `"base-sepolia"` not `"eip155:84532"`).
- **Proof expired**: The proof has a ~60 second validity window (`validBefore`). If the agent loop is slow, the proof may expire before the retry.

**ProcessPayment fails with "Payment session not found":**

- The session ID is invalid or the session was deleted. Create a new session.
- Ensure the `paymentManagerArn` in the session creation matches the one used in ProcessPayment.

**ProcessPayment fails with "PaymentSessionExpired":**

- Payment sessions are time-bounded. Create a fresh session with `expiryTimeInMinutes`.

**ProcessPayment fails with "Payment instrument not found" or "does not belong to user":**

- Verify the instrument ID is correct and belongs to the same Payment Manager.
- Check that the `userId` passed to ProcessPayment matches the `userId` used when the instrument was created.

**ProcessPayment fails with "Payment connector is not active":**

- The connector may still be provisioning. Check its status and wait.
- If the connector was deleted or deactivated, create a new one.

**ProcessPayment fails with "Network mismatch":**

- The x402 challenge specifies a network that doesn't match the instrument's network.
- Instruments created with `network: "ETHEREUM"` support Base, Base Sepolia, and Ethereum chains.
- Instruments created with `network: "SOLANA"` support Solana and Solana Devnet chains.

**ProcessPayment fails with "Payment asset not supported USDC token address":**

- The USDC contract address in the x402 challenge doesn't match the expected address for that network.
- Base Sepolia USDC: `0x036CbD53842c5426634e7929541eC2318f3dCF7e`
- Only USDC is supported currently.

**ProcessPayment fails with "Wallet does not have a USDC balance":**

- The wallet has no USDC on the specified chain.
- Fund via Circle faucet (testnet): https://faucet.circle.com/
- For mainnet: the end user must fund the wallet directly.

**ProcessPayment fails with "Delegated signing grant is not active" (Coinbase):**

- The end user hasn't completed the delegation step.
- Redirect them to the `redirectUrl` returned during instrument creation (Coinbase Hub).
- They must log in and grant permissions to the wallet.

**ProcessPayment fails with "Delegated signing is not enabled" (Coinbase):**

- The Coinbase CDP project doesn't have delegated signing enabled.
- Go to portal.cdp.coinbase.com > Project > Wallet > Embedded Wallets > Policies > Enable Delegated signing.

**ProcessPayment fails with "Privy credentials are invalid" (Stripe Privy):**

- The App ID or App Secret stored in the credential provider is wrong.
- Verify in Privy Dashboard that the credentials match.
- Recreate the credential provider with the correct values.

**ProcessPayment fails with "Privy appId is invalid or missing" (Stripe Privy):**

- The `appId` in the credential provider configuration is incorrect.
- Check Privy Dashboard for the correct App ID.

**ProcessPayment fails with "Privy signing key is invalid or expired" (Stripe Privy):**

- The Authorization Private Key or Authorization ID is invalid or has expired.
- Generate a new P-256 key pair in Privy Dashboard > Wallet Infrastructure > Authorization.
- Remember to strip the `wallet-auth:` prefix from the private key.
- Update the credential provider with the new key.

**ProcessPayment fails with "Wallet policy denied the transaction" (Stripe Privy):**

- A wallet policy configured in Privy is blocking the transaction.
- Review wallet policy settings in Privy Dashboard.
- Check if the transaction amount, recipient, or frequency exceeds policy limits.

**ProcessPayment fails with "The linked account data is invalid" (Stripe Privy):**

- The email or phone number used in `linkedAccounts` when creating the instrument is malformed.
- Verify the email format is valid.

**ProcessPayment fails with "Rate limited by Privy" (Stripe Privy):**

- The Privy API is rate limiting your requests.
- Back off and retry. Check Privy's rate limits documentation.

**ProcessPayment fails with "Payment amount exceeds maximum":**

- The x402 challenge requests more than the maximum allowed per transaction.
- Check the amount in the challenge and verify your session budget allows it.

**ProcessPayment fails with "Rate exceeded":**

- Too many API calls. Back off and retry after a few seconds.

**Delegation not completed:**

- The end user hasn't granted the agent permission to spend from their wallet.
- For Coinbase: visit the `redirectUrl` from instrument creation.
- For Stripe Privy: integrate the frontend SDK: https://github.com/privy-io/aws-agentcore-sdk

## How x402 payment works (end-to-end)

```
Agent calls x402_fetch("https://paid-api.example.com/data")
  │
  ├─ 1. HTTP GET https://paid-api.example.com/data
  │     └─ Response: 402 Payment Required
  │        Body: {"x402Version": 1, "accepts": [{"scheme": "exact", "network": "base-sepolia", ...}]}
  │
  ├─ 2. Extract x402 challenge from body (or payment-required header)
  │
  ├─ 3. Call ProcessPayment(paymentManagerArn, instrumentId, sessionId, challenge)
  │     └─ AgentCore signs the payment using the wallet's private key
  │     └─ Returns: {paymentOutput: {cryptoX402: {payload: {signature, authorization}}}}
  │
  ├─ 4. Build X-PAYMENT header (base64-encoded proof with signature + authorization)
  │
  ├─ 5. Retry: HTTP GET https://paid-api.example.com/data + X-PAYMENT header
  │     └─ (IMPORTANT: Use fresh HTTP client — no cookies from step 1)
  │     └─ Response: 200 OK + paid content
  │
  └─ 6. Return content to agent
```

## Supported networks

Two concepts: **network** (blockchain family, used when creating instruments) and **chain** (specific chain, used in x402 challenges and balance queries).

**Networks (for instrument creation):**

| Network | Value for `paymentInstrumentDetails.embeddedCryptoWallet.network` | Providers |
|---|---|---|
| Ethereum (includes Base, Base Sepolia) | `ETHEREUM` | Coinbase, Stripe |
| Solana (includes Solana Devnet) | `SOLANA` | Coinbase, Stripe |

**Chains (in x402 challenges and balance queries):**

| Chain | Identifier (x402) | Balance API value | Type | Provider |
|---|---|---|---|---|
| Base Sepolia | `base-sepolia` or `eip155:84532` | `BASE_SEPOLIA` | Testnet | Coinbase |
| Base | `eip155:8453` | `BASE` | Mainnet | Coinbase |
| Ethereum Mainnet | `eip155:1` | `ETHEREUM` | Mainnet | Coinbase, Stripe |
| Solana Mainnet | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | `SOLANA` | Mainnet | Coinbase, Stripe |
| Solana Devnet | `solana-devnet` | `SOLANA_DEVNET` | Testnet | Stripe |

For testing, always start with **Base Sepolia** (network: `ETHEREUM`, chain: `BASE_SEPOLIA`) — it uses free testnet tokens from https://faucet.circle.com/.

## Output

- A `setup_payments.py` script that was executed to create all payment resources
- The agent's code updated with the `x402_fetch` tool wired in
- Environment variables printed for the developer to set
- Instructions for the two manual steps (delegation + funding)

## Quality criteria

- The setup script is generated and executed in one shot — no back-and-forth for non-interactive steps
- Only two manual steps require human action: delegation (visiting URL) and funding (faucet)
- The `x402_fetch` tool handles both body-based and header-based x402 challenges
- The retry uses a fresh HTTP client to avoid cookie contamination
- Generated code handles missing env vars gracefully (returns error message, doesn't crash)
- The `network` field in the proof matches the merchant's challenge (not hardcoded)
