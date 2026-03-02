# DEFI-Tradingview-BOT
Trade ETH, BTC, SOLANA from Tradingview Signals to your own Tradingview endpoint and personal swap. 
# TradingView Webhook Bot

Comprehensive retail-facing guide for the TradingView webhook module in this project.

## Product Summary

The TradingView Webhook Bot is a server-side execution engine that converts TradingView alerts into Solana swap transactions.

Core value proposition for retail users:

- simple webhook integration from TradingView
- shared-secret request protection
- automated buy/sell execution for `SOL`, `WETH`, and `WBTC`
- configurable risk controls (cooldown, default buy size, SOL reserve)
- in-app activity terminal for monitoring

## How It Is Designed

High-level flow:

1. TradingView sends `POST /api/tradingview/webhook`.
2. Server normalizes payload and parses nested JSON (`message`, `payload`, `strategy`) when present.
3. Server validates shared secret.
4. Server resolves action (`buy` or `sell`) and asset (`SOL`, `WETH`, `WBTC`).
5. Server resolves size rules.
6. Buy uses payload USD if present, otherwise global default USD.
7. Sell requires explicit size (`sellAll`, `percent`, or `amount`).
8. Server executes swap through configured route (`Jito`, `Nozomi`, or RPC).
9. Response is returned and activity is logged in the TradingView terminal.

## What You Need

### Required

- Node.js and npm
- Project dependencies installed (`npm ci` or `npm install`)
- `.env` file with:
- `PRIVATE_KEY` (server trading wallet)
- `RPC_URL` (Solana RPC endpoint)
- TradingView account with webhook alerts enabled
- A webhook URL TradingView can reach

### Wallet Funding Requirements

- SOL for network fees and optional tips
- USDC for buy-side execution
- Asset balances (SOL/WETH/WBTC) for sell-side execution

### Optional but Recommended

- `TRADINGVIEW_WEBHOOK_SECRET` (bootstrap secret from env)
- Jito and Nozomi connection settings if you use those routes
- Public tunnel/host (for local setups): Cloudflare Tunnel, ngrok, reverse proxy

## How To Run

### Option A: npm

```bash
npm ci
npm run server
```

Default port is `3000` unless overridden by `PORT`.

### Option B: included launcher scripts

- macOS/Linux: `LINK_COPY_TRADER.command` (main app on `3000`)
- Windows: `LINK_COPY_TRADER.bat` (main app on `3002`, plus extra ports)

If using the Windows launcher, webhook URL is usually:

```text
http://localhost:3002/api/tradingview/webhook
```

## How To Use (UI Setup)

Open the app and go to `TradingView Webhook Bot`.

1. Enable the bot.
2. Copy `Endpoint URL`.
3. Set and save `Shared Secret`.
4. Configure `Cooldown (seconds)`, `SOL buffer (USD)`, and `Default buy (USD)`.
5. Choose routing (`Bundle via Jito`, `Fallback Nozomi`).
6. Save settings.

Panel tools:

- `Copy Endpoint`
- `Copy Alert JSON`
- `Test Payload` (preview only, does not execute orders)
- Activity terminal with webhook/fill health chips

## TradingView Alert Setup

In TradingView alert configuration:

- Webhook URL: `https://YOUR_HOST/api/tradingview/webhook`
- Message format: valid JSON
- Include shared secret in the payload

### Buy Alert (minimum)

```json
{
  "secret": "YOUR_SECRET",
  "action": "buy",
  "symbol": "SOL"
}
```

If no buy amount is supplied, bot uses `defaultUsd`.

### Buy Alert (explicit size)

```json
{
  "secret": "YOUR_SECRET",
  "action": "buy",
  "symbol": "WETH",
  "usd": 100
}
```

Accepted buy size keys include:
`usd`, `usdc`, `amountUsd`, `sizeUsd`, `usdSize`, `notional`, `notionalUsd`, `quote`.

### Sell Alert (explicit size required)

Percent:

```json
{
  "secret": "YOUR_SECRET",
  "action": "sell",
  "symbol": "SOL",
  "percent": 50
}
```

Amount:

```json
{
  "secret": "YOUR_SECRET",
  "action": "sell",
  "symbol": "WETH",
  "amount": 0.25
}
```

Full exit:

```json
{
  "secret": "YOUR_SECRET",
  "action": "sell",
  "symbol": "WBTC",
  "sellAll": true
}
```

If sell size is missing, request is rejected intentionally.

## Payload Compatibility Rules

### Action Detection

The bot checks these keys:

- `action`, `signal`, `type`, `side`, `command`, `order`
- `strategy.order_action`, `strategy.order_side`, `strategy.action`

It maps:

