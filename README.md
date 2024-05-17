# Implementing-Account-Abstraction-in-Meta-transactions on zkSync Era


>**zkSync** is a layer 2 solution for Ethereum, enhancing scalability by offering lower gas fees and higher transaction rates while maintaining Ethereum's security. This results in improved user and developer experiences within the Ethereum ecosystem.
Read more about zkSync here.
 [zkSync Official docs](https://docs.zksync.io/build/developer-reference/account-abstraction.html)

In this tutorial, you will learn how to implement meta-transactions using zkSync’s account abstraction. Meta-transactions allow users to interact with the blockchain without having to manage gas fees directly, enhancing the user experience by having a relayer cover these fees.


### Table of Contents:
- [Introduction](#intro)
- [Prerequisite](#prerequisite)
- [Setting up the Environment](#setup) 
- [Account Abstraction](#abstract)
- [Conclusion](#conclusion)

## What is Account Abstraction in zkSync
**Account abstraction in zkSync** combines the features of initiating transactions and implementing arbitrary logic thereby enhancing security and interactions for users and developers within the ecosystem, offering increased flexibility and efficiency in managing transactions and interactions on Ethereum.

## Introduction to Meta-transactions:
Account abstraction in meta-transactions is implemented by separating the process of transaction authorization from the actual execution and payment of gas fees on the blockchain. 

This is how account abstraction is implemented:
- **Users Sign the Transaction:**

Users prepare and sign the transaction details off-chain,using their private key. This step generates a digital signature, ensuring the transaction is authorized by the user.
- **Relayer Submits the Transaction:**

A relayer collects the signed transaction and submits it to the blockchain. In zkSync, L1 relayers are used. The relayer pays the gas fees required to process the transaction.
- **Smart Contract Verifies and Executes:**

The smart contract verifies the user's signature using ECDSA (Elliptic Curve Digital Signature Algorithm). If the signature is valid, the contract executes the function call contained in the transaction.
>We import OpenZeppelin's ECDSA library to use for signature validation.

## [Prerequiste](#prerequiste)

- Make sure your machine satisfies the [system requirements](https://github.com/matter-labs/era-compiler-solidity/tree/main#system-requirements).
- Familiarity with zkSync and its development environment.
- Node.js and Yarn installed on your machine.
- A wallet with Sepolia ETH on zkSync Era Testnet for deployment (You should also know [how to get your private key from your MetaMask wallet](https://support.metamask.io/hc/en-us/articles/360015289632-How-to-export-an-account-s-private-key))

## [Setting up the Environment](#setup) 


#### 1. Project Setup

1. **Open Your Terminal or Command Prompt**.

2. **Create the Project Using zkSync CLI**:
   ```sh
   npx zksync-cli create meta-transaction-tutorial --template hardhat_solidity
   ```

3. **Navigate into the Project Directory**:
   ```sh
   cd meta-transaction-tutorial
   ```

4. **Remove Example Contracts and Deploy Files**:
   ```sh
   rm -rf ./contracts/*
   rm -rf ./deploy/*
   ```

5. **Add Required Libraries**:
   ```sh
   yarn add -D @matterlabs/zksync-contracts @openzeppelin/contracts@4.9.5
   ```

6. **Configure Hardhat for zkSync in `hardhat.config.ts`:**
```typescript
import { HardhatUserConfig } from "hardhat/config";

import "@matterlabs/hardhat-zksync-deploy";
import "@matterlabs/hardhat-zksync-solc";
import "@matterlabs/hardhat-zksync-verify";

// dynamically alters endpoints for local tests
const zkSyncTestnet =
  process.env.NODE_ENV == "test"
    ? {
        url: "http://localhost:3050",
        ethNetwork: "http://localhost:8545",
        zksync: true,
      }
    : {
        url: "https://sepolia.era.zksync.dev",
        ethNetwork: "sepolia",
        zksync: true,
        verifyURL: "https://explorer.sepolia.era.zksync.dev/contract_verification", // Verification endpoint
      };

const config: HardhatUserConfig = {
  zksolc: {
    version: "latest", // Uses latest available in https://github.com/matter-labs/zksolc-bin/
    settings: {},
  },
  defaultNetwork: "zkSyncTestnet",
  networks: {
    hardhat: {
      zksync: false,
    },
    zkSyncTestnet,
  },
  // etherscan: { // Optional - If you plan on verifying a smart contract on Ethereum within the same project
  //   apiKey: //<Your API key for Etherscan>,
  // },
  solidity: {
    version: "0.8.17",
  },
};

export default config;

```

#### 2. Writing the Smart Contract

**Create a contract to handle meta-transactions (`MetaTransaction.sol`):**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MetaTransaction is Ownable {
    using ECDSA for bytes32;

    event MetaTransactionExecuted(address userAddress, address relayerAddress, bytes functionSignature);

    mapping(address => uint256) nonces;

    function getNonce(address user) public view returns (uint256) {
        return nonces[user];
    }

    function executeMetaTransaction(address userAddress, bytes memory functionSignature, bytes memory signature) public payable {
        bytes32 hash = keccak256(abi.encodePacked(userAddress, functionSignature, nonces[userAddress])).toEthSignedMessageHash();
        address signer = hash.recover(signature);
        require(signer == userAddress, "Invalid signature");

        nonces[userAddress]++;

        (bool success, bytes memory data) = address(this).call(abi.encodePacked(functionSignature, userAddress));
        require(success, "Function call not successful");

        emit MetaTransactionExecuted(userAddress, msg.sender, functionSignature);
    }

    function transfer(address to, uint256 amount) public {
        require(msg.sender == owner(), "Only owner can execute");
        payable(to).transfer(amount);
    }

    receive() external payable {}
}
```

#### 3. Deploying the Contract
**Write the deployment script (`scripts/deploy.js`):**
```javascript
const { ethers } = require("hardhat");

async function main() {
  const MetaTransaction = await ethers.getContractFactory("MetaTransaction");
  const metaTransaction = await MetaTransaction.deploy();
  await metaTransaction.deployed();
  console.log("MetaTransaction deployed to:", metaTransaction.address);
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

**Deploy the contract:**
```bash
npx hardhat run scripts/deploy.js --network zkSyncSepoliaTestnet
```

#### 4. Creating the Meta-Transaction Off-chain

**Generate the meta-transaction and signature in a script (`scripts/createMetaTransaction.js`):**
```javascript
const { ethers } = require("ethers");
const MetaTransaction = require("../artifacts/contracts/MetaTransaction.sol/MetaTransaction.json");

async function createMetaTransaction() {
  const provider = new ethers.providers.JsonRpcProvider("https://sepolia.era.zksync.dev");
  const wallet = new ethers.Wallet("YOUR_PRIVATE_KEY", provider);
  const contract = new ethers.Contract("CONTRACT_ADDRESS", MetaTransaction.abi, wallet);

  const nonce = await contract.getNonce(wallet.address);
  const functionSignature = contract.interface.encodeFunctionData("transfer", ["RECIPIENT_ADDRESS", ethers.utils.parseEther("0.1")]);

  const hash = ethers.utils.solidityKeccak256(
    ["address", "bytes", "uint256"],
    [wallet.address, functionSignature, nonce]
  );

  const signature = await wallet.signMessage(ethers.utils.arrayify(hash));

  console.log("Meta-transaction data:", {
    userAddress: wallet.address,
    functionSignature: functionSignature,
    signature: signature,
  });
}

createMetaTransaction();
```

#### 5. Submitting the Meta-Transaction On-chain

**Write the script to submit the meta-transaction (`scripts/submitMetaTransaction.js`):**
```javascript
const { ethers } = require("ethers");
const MetaTransaction = require("../artifacts/contracts/MetaTransaction.sol/MetaTransaction.json");

async function submitMetaTransaction() {
  const provider = new ethers.providers.JsonRpcProvider("https://sepolia.era.zksync.dev");
  const wallet = new ethers.Wallet("YOUR_PRIVATE_KEY", provider);
  const contract = new ethers.Contract("CONTRACT_ADDRESS", MetaTransaction.abi, wallet);

  const metaTxData = {
    userAddress: "USER_ADDRESS",
    functionSignature: "FUNCTION_SIGNATURE",
    signature: "SIGNATURE"
  };

  const tx = await contract.executeMetaTransaction(metaTxData.userAddress, metaTxData.functionSignature, metaTxData.signature, { gasLimit: 1000000 });
  await tx.wait();

  console.log("Meta-transaction submitted:", tx.hash);
}

submitMetaTransaction();
```

### Conclusion
Meta-transactions enable a smoother user experience by abstracting gas fees, making blockchain interactions more accessible. By following this tutorial, developers can implement meta-transactions using zkSync's account abstraction, enhancing the usability of their dApps.

### Additional Resources
- [zkSync Documentation](https://era.zksync.io/docs/)
- [OpenZeppelin Documentation](https://docs.openzeppelin.com/)
- [Ethereum Documentation](https://ethereum.org/en/developers/docs/)

