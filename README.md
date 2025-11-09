# Drosera Mainnet Trap Guide

This repo is a **step-by-step** guide to:

- build a Drosera Trap in Solidity,
- configure it for **Ethereum Mainnet**,
- deploy it via `drosera` CLI,
- run your own **Drosera Operator**,



The included `example-trap` is intentionally simple, so you can replace it with **any custom logic** (arb, invariants, oracle checks, protocol-specific conditions, etc.).

---

## 0. What is Drosera (Quick)

Very short version:

- A **Trap** is a smart contract that defines conditions, evaluated off-chain by Operators each block.
- A **Response Contract** is the on-chain function Operators call when the Trap says “yes, respond”.
- An **Operator** is a node (`drosera-operator`) that:
  - fetches Trap config from Drosera,
  - runs `collect` + `shouldRespond`,
  - submits responses when conditions are met.

For your trap to actually “run” on Ethereum mainnet:

> You must have at least one healthy Operator opted-in to your Trap.

Official docs (read later if you like):
- Drosera docs: `dev.drosera.io` 
- Mainnet Drosera address announcement 

---

## 1. Requirements

### You’ll need

- A Linux server or VPS recommended (Ubuntu 20.04+)
- Some **ETH on Ethereum mainnet**

### Two wallets (best practise)

Use two separate keys:

1. **TRAP_OWNER**
   - Deploys Trap & Response contracts
   - Owns / updates `drosera.toml`
2. **OPERATOR**
   - Runs the Drosera Operator node
   - Is whitelisted if the Trap is private

   Or you can just use the same wallet.


## 2. Install Dependencies

   - Install Foundry:
   
   ```bash
   curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup
   ```

  - Verify

    ```bash
    forge --version
    ```
  
  - Initialize Drosera project

  ```bash
  mkdir my-drosera-trap
cd my-drosera-trap
forge init -t drosera-network/trap-foundry-template
```

  - Install Drosera CLI

  ```bash
  curl -L https://app.drosera.io/install | bash
source ~/.bashrc
droseraup
 ```

 - Verify

 ```bash
drosera --help
 ```

- Drosera Operator

```bash
curl -LO https://github.com/drosera-network/releases/releases/download/v1.25.1/drosera-operator-v1.25.1-x86_64-unknown-linux-gnu.tar.gz

tar -xvf drosera-operator-v1.25.1-x86_64-unknown-linux-gnu.tar.gz

sudo mv drosera-operator /usr/local/bin/

drosera-operator --help
```

## 3. Mainnet constants

- Drosera mainnet contract address:
```text
0x01C344b8406c3237a6b9dbd06ef2832142866d87
```

- You need an Ethereum mainnet RPC: 
  You can get one at ankr: https://www.ankr.com/rpc/?utm_referral=LV5ZYDqpPk
  Or Alchemy: https://dashboard.alchemy.com/


## 4. Writing your Trap's Smart Contract:

You only ever need two on-chain pieces for a Drosera trap:

- Trap contract – implements Drosera’s ITrap interface.

- Response contract – exposes the function Drosera calls when the trap fires.

  # Mental model: what is a trap?

  A Drosera trap is pure logic + config, evaluated by off-chain operators, anchored on-chain.

    * For every trap:
    
    - Operators:
    
      - call collect() over time to build a history of encoded data.
    
      - call shouldRespond() with that history.
    
      - if shouldRespond() returns (true, payload), they call your response contract with that payload.

    * So your job as a trap author:

    - Decide what condition you want to detect.

    - Encode it as:

    - a collect() that gathers the relevant on-chain data.

    - a shouldRespond() that decides, purely from that data, if “something happened”.

    - Decide what to do when it triggers, that’s your response contract.


# An EXAMPLE of a trap skeleton that implements ITrap(The Drosera Trap Interface):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ITrap} from "@drosera/contracts/src/interfaces/ITrap.sol";

contract MyTrap is ITrap {
    function collect(bytes[] calldata data) external view override returns (bytes memory) {
        // read on-chain state only
    }

    function shouldRespond(bytes[] calldata data)
        external
        pure
        override
        returns (bool, bytes memory)
    {
        // pure logic on data
    }
}
```

## KEY RULES:

- collect must be *external view*.

  - Can read state.

  - Cannot write state.

- shouldRespond must be external pure.

  - No state reads, no storage, no block.timestamp, no msg.sender.

  - Only uses the data argument.

- No constructor args for the trap contract.

  -If you need constants (addresses, thresholds), hardcode them or use immutable with no constructor inputs, Drosera expects a predictable ABI.

- Deterministic:

  - Given the same data, all operators must get the same answer.



## Response contract:

This is whatever you want called when your trap fires.

EXAMPLES:

- Emit an event (for bots / off-chain infra)
- Call your protocol’s pause
- Start a safe / timelock action
- Trigger your arb bot hook(IN MY CASE).

# Here's a minimal one to build on

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MyResponder {
    event TrapFired(bytes data);

    function handleTrap(bytes calldata data) external {
        // In production: add access control!
        emit TrapFired(data);
    }
}
```

