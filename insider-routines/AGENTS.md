# 🔭 Insider Routines — The Seven AI Agents Suite

Welcome to the **Insider Routines** documentation! This suite contains seven specialized AI agents designed to read, analyze, and synthesize public government signals, institutional flows, on-chain whale transactions, and your personal portfolio drift. Together, they form a multi-agent consensus system that alerts you only when multiple independent sources agree.

---

## 🏗️ System Architecture & Data Flow

Below is the interaction and data flow diagram of the suite, showing how the Scouts feed the consensus engine, which triggers the dispatcher to notify you:

```mermaid
graph TD
    %% Scouts Row
    subgraph Scouts ["🔭 The Scouts (Signal Generators)"]
        eddie["Eddie (SEC Form 4)"]
        maggie["Maggie (13F Holdings)"]
        frank["Frank (Fed Speeches)"]
        maya["Maya (On-Chain Whales)"]
        janet["Janet (Portfolio Drift)"]
    end

    %% Storage & Logic
    db[("SQLite DB<br>~/.state/state.db<br>(signals, consensus)")]
    gemini["Google Gemini API<br>(Gemini 2.5/3.5 Flash)"]
    
    %% Consensus & Dispatch
    sophie["Sophie (Consensus Analyst)<br>Runs every 30 mins"]
    ross["Ross (Dispatcher)<br>Runs every 30 mins"]
    
    %% Outputs
    gmail["Gmail Alert"]
    telegram["Telegram Message<br>(Optional)"]

    %% Connections
    eddie -->|Query / Parse| gemini
    maggie -->|Query / Parse| gemini
    frank -->|Query / Parse| gemini
    maya -->|Query / Parse| gemini
    
    gemini -->|Save Signal| db
    janet -->|Save Signal (Local Logic)| db
    
    db -->|Read 7-Day Window| sophie
    sophie -->|Write Consensus Event| db
    
    db -->|Read Pending Events| ross
    ross -->|Send SMTP| gmail
    ross -->|Send Webhook| telegram
```

---

## 🔭 The Scouts (5 Agents)

The Scouts are individual intelligence agents that specialize in one specific sector of financial signal generation. Eddie, Maggie, Frank, and Maya use Gemini to perform complex parsing and web-research reasoning, while Janet uses high-speed local Python logic.

---

### 1. Eddie — The SEC Form 4 Watcher

* **Purpose**: Tracks corporate insider sentiment by monitoring open-market stock purchases made by C-suite executives and board directors of publicly traded companies.
* **Scope & Sources**: Scans the **SEC EDGAR database** full-text search index for newly filed Form 4 documents published in the last 24 hours.
* **Schedule**: Runs **Daily at 06:00 Local Time**.
* **Model**: Gemini (defaults to `gemini-2.5-flash`).
* **Filters & Criteria**:
  * Filer must hold a key decision-making role: CEO, CFO, President, Chairman, or Board Director.
  * Transaction code must be **`P`** (Open-market purchase; options exercises are excluded).
  * Minimum transaction value must be **$\ge$ $100,000**.
* **Example Use Case**:
  * *Scenario*: The CEO of a major tech firm buys $1.2M of their own company's shares on the open market following a stock correction.
  * *Eddie's Signal*: Emits a `BULLISH` signal with high confidence (`5`), citing the CEO's massive personal capital deployment.
* **SQLite Output Schema**:

  ```json
  {"ticker": "AAPL", "direction": "BULLISH", "confidence": 5, "reason": "CEO purchased $1.2M of shares on the open market."}
  ```

---

### 2. Maggie — The Smart-Money Tracker

* **Purpose**: Monitors long-term positioning and massive capital shifts made by the world's most successful institutional hedge funds and asset managers.
* **Scope & Sources**: Pulls and compares the latest **13F-HR filings** from five key funds on SEC EDGAR:
  1. *Berkshire Hathaway* (Warren Buffett)
  2. *Bridgewater Associates* (Ray Dalio)
  3. *Renaissance Technologies* (Jim Simons)
  4. *Citadel Advisors* (Ken Griffin)
  5. *Two Sigma Investments*
