# Sovereign Skill Hub — CrossTrade Integration Setup

This document outlines how to connect the Skill Hub server to NinjaTrader via the CrossTrade webhook bridge.

## 🚀 One-Minute Bridge Setup

1. **CrossTrade Setup**:
   - Obtain your **CrossTrade Webhook URL** and **API Key**.
   - Ensure the CrossTrade Trade Router is active and connected to your NinjaTrader instance.

2. **Replit Environment Variables**:
   Add the following secrets to your Replit environment:
   - `CROSSTRADE_WEBHOOK_URL`: Your unique CrossTrade endpoint.
   - `CROSSTRADE_KEY`: Your CrossTrade API key.
   - `CROSSTRADE_ACCOUNT`: Your NinjaTrader account name (e.g., `SIM101`). **MUST start with "SIM" for safety.**
   - `MAX_CONTRACTS`: Maximum number of contracts per trade (Default: 1).
   - `MAX_TRADES_PER_DAY`: Maximum number of executions allowed per 24h (Default: 3).

3. **Verification**:
   - Refresh the Skill Hub Dashboard.
   - Click the **🔗 CrossTrade** button in the "AI Futures Trader" tab to send a test MES Long order.
   - Verify the order appears instantly in your NinjaTrader SIM account.

## 🛡️ Safety Guardrails (Prop-Firm Safe)

The integration includes several hard-coded safety rules:

- **SIM Enforcement**: The server will refuse to send any orders if the `CROSSTRADE_ACCOUNT` does not begin with "SIM". This prevents accidental live execution.
- **Daily Caps**: Orders are capped at 3 per day by default (configurable).
- **Quantity Limit**: All orders are capped at the `MAX_CONTRACTS` setting (default 1) regardless of signal input.
- **Text/Plain Formatting**: Payload is automatically formatted to CrossTrade's required `key=...; command=...;` standard.

## 📊 Live Monitoring

- Logs will appear in the Replit console under `[crosstrade]`.
- The dashboard "CrossTrade" status badge will indicate if an execution is in progress or if an error occurred.