You then need to tell Drosera about this function in drosera.toml:

## Compile your contracts by running

```bash
forge build
```

If it throws no errors, your contracts satisfy the compiler conditions.


## 5. Deploy Your Response Contract

  We assume at this point:

  - Your trap + responder compile with forge build

  - You’re inside the trap project directory (e.g. example-trap/)

```bash
forge create \
  --rpc-url $ETH_RPC_URL \
  --private-key $TRAP_OWNER_PRIVATE_KEY \
  src/ExampleResponder.sol:ExampleResponder \
  --broadcast
```

*ETH_RPC_URL* = replace with your own ETH mainnet RPC
*TRAP_OWNER_PRIVATE_KEY* = replace with your TRAP OWNER wallet private key
*src/ExampleResponder.sol:ExampleResponder* = replace with your own real path

This prints something like:
```text
Deployed to: 0xYourResponseContractAddress
```

Copy that address; we’ll plug it into drosera.toml next.


## 6. Configure drosera.toml

drosera.toml tells Drosera:
- which trap artifact to use,

- which response function to call,

- and which operators may serve it.


# Open drosera.toml
```bash
nano drosera.toml
```
Set it up like this:
```toml
[network]
drosera_address = "0x01C344b8406c3237a6b9dbd06ef2832142866d87"

[traps.example_trap]
path = "out/ExampleTrap.sol/ExampleTrap.json" #replace with your real path to your compiled trap artifact from forge build
response_contract = "0xYourResponseContractAddress"
response_function = "Your Function Signature"
cooldown_period_blocks = 1
min_number_of_operators = 1
max_number_of_operators = 5
block_sample_size = 1
private_trap = true
address = "0xYourTrapAddress"

whitelist = ["Your Operator/s' wallet address"]
```

Then Ctrl+X+Y then Enter to save your file.


# Run drosera apply

```bash
DROSERA_PRIVATE_KEY=$YOUR_TRAP_OWNER_PRIVATE_KEY drosera apply --eth-rpc-url $YOUR_ETH_RPC
```
Replace the placeholders with your own details:
$YOUR_TRAP_OWNER_PRIVATE_KEY = Your Private Key
$YOUR_ETH_RPC = Your ETH Mainnet RPC

*What happens:*

If this is the first time:

  - Drosera deploys or registers your trap.

  - It may print the trap address.

  - If you already know your trap address:

  - Add it under address = "0xYourTrapAddress" in your drosera.toml file

  - Run drosera apply again to sync.


*At this point:*

- Your trap is registered on mainnet.

- Drosera’s config knows which trap + responder to use.

- But it won’t actually run until we add operators. 


## 7. Register the operator

```bash
drosera-operator register \
  --eth-rpc-url $ETH_RPC_URL \
  --eth-private-key $OPERATOR_PRIVATE_KEY \
  --drosera-address 0x01C344b8406c3237a6b9dbd06ef2832142866d87
```

Replace the placeholders *$ETH_RPC_URL* and *$OPERATOR_PRIVATE_KEY* with your actual ETH RPC and Operator wallet private key.

If you see:
```text
OperatorRegistered
```
That's a good thing. It was successful.

## Visit the Drosera Explorer to opt-in your trap with your Operator wallet.

- Visit: https://app.drosera.io/
- Connect Wallet and search for your deployed trap.
- Click on opt-in, then sign the message with your wallet.

Make sure you have whitelisted your Operator in your drosera.toml file or else this process will fail.

## 8. Run the Operator Node

On the machine that will run your operator:

- Open ports

```bash
sudo ufw allow 40113/tcp
sudo ufw allow 40114/tcp
```

- Run the node

```bash
drosera-operator node \
  --eth-rpc-url $ETH_RPC_URL \
  --eth-private-key $OPERATOR_PRIVATE_KEY \
  --drosera-address $DROSERA_ADDRESS \
  --listen-address 0.0.0.0 \
  --network-p2p-port 40113 \
  --server-port 40114 \
  --network-external-p2p-address $PUBLIC_IP
  ```

Replace all Placeholders with your real value.

You want to see logs like:
```text
Operator Node successfully spawned!
```

Keep your Operator Node running!!
