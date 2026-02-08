# 🔗 Chain Hook: Tame the Glitch
### *Gamified Uniswap v4 Liquidity Management*

**Chain Hook** is a Web3 pixel-art platformer where each "monster" represents a critical state of a Uniswap v4 Liquidity Pool. By leveraging the player's **NFT metadata**, the game identifies the user's LP position and allows them to "tame" market inefficiencies by triggering specific **Uniswap Hooks**.

---

## 👾 The Glitch Bestiary (The Hooks)

To stabilize the system, the player must defeat three types of "Glitches," each linked to a real-world Hook contract that manages the user's LP position:

### 1. The Range-Shifter
* **Glitch Name:** `Volatility Specter`
* **Logic:** Represents the inefficiency of an "Out of Range" position.
* **The Hook:** [Adjust position range](/hooks/FullRange.sol)
* **Action:** When defeated, the game triggers the hook logic to **rebalance the position** into a Full Range state, ensuring the player's liquidity is always earning fees regardless of price swings.

### 2. The Crash-Eater
* **Glitch Name:** `Dip-Vortex`
* **Logic:** Triggers when the price drops below a specific target threshold.
* **The Hook:** [Buyback Hook](/hooks/Treasury.sol)
* **Action:** Defeating this glitch activates an **automated buyback protocol**. The hook uses next user's fees to buy back the token during crashes, creating algorithmic support and accumulating assets at a discount for the user.

### 3. The Flash-Drainer (Final Boss)
* **Glitch Name:** `Arbitrage Singularity`
* **Logic:** Represents value lost to MEV or low liquidity density.
* **The Hook:** [Arbitrage Trigger](https://github.com/jtriley2p/huff-hooks)
* **Action:** The final boss requires low-level execution logic (Huff). Once tamed, it activates a **Flashloan-based arbitrage bot**. This bot exploits external price discrepancies to refill the player's liquid pool shares, turning potential arbitrage into LP profit.



---

## 🛠 Tech Stack

* **Frontend:** Vue.js 3 + Vite
* **Web3:** Ethers.js / Wagmi for MetaMask interaction
* **Contracts:** Solidity & Huff (Uniswap v4 Hooks)
* **Metadata:** JSON-based NFT traits for PoolKey identification

---

## ⚙️ Environment Configuration

To connect the game to the blockchain and fetch your NFT metadata/LP status, create a `.env` file in the root directory:

```env
# RPC Provider (Alchemy or Infura)
VITE_RPC_URL=[https://eth-mainnet.g.alchemy.com/v2/YOUR_API_KEY](https://eth-mainnet.g.alchemy.com/v2/YOUR_API_KEY)

# The Contract Address of the NFT representing your LP position
VITE_NFT_COLLECTION_ADDRESS=0x...

# Chain ID (e.g., 1 for Mainnet, 11155111 for Sepolia)
VITE_CHAIN_ID=1

# Optional: Explorer API for fetching pool logs
VITE_ETHERSCAN_API_KEY=YOUR_API_KEY

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```