* **Schedule**: Runs **Weekly on Sunday at 19:00 Local Time**.
* **Model**: Gemini (defaults to `gemini-2.5-flash`).
* **Filters & Criteria**:
  * Classifies transactions as `NEW POSITION`, `INCRED` ( $\ge 25\%$ increase), or `EXITED`.
  * Minimum trade value must be **$\ge$ $50,000,000**.
  * Selects the single most significant, high-conviction move across all five funds.
* **Example Use Case**:
  * *Scenario*: Warren Buffett's Berkshire Hathaway completely exits a $300M stake in a major bank.
  * *Maggie's Signal*: Emits a `BEARISH` signal on the banking ticker with confidence (`4`), highlighting a institutional complete exit.
* **SQLite Output Schema**:

  ```json
  {"ticker": "BAC", "direction": "BEARISH", "confidence": 4, "reason": "Berkshire Hathaway completely EXITED their $300M position."}
  ```

---

### 3. Frank — The Fed-Speech Reader

* **Purpose**: Deciphers macroeconomic direction, interest rate outlooks, and liquidity trends by reading central bank communications.
* **Scope & Sources**: Scans newly published Federal Reserve speeches, congressional testimonies, and FOMC policy statements from **`federalreserve.gov/newsevents/speeches.htm`** in the last 7 days.
* **Schedule**: Runs **Weekly on Monday at 08:00 Local Time**.
* **Model**: Gemini (defaults to `gemini-2.5-flash`).
* **Filters & Criteria**:
  * Reads and extracts speakers, stances, and quotes ($\le 25$ words).
  * Classifies the speeches as Hawkish (tighter policy), Dovish (looser policy), or Neutral.
  * Aggregates the net stance and applies it to risk assets (Equities & Crypto).
  * *Stance Mapping*: Net Dovish $\rightarrow$ `BULLISH` (liquidity injection); Net Hawkish $\rightarrow$ `BEARISH` (liquidity withdrawal).
* **Example Use Case**:
  * *Scenario*: Fed Chairman Jerome Powell gives a highly anticipated speech hinting that interest rate cuts are coming.
  * *Frank's Signal*: Emits a `BULLISH` signal on `MACRO` with confidence (`5`), citing clear dovish guidance on interest rates.
* **SQLite Output Schema**:

  ```json
  {"ticker": "MACRO", "direction": "BULLISH", "confidence": 5, "reason": "Powell dovish pivot: Speeches indicate rate cuts are coming as inflation cools."}
  ```

---

### 4. Maya — The On-Chain Whale Watcher

* **Purpose**: Flags immediate large-scale movements of capital on public blockchains (Ethereum / Bitcoin) that signal whale accumulation or distribution.
* **Scope & Sources**: Queries public explorers for high-value blockchain transfers in the last 6 hours, tracking **WBTC, WETH, USDC, and USDT**.
* **Schedule**: Runs **Every 6 Hours**.
* **Model**: Gemini (defaults to `gemini-2.5-flash`).
* **Filters & Criteria**:
  * Minimum transaction value must be **$\ge$ $5,000,000**.
  * *Accumulation*: Assets moving from a Centralized Exchange (CEX) wallet to a private self-custody wallet $\rightarrow$ `BULLISH` (supply shock / holding).
  * *Distribution*: Assets moving from a private wallet into a CEX wallet $\rightarrow$ `BEARISH` (selling pressure / liquidation).
* **Example Use Case**:
  * *Scenario*: A whale transfers $42M worth of Ethereum (WETH) from a private cold storage wallet directly into Binance.
  * *Maya's Signal*: Emits a `BEARISH` signal on `ETH` with confidence (`4`), indicating potential distribution / incoming sell pressure.