- `buy` or `long` to buy
- `sell` or `short` to sell

### Asset Detection

The bot checks:

- `asset`, `symbol`, `ticker`, `pair`, `market`, `strategy.ticker`

Resolved assets:

- `SOL`
- `WETH`
- `WBTC`

Unknown assets default to `SOL`.

### Secret Extraction

Secret can be read from:

- `secret`, `passphrase`, `key`, `token`, `auth`, `authorization`
- same keys inside `strategy.*`

### Nested JSON Merge

If these fields contain JSON strings, they are parsed and merged:

- `message`
- `payload`
- `strategy`

## Execution Rules

### Buy Path

- Applies buy cooldown.
- Applies in-flight buy lock (`Buy alert already in progress` safeguard).
- Uses payload USD if present.
- Falls back to global `defaultUsd`.
- Rejects if resulting buy amount is invalid.

### Sell Path

Sell requests must include one of:

- `sellAll=true`
- `percent > 0`
- `amount`/`units > 0`

Otherwise request is blocked.

### SOL Sell-All Reserve Logic

For SOL `sellAll`, bot subtracts:

- configured SOL reserve (`solReserveUsd`, converted to lamports)
- execution tip reserve (if Jito/Nozomi used)
- fee overhead

This helps prevent complete SOL depletion.

### Routing Priority

- If `useJito=true`, Jito path is preferred.
- If `useJito=false` and `useNozomi=true`, Nozomi path can be used.
- Otherwise standard RPC route is used.

## Security and Safeguards

- Shared-secret validation on every webhook.
- Buy cooldown and buy in-flight lock.
- Invalid/malformed payload rejection.
- Balance checks before execution.
- Explicit-size sell requirement to reduce accidental full liquidation.

## API Endpoints

- `GET /api/tradingview/config`: fetch runtime config
- `POST /api/tradingview/config`: update runtime config
- `POST /api/tradingview/secret`: set/update shared secret
- `POST /api/tradingview/webhook`: receive TradingView alert and execute
- `GET /api/tradingview/logs`: retrieve recent TradingView logs
- `POST /api/tradingview/logs/clear`: clear in-memory TradingView logs

## Example Webhook Response

Success:

```json
{
  "ok": true,
  "action": "buy",
  "asset": "SOL",
  "mode": "rpc",
  "sig": "5x...",
  "summary": {
    "action": "BUY",
    "input": "25.00",
    "expectedOut": "0.3123",
    "route": "jupiter",
    "ts": "2026-03-02T12:00:00.000Z"
  }
}
```

Failure:

```json
{
  "ok": false,
  "error": "Invalid TradingView webhook secret."
}
```

## Recommended Environment Variables

### Minimum

- `PRIVATE_KEY`
- `RPC_URL`

### TradingView Module Tuning

- `PORT`
- `TRADINGVIEW_WEBHOOK_SECRET`
- `TRADINGVIEW_COOLDOWN_MS`
- `TRADINGVIEW_DEFAULT_USD`
- `TRADINGVIEW_SOL_BUFFER_USD`
- `TRADINGVIEW_LOG_LIMIT`
- `TRADINGVIEW_EXPOSE_SECRET`

### Execution Tuning Commonly Used

- `DEFAULT_SLIPPAGE`
- `JITO_TIP_BASE`
- `JITO_TIP_MIN`
- `JITO_TIP_MAX`
- `NOZOMI_RPC_URL`
- `NOZOMI_API_KEY`

## Troubleshooting

`Invalid TradingView webhook secret.`

- Alert secret does not match server secret.
- Re-check exact value and whitespace.

`TradingView bot is disabled.`

- Enable in the UI or config endpoint.

`Buy alert already in progress.`

- Another buy is actively executing.

`Cooldown active. Next buy ready in Xs.`

- Cooldown timer still active.

`Buy amount missing. Set a USD value in the alert payload.`

- No valid payload amount and no usable default.

`Sell size missing. Set sellAll=true, percent, or amount.`

- Sell payload did not include explicit size.

`Insufficient USDC` or `No <asset> balance to sell.`

- Fund wallet or reduce trade size.

## Retail Packaging Checklist

- Give each customer a dedicated endpoint and unique secret.
- Ship conservative defaults (`cooldown`, `defaultUsd`, `solReserveUsd`).
- Provide ready-to-paste buy and sell TradingView templates.
- Include a support runbook that covers server health check, webhook test payload, wallet balance verification, and recent log inspection.

## Important Risk Notice

This software can execute real blockchain transactions.

- Not financial advice.
- No profit guarantee.
- Slippage, latency, routing, and liquidity can affect fills.
- Always test in small size before full live deployment.
