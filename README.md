# Solana Volume Bot

A Solana bot that generates trading volume on **Raydium AMM v4** pools using multiple wallets and **Jito bundles** for reliable transaction inclusion.

**Contact:** [Telegram @jerrix1](https://t.me/jerrix1)

**Repository:** [github.com/thesSmartApe/solana-volume-bot](https://github.com/thesSmartApe/solana-volume-bot)

---

## About the Bot

This bot creates **simulated trading volume** on a given Raydium liquidity pool by:

- Using **multiple keypair wallets** (default: 5) that each hold SOL/WSOL
- Executing **buy then sell** swaps on the pool in a single transaction per wallet
- Sending all swap transactions as a **Jito bundle** so they land in the same block
- Repeating the process for a configurable number of **cycles** with a **delay** between each cycle

Use cases include testing pool behavior, generating volume for analytics or incentives, and similar on-chain workflows. You are responsible for complying with applicable rules and for any SOL/token loss from fees, tips, and slippage.

---

## Bot Strategy

### How It Works

1. **Pool & market**  
   You provide a **Raydium AMM pool ID** (market ID). The bot derives pool keys (AMM, OpenBook market, vaults, etc.) from this pool.

2. **Wallets**  
   The bot uses **5 wallets** by default (stored in `src/keypairs/`). Each wallet needs SOL and a WSOL (wrapped SOL) ATA to swap.

3. **Swap flow (per wallet, per cycle)**  
   For each wallet in a cycle the bot builds one transaction that:
   - Creates the **base token ATA** for that wallet if needed (idempotent)
   - **Buys** the pool’s base token with WSOL (swap: WSOL → token)
   - **Sells** that token back for WSOL (swap: token → WSOL)

   So each cycle is one round-trip (buy + sell) per wallet, generating volume on both sides.

4. **Jito bundles**  
   All per-wallet transactions in a cycle are sent as a **single Jito bundle**. The **last transaction** in the bundle includes a **Jito tip** (SOL) from your main wallet so the bundle is prioritized by block engines.

5. **Costs**  
   - **Jito tip** (SOL) per bundle  
   - **Pool/swap fees** (e.g. ~0.25–0.30% on Raydium; token taxes if any are extra)  
   - **SOL rent** for ATAs and transaction fees  

   The **Simulate Volume** option in the menu helps you estimate total volume and SOL loss before running.

6. **Reclaim**  
   After you’re done, **Reclaim SOL/WSOL** closes WSOL ATAs and sends remaining SOL from the volume wallets back to your main wallet.

---

## Prerequisites

- **Node.js** (v18+)
- **Solana RPC** (e.g. Helius, QuickNode, or another mainnet RPC)
- **SOL** on your main wallet for:
  - Funding the 5 volume wallets
  - Jito tips per bundle
  - Transaction and rent fees
- **Jito**  
  - A Jito **block engine** (default: `frankfurt.mainnet.block-engine.jito.wtf`)  
  - An **auth keypair** for the searcher (stored as `blockengine.json` at project root)

---

## Setup

### 1. Clone and install

```bash
git clone https://github.com/thesSmartApe/solana-volume-bot.git
cd solana-volume-bot
npm install
```

### 2. Configure `config.ts` (root)

Edit **`config.ts`** in the project root:

- **`connection`**  
  Set your Solana RPC URL (mainnet).

- **`wallet`**  
  Set the **private key** of the wallet that:
  - Sends SOL to the volume wallets
  - Pays Jito tips
  - Signs the “tip” transaction in each bundle  

  Use a **base58**-encoded secret key:

```ts
export const connection = new Connection('YOUR_RPC_URL', { commitment: 'confirmed' });

export const wallet = Keypair.fromSecretKey(
  bs58.decode('YOUR_BASE58_PRIVATE_KEY')
);
```

### 3. Jito configuration

- **Auth keypair**  
  Place your Jito searcher auth keypair at **`blockengine.json`** (root). The file should contain the keypair as a JSON array of numbers (same format as Solana keypair files).

- **Optional env (`.env`)**  
  You can override defaults via environment variables (see `src/clients/config.ts`), for example:
  - `BLOCK_ENGINE_URLS` – block engine endpoint(s)
  - `AUTH_KEYPAIR_PATH` – path to auth keypair (default `./blockengine.json`)
  - `RPC_URL` – RPC URL (if you want to use it from Jito client config)
  - `GEYSER_URL` / `GEYSER_ACCESS_TOKEN` – only if you use Geyser

The bot uses the Jito client that reads from this config and sends bundles to the first block engine URL.

### 4. Create volume wallets (first-time)

Run the bot and choose **option 1 – Create Keypairs**:

- **Create new:** generates 5 new keypairs and saves them under `src/keypairs/` (e.g. `keypair1.json` … `keypair5.json`).
- **Use existing:** loads keypairs from that folder.

Back up `src/keypairs/` and any `keyInfo.json` if you care about reusing the same wallets.

---

## How to Use

Start the bot:

```bash
npx ts-node main.ts
```

You’ll see a menu:

| Option | Description |
|--------|-------------|
| **1. Create Keypairs** | Generate 5 new volume wallets or load existing ones from `src/keypairs/`. |
| **2. Distribute SOL/WSOL** | Send SOL to volume wallets and create WSOL ATAs, or send WSOL for “volume” mode. |
| **3. Simulate Volume** | Estimate total volume (SOL/USD) and SOL loss (fees + Jito tips) before running. |
| **4. Start Volume** | Run the volume strategy: enter pool ID, cycles, delay, and Jito tip; bot runs bundled swaps. |
| **5. Reclaim SOL/WSOL** | Close WSOL ATAs and return SOL from volume wallets to your main wallet. |
| **exit** | Quit. |

### Typical workflow

1. **Create Keypairs** (once)  
   Create or load the 5 volume wallets.

2. **Distribute SOL/WSOL**  
   - Sub-option **1**: Send SOL to each wallet and create WSOL ATAs (for running volume).
   - Sub-option **2**: Send WSOL only (if you already have ATAs and just need more WSOL).

3. **Simulate Volume** (optional)  
   Enter SOL price, Jito tip per bundle, number of executions, and SOL per wallet. The bot prints estimated total volume and SOL spent.

4. **Start Volume**  
   - **Market ID:** Raydium AMM pool ID (the pool you want to generate volume on).
   - **Number of bundled swaps:** e.g. `10` = 10 cycles (each cycle = one bundle of 5 wallets doing buy+sell).
   - **Delay (seconds):** e.g. `3` = wait 3 seconds between cycles.
   - **Jito tip (SOL):** e.g. `0.01` per bundle.

   The bot will run that many cycles, each time building and sending one Jito bundle (all 5 wallets, buy+sell each), then waiting the specified delay.

5. **Reclaim SOL/WSOL**  
   When finished, run this to pull SOL back from the volume wallets and close their WSOL ATAs.

---

## Important Notes

- **RPC:** Use a reliable, rate-limited mainnet RPC. Free public RPCs may throttle or drop requests.
- **Jito:** Bundles are sent to the Jito block engine; tip amount affects inclusion. If you see “Bundle Dropped, no connected leader up soon,” try a higher tip or different block engine region.
- **Pool:** The pool must be a Raydium AMM v4 pool; the bot derives OpenBook/Raydium accounts from the pool ID.
- **Fees & slippage:** You lose SOL to fees, Jito tips, and slippage. Simulate first and start with small amounts.
- **Security:** Never commit `config.ts` or `blockengine.json` or any keypair file; add them to `.gitignore` and keep backups secure.

---

## Project structure (overview)

- **`main.ts`** – Entry point and menu (create keypairs, distribute, simulate, start volume, reclaim).
- **`config.ts`** – RPC URL and main wallet keypair (SOL sender + Jito tipper).
- **`src/bot.ts`** – Volume logic: load keypairs, build buy/sell swaps, send Jito bundle.
- **`src/distribute.ts`** – Distribute SOL/WSOL to keypairs and reclaim SOL/WSOL.
- **`src/simulate.ts`** – Estimate volume and SOL loss.
- **`src/createKeys.ts`** – Create or load keypairs under `src/keypairs/`.
- **`src/clients/`** – Jito client, pool keys (Raydium/OpenBook), lookup tables, etc.
- **`blockengine.json`** – Jito searcher auth keypair (do not commit).

---

## License

ISC.

For questions or support, contact **[@jerrix1 on Telegram](https://t.me/jerrix1)**.