* **SQLite Output Schema**:

  ```json
  {"ticker": "ETH", "direction": "BEARISH", "confidence": 4, "reason": "Whale moved $42M WETH from private wallet to Binance (potential selling pressure)."}
  ```

---

### 5. Janet — The Portfolio-Drift Accountant

* **Purpose**: Keeps your portfolio allocation aligned with your target strategy, signaling when market moves have drifted you out of tolerance.
* **Scope & Sources**: Pure local Python logic. Compares your current asset valuation (`config/portfolio_current.json`) against your target allocations (`config/portfolio_target.json`).
* **Schedule**: Runs **Daily at 17:00 Local Time**.
* **Model**: None (runs high-speed, cost-free local mathematical logic).
* **Filters & Criteria**:
  * Triggers only when an asset's allocation drifts **$\ge$ 5 percentage points** away from its target.
  * *Underweight*: Allocation is too low $\rightarrow$ `BULLISH` (signals that you should buy more to rebalance).
  * *Overweight*: Allocation is too high $\rightarrow$ `BEARISH` (signals that you should trim/sell to rebalance).
* **Example Use Case**:
  * *Scenario*: A massive rally in Bitcoin (BTC) drives your cryptocurrency allocation up to 32%, while your target allocation is only 25%.
  * *Janet's Signal*: Emits a `BEARISH` signal on `BTC` with confidence (`2`) indicating that your portfolio is overweight and it is time to trim.
* **SQLite Output Schema**:

  ```json
  {"ticker": "BTC", "direction": "BEARISH", "confidence": 2, "reason": "BTC drifted +7.0pp (target 25.0% -> current 32.0%) - trim position."}
  ```

---

## ⚖️ The Consensus (1 Agent)

### Sophie — The Consensus Analyst

* **Purpose**: The central brain. Evaluates all scout signals generated over a rolling 7-day window and filters out the noise. Sophie **only fires** when there is a clear confluence of agreement across multiple distinct disciplines.
* **Scope & Sources**: Reads the `signals` table in the local SQLite state store.
* **Schedule**: Runs **Every 30 minutes**.
* **Logic & Criteria**:
  * Groups active non-neutral signals in the last 7 days by `(ticker, direction)`.
  * Guards against duplicates (only counts the latest signal per scout per ticker).
  * Fires a **CONSENSUS** event only if **$\ge$ 3 distinct scouts** agree on the exact same ticker and direction.
* **Example Output in database (`consensus` table)**:

  ```json
  {
    "ticker": "BTC",
    "direction": "BULLISH",
    "scouts": ["eddie", "frank", "maggie"],
    "reasons": [
      "eddie: CEO purchased $250,000 of shares on open market.",
      "frank: Fed commentary shifted net-dovish on cooling inflation.",
      "maggie: Berkshire Hathaway opened a new $150M position."
    ],
    "ts": "2026-05-27T00:12:00Z",
    "dispatched": 0
  }
  ```

---

## 📡 The Dispatcher (1 Agent)

### Ross — The Dispatcher

* **Purpose**: Secure delivery. Reads the local SQLite database for newly generated consensus events that have not yet been sent to the user.
* **Scope & Sources**: Reads pending items in `consensus` table where `dispatched = 0`.
* **Schedule**: Runs **Every 30 minutes** (offset slightly from Sophie).
* **Delivery Channels**:
  1. **Gmail SMTP**: Sends a beautifully formatted plain-text structured email to your configured inbox.
  2. **Telegram Bot** (Optional): If configured with bot token and chat ID, sends a markdown-formatted message directly to your chat app.
* **Execution Rule**: Once successfully sent via Gmail, Ross marks `dispatched = 1` in the database to prevent duplicate notifications.
* **🛡️ Core Security Principle**: **Never places trades.** Output is strictly informational. The human is always the ultimate decision-maker.
