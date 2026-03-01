# veCUBE — Rewards Protocol

> **Locked Rewards UI for CubeNFT holders on Monad Mainnet (Chain ID: 143)**

A fully on-chain rewards dashboard that lets CubeNFT holders connect their wallet, scan owned NFTs, and claim epoch-based CUBE + BMC rewards via the veCube staking contracts.

---

## Table of Contents

- [Overview](#overview)
- [Live Contracts](#live-contracts)
- [Architecture](#architecture)
- [Smart Contract ABIs](#smart-contract-abis)
  - [CubeNFT ABI](#cubenfr-abi)
  - [Rewards ABI](#rewards-abi)
- [Frontend Functions](#frontend-functions)
  - [`connectWallet()`](#connectwallet)
  - [`loadNFTs()`](#loadnfts)
  - [`renderNFTs(nfts)`](#rendernftsnfts)
  - [`pick(id, sym)`](#pickid-sym)
  - [`calcRewards()`](#calcrewards)
  - [`claimRewards()`](#claimrewards)
  - [`initRead()`](#initread)
  - [`tick()`](#tick)
  - [`log(msg, cls)`](#logmsg-cls)
  - [`toast(msg)`](#toastmsg)
  - [`cp(addr)`](#cpaddr)
- [Epoch System](#epoch-system)
- [Token Prices (Oracle)](#token-prices-oracle)
- [Reward Calculation](#reward-calculation)
- [Setup & Local Development](#setup--local-development)
- [Network Configuration](#network-configuration)
- [Dependencies](#dependencies)

---

## Overview

veCUBE is a rewards protocol built on **Monad Mainnet**. Users who hold CubeNFTs and have registered vaults can claim staking rewards in two tokens — **CUBE** and **BMC** — at the end of each epoch. The UI provides:

- Wallet connection (MetaMask / Rabby / any injected provider)
- Parallel NFT scanning across the full mint range
- Live pending reward calculation via on-chain `pendingRewards()`
- One-click `claimAll()` transaction with epoch validation
- Countdown timer and epoch progress bar
- Live oracle prices for CUBE, BMC, and MON
- A terminal log panel for all RPC interactions

---

## Live Contracts

| Contract      | Address                                      | Description                        |
|---------------|----------------------------------------------|------------------------------------|
| `CubeNFT`     | `0x2BfB8C4a8D7153E88BF1947C193cA6a55e7008D1` | ERC-721 NFT with metadata          |
| `Rewards`     | `0x15E85660239753A1d1C7bf8C19b2F4c094a80669` | Epoch-based reward distribution    |
| `Oracle`      | `0xEa1644D8296b4E043b935FdCf00803E4aD4307A2` | Price feeds for CUBE, BMC, MON     |
| `Factory`     | `0x48749Ba36a2E566be33865E918BA133CA829aE2B` | veCube wallet factory              |
| `BMC Token`   | `0x244DF0E3A8276aDD749AF3ec3Ff7e642B1dC46a5` | BMC ERC-20 reward token            |
| `CUBE Token`  | `0x87Bdd0E32aca3AC42d5D35baF38937Eae840f5A3` | CUBE ERC-20 reward token           |

**veCube Contracts (selectable in UI):**

| Short Name   | Full Address                                 |
|--------------|----------------------------------------------|
| `0xc631...c558` | `0xc631D96942d54745604CEeCAc2a0C73449cAc558` |
| `0xEf7A...dA3`  | `0xEf7Af1c55eDf4d15eBCE99734D64451716B77dA3` |
| `0x4d90...83C`  | `0x4d90Ed14b5d2cA55E105A3c2F71Da210EddeD83C` |
| `0x322a...De3`  | `0x322a3A51872121E264BaB0F8e85b989055A89De3` |
| `0x1182...83A`  | `0x118257222f4Dd6E89F318694bd6e05eF0a00C83A` |

---

## Architecture

```
User Browser
    │
    ├─ window.ethereum (MetaMask / Rabby)
    │       └─ BrowserProvider → Signer → write calls
    │
    ├─ JsonRpcProvider (Alchemy RPC — read-only)
    │       ├─ CubeNFT.totalMinted()
    │       ├─ CubeNFT.balanceOf(wallet)
    │       ├─ CubeNFT.ownerOf(id) × N  (parallel batches)
    │       ├─ CubeNFT.getCubeData(id)  × owned
    │       ├─ Rewards.currentEpoch()
    │       └─ Rewards.pendingRewards(veCube, tokenId)
    │
    └─ Transaction (write)
            └─ Rewards.claimAll(veCube, tokenId, epoch - 1)
```

---

## Smart Contract ABIs

### CubeNFT ABI

#### Read Functions

| Function | Signature | Returns | Description |
|----------|-----------|---------|-------------|
| `totalMinted` | `totalMinted() view` | `uint256` | Total NFTs ever minted |
| `balanceOf` | `balanceOf(address owner) view` | `uint256` | NFT balance of an address |
| `ownerOf` | `ownerOf(uint256 tokenId) view` | `address` | Owner of a specific token |
| `getCubeData` | `getCubeData(uint256 tokenId) view` | `CubeData` tuple | Full metadata for a token |
| `getCubeCoin` | `getCubeCoin(uint256 tokenId) view` | `address` | Meme coin address tied to token |
| `cubeData` | `cubeData(uint256) view` | tuple fields | Raw mapping access |
| `cubeCoin` | `cubeCoin(uint256) view` | `address` | Raw coin mapping access |
| `getApproved` | `getApproved(uint256 tokenId) view` | `address` | Approved address for token |
| `isApprovedForAll` | `isApprovedForAll(address,address) view` | `bool` | Operator approval check |
| `name` | `name() view` | `string` | Token collection name |
| `symbol` | `symbol() view` | `string` | Token collection symbol |
| `tokenURI` | `tokenURI(uint256 tokenId) view` | `string` | Metadata URI |
| `owner` | `owner() view` | `address` | Contract owner (Ownable) |
| `supportsInterface` | `supportsInterface(bytes4) view` | `bool` | ERC-165 check |

#### Write Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `mint` | `mint(address to)` | Mints a new CubeNFT to `to` (owner only) |
| `approve` | `approve(address to, uint256 tokenId)` | Approve single token transfer |
| `setApprovalForAll` | `setApprovalForAll(address operator, bool approved)` | Approve operator for all tokens |
| `transferFrom` | `transferFrom(address from, address to, uint256 tokenId)` | Transfer token |
| `safeTransferFrom` | `safeTransferFrom(address from, address to, uint256 tokenId)` | Safe transfer (with receiver check) |
| `safeTransferFrom` | `safeTransferFrom(address,address,uint256,bytes)` | Safe transfer with data payload |
| `transferOwnership` | `transferOwnership(address newOwner)` | Transfer contract ownership |
| `renounceOwnership` | `renounceOwnership()` | Permanently remove owner |

#### CubeData Struct

```solidity
struct CubeData {
    uint8   colorIndex;   // Index into color palette (0–7)
    uint256 seed;         // Random seed used for generation
    uint256 timestamp;    // Block timestamp at mint
    address memeCoin;     // Associated meme coin contract
    string  coinSymbol;   // Ticker symbol of the meme coin
}
```

#### Events

| Event | Parameters | Description |
|-------|------------|-------------|
| `CubeMinted` | `tokenId, memeCoin, color, coinName` | Fired on every mint |
| `Transfer` | `from, to, tokenId` (all indexed) | Standard ERC-721 transfer |
| `Approval` | `owner, approved, tokenId` (all indexed) | Token approval |
| `ApprovalForAll` | `owner, operator, approved` | Operator approval |
| `OwnershipTransferred` | `previousOwner, newOwner` | Ownership change |

---

### Rewards ABI

| Function | Signature | Description |
|----------|-----------|-------------|
| `currentEpoch` | `currentEpoch() view returns (uint256)` | Returns the current epoch number |
| `pendingRewards` | `pendingRewards(address veCube, uint256 tokenId) view returns (address[] tokens, uint256[] amounts, uint256 epoch)` | Returns claimable token addresses and amounts for a given veCube + NFT |
| `claimAll` | `claimAll(address veCube, uint256 tokenId, uint256 epoch) external` | Claims all pending rewards for `epoch` (must be `currentEpoch - 1`) |

> **Note:** Claiming is only possible after an epoch ends. Epoch 0 ends **March 31, 2026**. Call `advanceEpoch()` on the contract to open claiming for all holders.

---

## Frontend Functions

### `connectWallet()`

Connects the user's injected wallet (MetaMask, Rabby, etc.) and switches to Monad Mainnet (Chain ID 143). If the chain is not added, it prompts the wallet to add it automatically.

**Flow:**
1. Calls `eth_requestAccounts` to get the user's address
2. Calls `wallet_switchEthereumChain` to switch to Monad (`chainId: 0x8f`)
3. Falls back to `wallet_addEthereumChain` if Monad is not in the wallet
4. Creates a `BrowserProvider` + `Signer` via ethers v6
5. Validates the connected chain ID matches `143`
6. Updates the header UI with a shortened wallet address pill
7. Triggers `loadNFTs()` automatically

**Error handling:** Handles user rejection (`code: 4001`), chain-not-added (`code: 4902`), and generic RPC errors. Always continues gracefully.

---

### `loadNFTs()`

Scans the full CubeNFT collection for tokens owned by the connected wallet, using parallel RPC batches of 20.

**Flow:**
1. Calls `balanceOf(wallet)` and `totalMinted()` in parallel
2. Iterates token IDs `1..totalMinted` in batches of 20 using `Promise.allSettled`
3. For each token ID, calls `ownerOf(id)` — if matched, calls `getCubeData(id)`
4. Renders found NFTs progressively as each batch resolves
5. Stops early once balance count is satisfied

**Performance:** Uses `Promise.allSettled` so a single failed RPC call doesn't abort the entire scan. Progress is logged to the terminal in real time.

---

### `renderNFTs(nfts)`

Renders the scanned NFT array into the NFT grid panel as clickable cards.

**Parameters:**
- `nfts` — `Array<{ id: number, ci: number, sym: string }>` — scanned NFT objects

Each card shows:
- A colored SVG hexagon (color derived from `colorIndex % 8`)
- Token ID formatted as `#001`
- Coin symbol (e.g. `PEPE`)
- A "tap to select" hint

Clicking a card calls `pick(id, sym)`.

---

### `pick(id, sym)`

Selects an NFT card and auto-populates the Claim form.

**Parameters:**
- `id` — Token ID (number)
- `sym` — Coin symbol string

**Actions:**
- Removes `.sel` class from all NFT cards
- Adds `.sel` to the clicked card (green border + glow)
- Sets the Token ID input field value
- Calls `calcRewards()` to fetch pending amounts

---

### `calcRewards()`

Fetches pending reward amounts for the currently selected veCube contract + token ID.

**Reads:** `Rewards.pendingRewards(veCube, tokenId)` via the read-only provider.

**Returns** two token arrays — iterates them to separate CUBE and BMC amounts by matching token addresses against `ADDR.cube` and `ADDR.bmc`.

**Displays:**
- `cubeAmt` — formatted CUBE amount (18 decimals)
- `cubeUsd` — USD value at `$0.0010/CUBE`
- `bmcAmt` — formatted BMC amount (18 decimals)
- `bmcUsd` — USD value at `$0.0500/BMC`

**On error** (e.g. epoch still active): displays `0` amounts and `(epoch active)` message.

---

### `claimRewards()`

Submits the `claimAll()` transaction for the selected veCube + token ID.

**Pre-checks:**
- Wallet must be connected (calls `connectWallet()` if not)
- veCube contract and token ID must be selected
- `currentEpoch()` must be `> 0` (Epoch 0 is not yet claimable)

**Transaction:**
- Calls `Rewards.claimAll(veCube, tokenId, currentEpoch - 1)` via the signer
- Waits for `tx.wait()` (1 confirmation)
- Re-fetches rewards after success via `calcRewards()`

**Button states:** Disabled during signing/broadcasting/confirming with appropriate labels. Re-enabled on completion or error.

---

### `initRead()`

Initializes the read-only `JsonRpcProvider` on page load and populates global stats.

**Reads:**
- `CubeNFT.totalMinted()` → updates `#s-minted`
- `Rewards.currentEpoch()` → updates `#s-epoch` and `#epochNum`

Called automatically on page load (no wallet required).

---

### `tick()`

Updates the epoch countdown timer every second using `setInterval`.

**Calculates:**
- Days, hours, minutes, seconds remaining until `2026-03-31T00:00:00Z`
- Epoch progress percentage based on `(now - MAR_01) / (MAR_31 - MAR_01)`

**Updates:**
- `#cd-d`, `#cd-h`, `#cd-m`, `#cd-s` — countdown digits
- `#progF` — progress bar fill width
- `#progPct` — percentage label

---

### `log(msg, cls)`

Appends a timestamped line to the terminal panel at the bottom of the UI.

**Parameters:**
- `msg` — Log message string
- `cls` — CSS class: `'ok'` (green), `'er'` (red), `'wn'` (yellow). Default: `'ok'`

Auto-scrolls the terminal to the latest entry.

---

### `toast(msg)`

Displays a brief notification toast at the bottom center of the screen.

**Parameters:**
- `msg` — Message string to display

Auto-dismisses after **3.5 seconds**.

---

### `cp(addr)`

Copies a contract address to the clipboard and shows a confirmation toast.

**Parameters:**
- `addr` — Full Ethereum address string

Used by the Contracts panel for one-click address copying.

---

## Epoch System

| Property     | Value                        |
|--------------|------------------------------|
| Epoch 0 Start | March 1, 2026               |
| Epoch 0 End   | March 31, 2026              |
| APR (both tokens) | 10%                    |
| Total BMC Pool | 10 Trillion BMC            |
| Registered Vaults | 132                   |

Rewards accrue during an epoch but **cannot be claimed until the epoch ends**. After `advanceEpoch()` is called on the contract, all holders may claim for the previous epoch (`currentEpoch - 1`).

---

## Token Prices (Oracle)

Prices displayed in the UI are sourced from Oracle contract `0xEa16...7A2`:

| Token | Price    |
|-------|----------|
| CUBE  | $0.0010  |
| BMC   | $0.0500  |
| MON   | $0.0208  |

---

## Reward Calculation

Rewards displayed in the UI are fetched live from the contract:

```
pendingRewards(veCubeAddress, tokenId)
  → returns (address[] tokens, uint256[] amounts, uint256 epoch)
```

USD value is calculated client-side:
```
CUBE USD = cubeAmount × 0.001
BMC  USD = bmcAmount  × 0.05
```

---

## Setup & Local Development

No build step required — this is a single-file HTML application.

```bash
# Clone the repository
git clone https://github.com/your-org/vecube-rewards.git
cd vecube-rewards

# Serve locally (any static server works)
npx serve .
# or
python3 -m http.server 8080
```

Open `http://localhost:8080` in a browser with MetaMask or Rabby installed.

---

## Network Configuration

| Parameter    | Value                                                      |
|--------------|------------------------------------------------------------|
| Chain Name   | Monad Mainnet                                              |
| Chain ID     | `143` (`0x8f`)                                             |
| RPC (primary)| `https://rpc.monad.xyz`                                    |
| RPC (Alchemy)| `https://monad-mainnet.g.alchemy.com/v2/<KEY>`            |
| Explorer     | `https://monadexplorer.com`                                |
| Currency     | MON (18 decimals)                                          |

The UI will automatically prompt the user to add Monad to their wallet if it is not already configured.

---

## Dependencies

| Library   | Version | Source                                     | Usage                         |
|-----------|---------|--------------------------------------------|-------------------------------|
| ethers.js | 6.7.0   | cdnjs.cloudflare.com                       | RPC provider, ABI encoding    |
| IBM Plex Mono | —   | Google Fonts                               | UI monospace font             |
| Bebas Neue | —      | Google Fonts                               | Display / number font         |

No npm install, no bundler, no framework — pure HTML + vanilla JS + ethers UMD build.

---

## Error Reference

| Error Code | Meaning | Handling |
|------------|---------|----------|
| `4001` | User rejected request | Show toast, log warning |
| `4902` | Chain not added to wallet | Auto-add Monad via `wallet_addEthereumChain` |
| `-32603` | Internal RPC error (often chain mismatch) | Attempt chain add |
| `ERC721NonexistentToken` | Token ID does not exist | Skipped silently during scan |
| `currentEpoch === 0` | Epoch not yet ended | Block claim, show message |

---

## License

MIT — see `LICENSE` for details.
