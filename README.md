# Deploy a Smart Contract on Base Mainnet

This guide walks you through the full journey of deploying a smart contract to the [Base](https://base.org) mainnet. It covers environment preparation, bridging funds, configuring your tooling, deploying with Hardhat, verifying your contract, and post-deployment best practices. Even if you are new to Base, you can follow this guide end to end to achieve a successful deployment.

---

## 1. Prerequisites

### 1.1 Skills and Knowledge
- Comfortable with JavaScript/TypeScript or Solidity basics.
- Familiarity with Ethereum development workflows (accounts, gas, transactions).
- Understanding of command-line usage and package managers such as `npm` or `yarn`.

### 1.2 Software Requirements
| Tool | Version | Purpose |
| --- | --- | --- |
| Node.js | 18.x LTS or later | Runs Hardhat or Foundry scripts |
| npm or yarn | Latest | Installs project dependencies |
| Git | Latest | Manages your project repository |
| Hardhat or Foundry | Latest | Compiles and deploys contracts |
| OpenSSL | Latest | Creates encrypted `.env` files if desired |
| cURL / wget | Latest | Helpful for API calls and network checks |

> **Tip:** While Hardhat is used throughout this guide, the same steps can be adapted for Foundry, Truffle, or Remix.

### 1.3 Accounts and Access
1. **Wallet** – Install a wallet such as [MetaMask](https://metamask.io) or [Rabby](https://rabby.io). Create or import a wallet and securely store the seed phrase offline.
2. **Base Mainnet RPC** – Obtain an RPC endpoint. Options include:
   - [Base-provided public RPC](https://docs.base.org/network-information/#mainnet)
   - Providers such as Alchemy, Infura, QuickNode, Chainstack, or Blast.
3. **Explorer API Key** – Register for a [Basescan](https://basescan.org/apis) API key for contract verification.
4. **ETH on Base** – Bridge ETH from Ethereum mainnet via the [official Base Bridge](https://bridge.base.org) or centralized exchanges that support Base withdrawals.

---

## 2. Understand the Base Network

Base is an Ethereum Layer 2 (L2) built on the OP Stack. Key characteristics:
- **Chain ID:** `8453`
- **Currency Symbol:** `ETH`
- **Block Explorer:** [https://basescan.org](https://basescan.org)
- **Finality:** ~2 seconds, with settlement back to Ethereum L1.
- **Gas Token:** ETH bridged to Base.

Knowing these parameters ensures your deployments target the correct network.

---

## 3. Set Up Your Project

### 3.1 Create a New Hardhat Project
```bash
mkdir base-contract && cd base-contract
npm init -y
npm install --save-dev hardhat
npx hardhat init
```
Choose "Create a JavaScript project" (or TypeScript) when prompted. Hardhat will scaffold directories such as `contracts/`, `scripts/`, and `test/`.

### 3.2 Install Useful Dependencies
```bash
npm install --save-dev @nomicfoundation/hardhat-toolbox dotenv
```
The `@nomicfoundation/hardhat-toolbox` package provides Ethers.js, Waffle, and useful Hardhat plugins. `dotenv` helps manage secrets securely.

### 3.3 Configure Environment Variables
Create a `.env` file (never commit it to version control) and define:
```bash
BASE_RPC_URL="https://base-mainnet.g.alchemy.com/v2/YOUR_KEY"
BASESCAN_API_KEY="your_basescan_key"
DEPLOYER_PRIVATE_KEY="0xYOUR_PRIVATE_KEY"
```
> **Security Reminder:** Never expose your private key. Use hardware wallets or separate deployer accounts with limited funds whenever possible.

### 3.4 Update Hardhat Configuration
Edit `hardhat.config.js` (or `.ts`) to include Base network settings:
```js
require("dotenv").config();
require("@nomicfoundation/hardhat-toolbox");

const { BASE_RPC_URL, DEPLOYER_PRIVATE_KEY, BASESCAN_API_KEY } = process.env;

module.exports = {
  solidity: "0.8.20",
  networks: {
    base: {
      url: BASE_RPC_URL,
      chainId: 8453,
      accounts: DEPLOYER_PRIVATE_KEY ? [DEPLOYER_PRIVATE_KEY] : [],
    },
  },
  etherscan: {
    apiKey: {
      base: BASESCAN_API_KEY,
    },
    customChains: [
      {
        network: "base",
        chainId: 8453,
        urls: {
          apiURL: "https://api.basescan.org/api",
          browserURL: "https://basescan.org",
        },
      },
    ],
  },
};
```

---

## 4. Write and Compile Your Contract

### 4.1 Sample Solidity Contract
Create `contracts/Counter.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Counter {
    uint256 private _count;

    event CountChanged(uint256 newValue);

    function current() external view returns (uint256) {
        return _count;
    }

    function increment() external {
        _count += 1;
        emit CountChanged(_count);
    }

    function reset(uint256 value) external {
        _count = value;
        emit CountChanged(_count);
    }
}
```

### 4.2 Compile the Contract
Run:
```bash
npx hardhat compile
```
Resolve any compiler warnings or errors before deploying.

---

## 5. Fund Your Deployer Account

1. **Bridge ETH to Base** using the [Base Bridge](https://bridge.base.org). Transfer a small amount (e.g., 0.05 ETH) to cover deployment gas and testing transactions.
2. **Verify Receipt** in your wallet on the Base network.
3. **Check Balance** using Hardhat or Etherscan:
```bash
npx hardhat console --network base
> const [signer] = await ethers.getSigners();
> (await signer.getBalance()).toString();
```

---

## 6. Deploy to Base Mainnet

### 6.1 Create a Deployment Script
In `scripts/deploy.js`:
```js
const hre = require("hardhat");

async function main() {
  const Counter = await hre.ethers.getContractFactory("Counter");
  const counter = await Counter.deploy();
  await counter.deployed();

  console.log("Counter deployed to:", counter.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### 6.2 Run the Deployment
```bash
npx hardhat run scripts/deploy.js --network base
```
Monitor the terminal for the contract address. Record it carefully.

### 6.3 Verify Deployment on Basescan
- Open `https://basescan.org/address/<DEPLOYED_ADDRESS>` to confirm your contract exists and that the transaction succeeded.
- Review gas usage and confirm the contract bytecode matches expectations.

---

## 7. Verify Your Contract

### 7.1 Automatic Verification via Hardhat
With the `etherscan` configuration in place, run:
```bash
npx hardhat verify --network base <DEPLOYED_ADDRESS>
```
Hardhat will match the contract metadata to Basescan and publish the verified source.

### 7.2 Manual Verification (Fallback)
If automatic verification fails:
1. Recompile your contract with `npx hardhat clean && npx hardhat compile`.
2. Confirm the constructor arguments and libraries used.
3. Use Basescan's "Verify and Publish" UI, uploading flattened Solidity code or metadata JSON.

---

## 8. Interact with the Contract

### 8.1 Hardhat Console
```bash
npx hardhat console --network base
> const counter = await ethers.getContractAt("Counter", "<DEPLOYED_ADDRESS>");
> await counter.increment();
> (await counter.current()).toString();
```

### 8.2 Scripted Interactions
Create a script (e.g., `scripts/increment.js`) that calls contract functions. This is ideal for automation or multisig-controlled actions.

### 8.3 Front-End Integration
1. Point your DApp to the Base network using the deployed contract address and ABI.
2. Update wallet connection libraries (e.g., Wagmi, RainbowKit) to include Base's chain configuration.
3. Test user flows end-to-end on Base mainnet before launching publicly.

---

## 9. Security and Risk Mitigation

- **Audits & Reviews:** Conduct code reviews and audits before deploying significant value.
- **Testnet First:** Deploy to [Base Sepolia](https://docs.base.org/tools/network-faucets) for end-to-end testing.
- **Gas Management:** Set reasonable gas price limits; Base uses EIP-1559-style fees. Monitor [Base Gas Tracker](https://basescan.org/gastracker).
- **Key Management:** Prefer hardware wallets or multisig setups (e.g., Safe) for contract ownership.
- **Upgrade Safety:** If using proxies, thoroughly test upgrade scripts and add timelocks.
- **Monitoring:** Subscribe to on-chain alerts via Tenderly, Blocknative, or custom bots.

---

## 10. Post-Deployment Checklist

1. ✅ Contract verified on Basescan.
2. ✅ Ownership transferred to production address or multisig.
3. ✅ Admin scripts documented and stored securely.
4. ✅ Event monitoring set up (logs, webhooks, analytics).
5. ✅ Front-end updated with Base contract address.
6. ✅ Announcement and documentation prepared for users.

---

## 11. Troubleshooting Tips

| Issue | Possible Cause | Resolution |
| --- | --- | --- |
| Transaction stuck | Low gas price | Increase `maxFeePerGas` and resubmit with same nonce |
| RPC errors | Rate limits or misconfigured RPC URL | Switch providers or upgrade plan |
| Verification fails | Mismatched compiler settings | Ensure `solc` version and optimizer settings match deploy build |
| Account nonce mismatch | Multiple deployments from same account | Check latest nonce via `ethers.provider.getTransactionCount` |
| Insufficient funds | Not enough bridged ETH | Bridge more ETH or reduce gas usage |

---

## 12. Additional Resources

- [Base Documentation](https://docs.base.org)
- [OP Stack Docs](https://docs.optimism.io)
- [Basescan API Documentation](https://basescan.org/apis)
- [Hardhat Official Docs](https://hardhat.org/getting-started/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts)

---

## 13. Conclusion

With the steps above, you have everything needed to deploy a smart contract to Base mainnet: from preparing your environment and securing funds to verifying and monitoring your contract. Iterate carefully, monitor your contract post-deployment, and keep improving your operational security. Happy building on Base!
