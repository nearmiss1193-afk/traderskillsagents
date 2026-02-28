# Sovereign Skill Hub - Replit Sync Pack & CrossTrade Setup

Use this guide to sync the latest changes (Verified CrossTrade Payload + Live Account Support) to your Replit.

## 1. NinjaTrader MANDATORY Setup (ATI)

CrossTrade **cannot** place orders until **Automated Trading Interface (ATI)** is turned on.

1. In NinjaTrader, go to **Tools -> Options**.
2. Select **Automated Trading Interface** on the left.
3. Check **"Enable"**.
4. **RESTART NinjaTrader**.

---

## 2. Live Account Setup (How to trade your Apex/Funded Account)

By default, the Skill Hub blocks non-SIM trades for your protection. To use your actual account:

### Step A: Find your Account Name

1. In NinjaTrader, go to the **"Accounts"** tab in the Control Center.
2. Look for the exact name (e.g., `APEX-1234-5`). **Note: It must match exactly.**

### Step B: Enable Live Trading in Replit

1. In Replit, go to **Tools -> Secrets**.
2. Add a new secret:
    * **Key:** `ALLOW_LIVE_TRADES`
    * **Value:** `true`

### Step C: Use the Dashboard

1. Type your account name into the box on the Skill Hub dashboard.
2. Click **🔗 CrossTrade**.

---

## 3. Replit Sync (Code Fixes)

Please **manually copy-paste** the contents of this file into your Replit `server/services/crosstrade.ts`.

### File: `server/services/crosstrade.ts`

```typescript
import https from "https";

const CROSSTRADE_WEBHOOK_URL = process.env.CROSSTRADE_WEBHOOK_URL || "";
const CROSSTRADE_KEY = process.env.CROSSTRADE_KEY || "";
const CROSSTRADE_ACCOUNT_DEFAULT = process.env.CROSSTRADE_ACCOUNT || "SIM101";
const MAX_CONTRACTS = parseInt(process.env.MAX_CONTRACTS || "1", 10);
const MAX_TRADES_PER_DAY = parseInt(process.env.MAX_TRADES_PER_DAY || "3", 10);

// In-memory trade counter for safety
let dailyTradeCount = 0;
let lastTradeDate = new Date().toDateString();

interface CrossTradeSignal {
    symbol: string;
    direction: "LONG" | "SHORT" | "Long" | "Short";
    qty?: number;
    orderType?: string;
    account?: string;
}

export async function sendToCrossTrade(signal: CrossTradeSignal): Promise<{ success: boolean; message: string; payload?: string }> {
    // 1. Reset daily counter if date changed
    const today = new Date().toDateString();
    if (today !== lastTradeDate) {
        dailyTradeCount = 0;
        lastTradeDate = today;
    }

    const targetAccount = signal.account || CROSSTRADE_ACCOUNT_DEFAULT;

    // 2. Safety Guardrail: Only allow SIM accounts unless ALLOW_LIVE_TRADES is true
    const isSim = targetAccount.toUpperCase().startsWith("SIM");
    if (!isSim && process.env.ALLOW_LIVE_TRADES !== "true") {
        const error = `Safety Block: Account '${targetAccount}' is NOT a SIM account. Add 'ALLOW_LIVE_TRADES=true' to Replit Secrets to enable live trading.`;
        console.error(`[crosstrade] ${error}`);
        return { success: false, message: error };
    }

    // 3. Safety Check: Max Trades Per Day
    if (dailyTradeCount >= MAX_TRADES_PER_DAY) {
        const error = `Safety Block: Daily trade limit (${MAX_TRADES_PER_DAY}) reached.`;
        console.error(`[crosstrade] ${error}`);
        return { success: false, message: error };
    }

    // 4. Validate Environment
    if (!CROSSTRADE_WEBHOOK_URL || !CROSSTRADE_KEY) {
        const error = "Missing CROSSTRADE_WEBHOOK_URL or CROSSTRADE_KEY in environment.";
        console.error(`[crosstrade] ${error}`);
        return { success: false, message: error };
    }

    const action = signal.direction.toUpperCase() === "LONG" ? "BUY" : "SELL";
    const qty = Math.min(signal.qty || 1, MAX_CONTRACTS);

    // VERIFIED FORMAT: CrossTrade expects semicolon-separated key-value pairs WITH spaces and a trailing semicolon
    const payload = `key=${CROSSTRADE_KEY}; command=PLACE; account=${targetAccount}; instrument=${signal.symbol}; action=${action}; qty=${qty}; order_type=${signal.orderType || "MARKET"}; tif=DAY;`;

    console.log(`[crosstrade] Sending payload to account ${targetAccount}: ${payload.replace(CROSSTRADE_KEY, "****")}`);

    return new Promise((resolve) => {
        const req = https.request(CROSSTRADE_WEBHOOK_URL, {
            method: "POST",
            headers: {
                "Content-Type": "text/plain",
                "Content-Length": Buffer.byteLength(payload)
            }
        }, (res) => {
            let data = "";
            res.on("data", (chunk) => data += chunk);
            res.on("end", () => {
                if (res.statusCode === 200 || res.statusCode === 201) {
                    dailyTradeCount++;
                    console.log(`[crosstrade] Order successful. Resp: ${data}`);
                    resolve({ success: true, message: `Order sent: ${data}`, payload });
                } else {
                    console.error(`[crosstrade] Error ${res.statusCode}: ${data}`);
                    resolve({ success: false, message: `HTTP ${res.statusCode}: ${data}`, payload });
                }
            });
        });

        req.on("error", (err) => {
            console.error(`[crosstrade] Network error: ${err.message}`);
            resolve({ success: false, message: `Network error: ${err.message}`, payload });
        });

        req.write(payload);
        req.end();
    });
}
```

---

## 4. Troubleshooting

* **"Order Received" but nothing in NinjaTrader?**
  * Check NinjaTrader's "Orders" tab.
  * Ensure the Account Name in the Skill Hub matches the "Account" name in NinjaTrader *exactly*.
  * Ensure **Global Simulation Mode** is turned OFF in NinjaTrader if you want live trades (Tools -> Global Simulation Mode).
