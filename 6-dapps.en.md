# DAD Lab 6 — Building a Decentralized Application (dApp)

> **Prerequisites:** Lab 3 (CosmWasm fundamentals, project structure, testing, deployment), Lab 5 (state design, `#[cw_serde]`, `addr_make` testing patterns)  
> **Duration:** ~3 hours  
> **Outcome:** Build a complete dApp end-to-end: a counter smart contract deployed on Injective Testnet, plus a React frontend that connects a wallet, reads on-chain state, and broadcasts transactions.

---

## Table of Contents

1. [What Is a dApp?](#1-what-is-a-dapp)
2. [Counter Contract Overview](#2-counter-contract-overview)
3. [Create the Project](#3-create-the-project)
4. [Messages & State](#4-messages--state)
5. [Contract Logic](#5-contract-logic)
6. [Testing](#6-testing)
7. [Optimized Build & Deploy to Injective Testnet](#7-optimized-build--deploy-to-injective-testnet)
8. [Interact on Testnet (CLI)](#8-interact-on-testnet-cli)
9. [Part 2 — Frontend Overview](#9-part-2--frontend-overview)
10. [Scaffold the Frontend](#10-scaffold-the-frontend)
11. [Configuration & Wallet Connection](#11-configuration--wallet-connection)
12. [Querying the Smart Contract](#12-querying-the-smart-contract)
13. [Executing Transactions](#13-executing-transactions)
14. [Exercises & Wrap-Up](#14-exercises--wrap-up)

---

## 1. What Is a dApp?

In the previous labs we wrote smart contracts, compiled them to Wasm, deployed them on Injective Testnet, and interacted with them through the CLI. That works for developers, but end users aren't going to type `injectived tx wasm execute ...` commands. They need a **web interface**.

A **decentralized application (dApp)** is a regular web application with one key difference: instead of talking to your own backend server, it talks to a **blockchain**. The smart contract *is* the backend — it stores the state and enforces the rules. A wallet extension (like Keplr) replaces the traditional login system — users sign transactions with their private keys instead of typing passwords.

### Architecture

```
┌───────────────┐        ┌─────────────┐        ┌─────────────────────┐
│  React App    │──gRPC──│  Injective   │──Wasm──│  Smart Contract     │
│  (browser)    │        │  Node        │        │  (on-chain state)   │
└──────┬────────┘        └─────────────┘        └─────────────────────┘
       │
       │ sign
       ▼
┌───────────────┐
│  Keplr Wallet │
│  (extension)  │
└───────────────┘
```

**Reads (queries)** go directly from the browser to an Injective node via gRPC — no wallet needed, no gas fees, anyone can read.

**Writes (transactions)** require the user to sign with their wallet. The signed transaction is broadcast to the network, where validators execute the smart contract logic and update the state.

### What We'll Build

Today's dApp has two parts:

1. **A counter smart contract** — the simplest possible on-chain state machine. It stores a number. You can increment it or reset it.
2. **A React frontend** — a single page that displays the counter value and lets you click buttons to change it.

Simple? Yes. But this tiny app exercises the full stack: state management, authorization, wallet connection, gRPC queries, transaction signing, and broadcasting. Once you understand this flow, building complex dApps is just more of the same.

---

## 2. Counter Contract Overview

Our counter contract has three capabilities:

| Message | Type | Description |
|---------|------|-------------|
| `InstantiateMsg { count: i32 }` | Instantiate | Sets the initial count and saves the sender as the **owner** |
| `ExecuteMsg::Increment {}` | Execute | Adds 1 to the count (owner only) |
| `ExecuteMsg::Reset { count: i32 }` | Execute | Sets the count to a new value (owner only) |
| `QueryMsg::GetCount {}` | Query | Returns the current count |

**Why owner-only execution?** This is a deliberate design choice. Anyone can *read* the counter, but only the deployer can *change* it. This mirrors a common pattern in production contracts (admin actions, minter roles, governance). It also gives us a reason to write an `Unauthorized` error path.

### State

Just two values in storage:

| Key | Type | Purpose |
|-----|------|---------|
| `COUNT` | `Item<i32>` | The counter value |
| `OWNER` | `Item<Addr>` | The address that deployed the contract |

This is the simplest state we've had — even simpler than Lab 3's admin list.

---

## 3. Create the Project

```bash
cargo new --lib ./dad-counter
cd dad-counter
```

Replace the contents of `Cargo.toml`:

```toml
[package]
name = "dad-counter"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
cosmwasm-std = { version = "2.2", features = ["staking"] }
cosmwasm-schema = "2"
cw-storage-plus = "2"
thiserror = "1"

[dev-dependencies]
cw-multi-test = "2"
```

Create the build alias:

```bash
mkdir .cargo
```

```toml
# .cargo/config.toml
[alias]
wasm = "build --target wasm32-unknown-unknown --release"
wasm-debug = "build --target wasm32-unknown-unknown"
```

Create `src/lib.rs`:

```rust
use cosmwasm_std::{entry_point, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult};

mod contract;
mod error;
mod msg;
mod state;

use error::ContractError;
use msg::{ExecuteMsg, InstantiateMsg, QueryMsg};

#[entry_point]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    contract::instantiate(deps, env, info, msg)
}

#[entry_point]
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    contract::execute(deps, env, info, msg)
}

#[entry_point]
pub fn query(deps: Deps, env: Env, msg: QueryMsg) -> StdResult<Binary> {
    contract::query(deps, env, msg)
}
```

Create empty placeholder files so the project structure is complete:

```bash
touch src/contract.rs src/msg.rs src/state.rs src/error.rs
```

Your project structure:

```
dad-counter/
├── .cargo/
│   └── config.toml
├── src/
│   ├── lib.rs
│   ├── contract.rs    ← (empty)
│   ├── msg.rs         ← (empty)
│   ├── state.rs       ← (empty)
│   └── error.rs       ← (empty)
└── Cargo.toml
```

> The project won't compile yet — `lib.rs` references types and functions that don't exist. We'll fix that in the next section.

---

## 4. Messages & State

### 4.1 Message Types

Create `src/msg.rs`:

```rust
use cosmwasm_schema::{cw_serde, QueryResponses};

#[cw_serde]
pub struct InstantiateMsg {
    pub count: i32,
}

#[cw_serde]
pub enum ExecuteMsg {
    Increment {},
    Reset { count: i32 },
}

#[cw_serde]
#[derive(QueryResponses)]
pub enum QueryMsg {
    #[returns(GetCountResponse)]
    GetCount {},
}

#[cw_serde]
pub struct GetCountResponse {
    pub count: i32,
}
```

All familiar from previous labs. `#[cw_serde]` gives us Serialize, Deserialize, Clone, Debug, PartialEq, and JsonSchema in one attribute. `#[derive(QueryResponses)]` with the `#[returns(...)]` annotations maps each query variant to its response type.

### 4.2 State

Create `src/state.rs`:

```rust
use cosmwasm_std::Addr;
use cw_storage_plus::Item;

pub const COUNT: Item<i32> = Item::new("count");
pub const OWNER: Item<Addr> = Item::new("owner");
```

Two `Item`s — single values stored under fixed keys. No `Map`s needed because we're not tracking per-user data.

### 4.3 Error Type

Create `src/error.rs`:

```rust
use cosmwasm_std::StdError;
use thiserror::Error;

#[derive(Error, Debug, PartialEq)]
pub enum ContractError {
    #[error("{0}")]
    Std(#[from] StdError),

    #[error("Unauthorized")]
    Unauthorized {},
}
```

Just two variants: `Std` for bubbling up standard errors (storage, serialization), and `Unauthorized` for when someone other than the owner tries to execute.

### 4.4 Contract Stubs

To make the project compile, add temporary stubs to `src/contract.rs`:

```rust
use cosmwasm_std::{Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult};

use crate::error::ContractError;
use crate::msg::{ExecuteMsg, InstantiateMsg, QueryMsg};

pub fn instantiate(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    Ok(Response::new())
}

pub fn execute(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    Ok(Response::new())
}

pub fn query(_deps: Deps, _env: Env, _msg: QueryMsg) -> StdResult<Binary> {
    unimplemented!()
}
```

Verify it compiles:

```bash
cargo wasm
```

---

## 5. Contract Logic

Now replace the stubs with real logic. Replace the entire contents of `src/contract.rs`:

```rust
use cosmwasm_std::{
    to_json_binary, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult,
};

use crate::error::ContractError;
use crate::msg::{ExecuteMsg, GetCountResponse, InstantiateMsg, QueryMsg};
use crate::state::{COUNT, OWNER};

pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    COUNT.save(deps.storage, &msg.count)?;
    OWNER.save(deps.storage, &info.sender)?;

    Ok(Response::new()
        .add_attribute("action", "instantiate")
        .add_attribute("owner", info.sender)
        .add_attribute("count", msg.count.to_string()))
}

pub fn execute(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    use ExecuteMsg::*;

    match msg {
        Increment {} => exec::increment(deps, info),
        Reset { count } => exec::reset(deps, info, count),
    }
}

pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        GetCount {} => to_json_binary(&query::get_count(deps)?),
    }
}

mod exec {
    use super::*;

    pub fn increment(deps: DepsMut, info: MessageInfo) -> Result<Response, ContractError> {
        let owner = OWNER.load(deps.storage)?;
        if info.sender != owner {
            return Err(ContractError::Unauthorized {});
        }

        let count = COUNT.load(deps.storage)? + 1;
        COUNT.save(deps.storage, &count)?;

        Ok(Response::new()
            .add_attribute("action", "increment")
            .add_attribute("count", count.to_string()))
    }

    pub fn reset(
        deps: DepsMut,
        info: MessageInfo,
        count: i32,
    ) -> Result<Response, ContractError> {
        let owner = OWNER.load(deps.storage)?;
        if info.sender != owner {
            return Err(ContractError::Unauthorized {});
        }

        COUNT.save(deps.storage, &count)?;

        Ok(Response::new()
            .add_attribute("action", "reset")
            .add_attribute("count", count.to_string()))
    }
}

mod query {
    use super::*;

    pub fn get_count(deps: Deps) -> StdResult<GetCountResponse> {
        let count = COUNT.load(deps.storage)?;
        Ok(GetCountResponse { count })
    }
}
```

Walk through the key parts:

- **`instantiate`** saves both the initial `count` and the `info.sender` as `OWNER`. Whoever deploys the contract becomes its owner.

- **`exec::increment`** and **`exec::reset`** both start with the same authorization check: load the owner, compare to `info.sender`, reject with `Unauthorized` if they don't match. Then they update `COUNT`.

- **`query::get_count`** just loads and returns the current count. Queries are read-only — no `info` (no sender), and they return `StdResult` instead of `Result<_, ContractError>`.

Verify:

```bash
cargo wasm
```

---

## 6. Testing

Add the test module at the bottom of `src/contract.rs`:

```rust
#[cfg(test)]
mod tests {
    use cosmwasm_std::Addr;
    use cw_multi_test::{App, ContractWrapper, Executor};

    use super::*;
    use crate::msg::{GetCountResponse, InstantiateMsg};

    fn setup_contract(app: &mut App, initial_count: i32) -> (Addr, Addr) {
        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let owner = app.api().addr_make("owner");

        let contract_addr = app
            .instantiate_contract(
                code_id,
                owner.clone(),
                &InstantiateMsg {
                    count: initial_count,
                },
                &[],
                "Counter",
                None,
            )
            .unwrap();

        (contract_addr, owner)
    }

    fn query_count(app: &App, contract_addr: &Addr) -> i32 {
        let resp: GetCountResponse = app
            .wrap()
            .query_wasm_smart(contract_addr, &QueryMsg::GetCount {})
            .unwrap();
        resp.count
    }

    #[test]
    fn instantiate_sets_count() {
        let mut app = App::default();
        let (contract_addr, _owner) = setup_contract(&mut app, 17);

        assert_eq!(query_count(&app, &contract_addr), 17);
    }

    #[test]
    fn increment_adds_one() {
        let mut app = App::default();
        let (contract_addr, owner) = setup_contract(&mut app, 0);

        app.execute_contract(
            owner,
            contract_addr.clone(),
            &ExecuteMsg::Increment {},
            &[],
        )
        .unwrap();

        assert_eq!(query_count(&app, &contract_addr), 1);
    }

    #[test]
    fn reset_sets_count() {
        let mut app = App::default();
        let (contract_addr, owner) = setup_contract(&mut app, 0);

        // Increment a few times first
        for _ in 0..3 {
            app.execute_contract(
                owner.clone(),
                contract_addr.clone(),
                &ExecuteMsg::Increment {},
                &[],
            )
            .unwrap();
        }
        assert_eq!(query_count(&app, &contract_addr), 3);

        // Reset to 5
        app.execute_contract(
            owner,
            contract_addr.clone(),
            &ExecuteMsg::Reset { count: 5 },
            &[],
        )
        .unwrap();

        assert_eq!(query_count(&app, &contract_addr), 5);
    }

    #[test]
    fn unauthorized_reset_fails() {
        let mut app = App::default();
        let (contract_addr, _owner) = setup_contract(&mut app, 0);

        let stranger = app.api().addr_make("stranger");

        let err = app
            .execute_contract(
                stranger,
                contract_addr,
                &ExecuteMsg::Reset { count: 99 },
                &[],
            )
            .unwrap_err();

        assert!(err.root_cause().to_string().contains("Unauthorized"));
    }
}
```

We use the same testing patterns from Labs 4 and 5:
- `App::default()` creates an in-memory blockchain.
- `setup_contract` is a helper that stores the code, instantiates the contract, and returns both addresses.
- `query_count` is a helper that wraps the query boilerplate.
- `addr_make("stranger")` creates a deterministic test address that is different from the owner.

Run the tests:

```bash
cargo test
```

All four should pass. Try breaking the authorization check (comment out the `if info.sender != owner` block) and re-run — you'll see `unauthorized_reset_fails` catch the bug.

---

## 7. Optimized Build & Deploy to Injective Testnet

### 7.1 Optimized Build with Docker

```bash
# From the project root (dad-counter/)

# For x86/amd64 machines:
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/optimizer:0.17.0

# For Apple Silicon / ARM64:
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/optimizer-arm64:0.17.0
```

This creates:

```
artifacts/
├── dad_counter.wasm    ← the optimized binary
└── checksums.txt
```

Verify:

```bash
cosmwasm-check artifacts/dad_counter.wasm
```

### 7.2 Set Up Environment Variables

```bash
export INJ_ADDRESS=<your_inj_testnet_address>
export NODE="https://testnet.sentry.tm.injective.network:443"
export CHAIN_ID="injective-888"
```

Make sure you have testnet funds from the [Injective Faucet](https://faucet.injective.network/).

### 7.3 Upload (Store) the Wasm Code

```bash
injectived tx wasm store artifacts/dad_counter.wasm \
  --from=$INJ_ADDRESS \
  --chain-id=$CHAIN_ID \
  --yes \
  --fees=1000000000000000inj \
  --gas=2000000 \
  --node=$NODE
```

Note the `txhash` from the output, then query it:

```bash
injectived query tx <TXHASH> --node=$NODE
```

Look for `code_id` in the event attributes and export it:

```bash
export CODE_ID=<your_code_id>
```

### 7.4 Instantiate the Contract

```bash
INIT='{"count":0}'

injectived tx wasm instantiate $CODE_ID "$INIT" \
  --label="DadCounter" \
  --from=$INJ_ADDRESS \
  --chain-id=$CHAIN_ID \
  --yes \
  --fees=1000000000000000inj \
  --gas=2000000 \
  --no-admin \
  --node=$NODE
```

Query the transaction to find the contract address:

```bash
injectived query tx <TXHASH> --node=$NODE
```

Look for the `_contract_address` attribute. Export it:

```bash
export CONTRACT=<your_contract_address>
```

> **Save this address.** You'll need it in Part 2 when configuring the React frontend.

---

## 8. Interact on Testnet (CLI)

Before building the frontend, let's verify the contract works on-chain.

### 8.1 Query the Count

```bash
injectived query wasm contract-state smart $CONTRACT '{"get_count":{}}' \
  --node=$NODE \
  --output json
```

Expected:

```json
{"data":{"count":0}}
```

### 8.2 Increment

```bash
injectived tx wasm execute $CONTRACT '{"increment":{}}' \
  --from=$INJ_ADDRESS \
  --chain-id=$CHAIN_ID \
  --yes \
  --fees=1000000000000000inj \
  --gas=2000000 \
  --node=$NODE
```

Query again — the count should be `1`.

### 8.3 Reset

```bash
injectived tx wasm execute $CONTRACT '{"reset":{"count":42}}' \
  --from=$INJ_ADDRESS \
  --chain-id=$CHAIN_ID \
  --yes \
  --fees=1000000000000000inj \
  --gas=2000000 \
  --node=$NODE
```

Query again — the count should be `42`.

### 8.4 Test Unauthorized Access

Have a classmate (whose address is not the owner) try to increment or reset your contract. They should get an **Unauthorized** error.

---

Your contract is live and verified. Now let's build a web UI so anyone with a wallet can interact with it — no CLI required.

---

## 9. Part 2 — Frontend Overview

### The Injective TypeScript SDK

Injective provides a suite of TypeScript packages for building dApps. We'll use four of them:

| Package | What it does |
|---------|-------------|
| `@injectivelabs/sdk-ts` | Core SDK — message types (`MsgExecuteContractCompat`), gRPC client (`ChainGrpcWasmApi`), address utilities |
| `@injectivelabs/wallet-core` | **BaseWalletStrategy** and **MsgBroadcaster** — the wallet abstraction layer and transaction broadcaster |
| `@injectivelabs/wallet-cosmos` | **CosmosWalletStrategy** — concrete wallet implementation for Cosmos wallets (Keplr, Leap, etc.) |
| `@injectivelabs/wallet-base` | **Wallet** enum — wallet type identifiers |
| `@injectivelabs/networks` | Network helpers — resolves gRPC/REST/RPC endpoints for testnet or mainnet |

### The WalletStrategy Pattern

The Injective SDK uses a **strategy pattern** for wallet integration. Instead of coupling your app to a specific wallet, you:

1. Create **concrete strategies** for each wallet you want to support (e.g., `CosmosWalletStrategy` for Keplr).
2. Pass them into a **`BaseWalletStrategy`** which provides a unified interface.
3. Call generic methods like `walletStrategy.getAddresses()` and `walletStrategy.setWallet(Wallet.Keplr)`.

The `MsgBroadcaster` takes a `BaseWalletStrategy` and handles the full sign-and-broadcast flow: it simulates the transaction for gas estimation, asks the wallet to sign, then submits the signed transaction to the network.

### Data Flow

```
┌─────────────┐  query (gRPC)  ┌──────────────┐
│  React App  │───────────────►│  Injective   │
│             │                │  Node        │
│             │  broadcast tx  │              │
│             │───────────────►│              │
└──────┬──────┘                └──────────────┘
       │ sign
       ▼
┌─────────────┐
│   Keplr     │
└─────────────┘
```

- **Queries** go directly to the node — no wallet involved, no gas cost.
- **Transactions** are built by the app, signed by Keplr, then broadcast to the node.

### What We'll Build

A single `App.tsx` component with:
1. A "Connect Wallet" button
2. A large counter display (queried from the smart contract)
3. An "Increment" button and a "Reset (0)" button

All configuration lives in `config.ts`. That's it — two source files.

---

## 10. Scaffold the Frontend

### 10.1 Create the Vite Project

From the parent directory (not inside `dad-counter/`):

```bash
cd ..
npm create vite@latest counter-dapp -- --template react-ts
cd counter-dapp
npm install
```

### 10.2 Install Injective SDK Packages

```bash
npm install @injectivelabs/sdk-ts @injectivelabs/wallet-core @injectivelabs/wallet-cosmos @injectivelabs/wallet-base @injectivelabs/networks
```

The Injective SDK uses some Node.js APIs internally (like `Buffer`). In a browser, we need polyfills. Install the Vite plugin:

```bash
npm install -D vite-plugin-node-polyfills
```

### 10.3 Configure Vite

Replace the contents of `vite.config.ts`:

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { nodePolyfills } from "vite-plugin-node-polyfills";

export default defineConfig({
  plugins: [react(), nodePolyfills()],
});
```

### 10.4 Clean Up Boilerplate

Delete the files we don't need:

```bash
rm src/App.css src/index.css
rm -rf src/assets
```

Replace `src/main.tsx` with:

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

Replace `src/App.tsx` with a minimal skeleton:

```tsx
function App() {
  return (
    <div style={{ textAlign: "center", fontFamily: "sans-serif", marginTop: "10vh" }}>
      <h1>Counter dApp</h1>
      <p>Coming soon...</p>
    </div>
  );
}

export default App;
```

### 10.5 Verify

```bash
npm run dev
```

Open the URL shown in the terminal (usually `http://localhost:5173`). You should see "Counter dApp" centered on the page.

> **Prerequisite for the next sections:** Install the [Keplr wallet extension](https://www.keplr.app/) in your browser and import or create a testnet wallet. Make sure it has testnet INJ from the [faucet](https://faucet.injective.network/).

---

## 11. Configuration & Wallet Connection

### 11.1 Config File

Create `src/config.ts`:

```typescript
import { getNetworkEndpoints, Network } from "@injectivelabs/networks";

export const NETWORK = Network.Testnet;
export const ENDPOINTS = getNetworkEndpoints(NETWORK);
export const CHAIN_ID = "injective-888";
export const CONTRACT_ADDRESS = "<paste_your_contract_address_here>";
```

Replace `<paste_your_contract_address_here>` with the contract address you exported as `$CONTRACT` in Section 7.4.

> **Why a config file?** Hardcoding chain details and contract addresses throughout the app is fragile. A single config file makes it easy to switch between testnet and mainnet, or to point to a different contract instance.

### 11.2 Wallet Connection

Update `src/App.tsx`:

```tsx
import { useState } from "react";
import { Wallet } from "@injectivelabs/wallet-base";
import { CosmosWalletStrategy } from "@injectivelabs/wallet-cosmos";
import { BaseWalletStrategy } from "@injectivelabs/wallet-core";
import { CHAIN_ID } from "./config";

const walletStrategy = new BaseWalletStrategy({
  chainId: CHAIN_ID,
  strategies: {
    [Wallet.Keplr]: new CosmosWalletStrategy({
      chainId: CHAIN_ID,
      wallet: Wallet.Keplr,
    }),
  },
});
walletStrategy.setWallet(Wallet.Keplr);

function App() {
  const [address, setAddress] = useState("");

  const connectWallet = async () => {
    const addresses = await walletStrategy.getAddresses();
    if (addresses.length > 0) {
      setAddress(addresses[0]);
    }
  };

  return (
    <div style={{ textAlign: "center", marginTop: "10vh" }}>
      <h1>Counter dApp</h1>

      {!address ? (
        <button onClick={connectWallet} style={buttonStyle}>
          Connect Wallet
        </button>
      ) : (
        <p>Connected: {address}</p>
      )}
    </div>
  );
}

const buttonStyle: React.CSSProperties = {
  padding: "12px 24px",
  fontSize: "1.1rem",
  cursor: "pointer",
  borderRadius: "8px",
  border: "1px solid #ccc",
  background: "#f5f5f5",
  margin: "0 8px",
};

export default App;
```

Let's walk through the new code:

- **`CosmosWalletStrategy`** is the concrete strategy for Cosmos-native wallets like Keplr. You create one instance per wallet type you want to support.
- **`BaseWalletStrategy`** wraps the concrete strategies in a unified interface. The `strategies` map tells it which wallet types are available and how to handle each one.
- **`walletStrategy.setWallet(Wallet.Keplr)`** tells it which strategy to use. If you added more wallets to the `strategies` map, you could let the user choose and call `setWallet` accordingly.
- **`walletStrategy.getAddresses()`** triggers the Keplr popup. The user approves the connection, and we get back their `inj1...` address.
- The UI conditionally renders: if no address, show "Connect Wallet"; otherwise, show the connected address.

Try it out — click "Connect Wallet" and approve in Keplr. Your Injective address should appear.

> If Keplr doesn't recognize the Injective testnet, it will prompt you to add the chain automatically.

---

## 12. Querying the Smart Contract

Now we'll read the counter value from the blockchain and display it.

Update `src/App.tsx`:

```tsx
import { useState, useEffect } from "react";
import { Wallet } from "@injectivelabs/wallet-base";
import { CosmosWalletStrategy } from "@injectivelabs/wallet-cosmos";
import { BaseWalletStrategy } from "@injectivelabs/wallet-core";
import { ChainGrpcWasmApi } from "@injectivelabs/sdk-ts";
import { CHAIN_ID, ENDPOINTS, CONTRACT_ADDRESS } from "./config";

const walletStrategy = new BaseWalletStrategy({
  chainId: CHAIN_ID,
  strategies: {
    [Wallet.Keplr]: new CosmosWalletStrategy({
      chainId: CHAIN_ID,
      wallet: Wallet.Keplr,
    }),
  },
});
walletStrategy.setWallet(Wallet.Keplr);

const chainGrpcWasmApi = new ChainGrpcWasmApi(ENDPOINTS.grpc);

const toBase64 = (obj: object) =>
  Buffer.from(JSON.stringify(obj)).toString("base64");

function App() {
  const [address, setAddress] = useState("");
  const [count, setCount] = useState<number | null>(null);

  const fetchCount = async () => {
    const response = await chainGrpcWasmApi.fetchSmartContractState(
      CONTRACT_ADDRESS,
      toBase64({ get_count: {} })
    );

    const result = JSON.parse(
      Buffer.from(response.data).toString()
    );
    setCount(result.count);
  };

  // Fetch count on page load
  useEffect(() => {
    fetchCount();
  }, []);

  const connectWallet = async () => {
    const addresses = await walletStrategy.getAddresses();
    if (addresses.length > 0) {
      setAddress(addresses[0]);
    }
  };

  return (
    <div style={{ textAlign: "center", marginTop: "10vh" }}>
      <h1>Counter dApp</h1>

      {!address ? (
        <button onClick={connectWallet} style={buttonStyle}>
          Connect Wallet
        </button>
      ) : (
        <p>Connected: {address}</p>
      )}

      <p style={{ fontSize: "8rem", fontWeight: "bold", margin: "0.2em 0" }}>
        {count !== null ? count : "—"}
      </p>
    </div>
  );
}

const buttonStyle: React.CSSProperties = {
  padding: "12px 24px",
  fontSize: "1.1rem",
  cursor: "pointer",
  borderRadius: "8px",
  border: "1px solid #ccc",
  background: "#f5f5f5",
  margin: "0 8px",
};

export default App;
```

What's new:

- **`ChainGrpcWasmApi`** — the gRPC client for querying Wasm smart contracts. Created once at module level with the testnet gRPC endpoint.
- **`toBase64`** — a helper that converts a JavaScript object to a base64-encoded JSON string. CosmWasm queries are serialized as JSON, and the gRPC API expects them in base64.
- **`fetchCount`** — sends a `get_count` query to the contract. The response data comes back as bytes, which we decode to JSON and extract the `count` field.
- **`useEffect`** — calls `fetchCount` when the page loads. This runs *without* wallet connection — queries are public and free.
- The counter is displayed in large text (`8rem`). While loading, it shows "—".

> **Notice:** the counter displays immediately, even before connecting a wallet. This is because queries don't need a wallet — they're read-only operations on public blockchain state.

---

## 13. Executing Transactions

The final step: add the Increment and Reset buttons that broadcast transactions.

Replace `src/App.tsx` with the complete final version:

```tsx
import { useState, useEffect } from "react";
import { Wallet } from "@injectivelabs/wallet-base";
import { CosmosWalletStrategy } from "@injectivelabs/wallet-cosmos";
import { BaseWalletStrategy, MsgBroadcaster } from "@injectivelabs/wallet-core";
import {
  ChainGrpcWasmApi,
  MsgExecuteContractCompat,
} from "@injectivelabs/sdk-ts";
import { CHAIN_ID, NETWORK, ENDPOINTS, CONTRACT_ADDRESS } from "./config";

// --- SDK setup (module-level, created once) ---

const walletStrategy = new BaseWalletStrategy({
  chainId: CHAIN_ID,
  strategies: {
    [Wallet.Keplr]: new CosmosWalletStrategy({
      chainId: CHAIN_ID,
      wallet: Wallet.Keplr,
    }),
  },
});
walletStrategy.setWallet(Wallet.Keplr);

const msgBroadcaster = new MsgBroadcaster({
  walletStrategy,
  network: NETWORK,
  simulateTx: true,
});

const chainGrpcWasmApi = new ChainGrpcWasmApi(ENDPOINTS.grpc);

const toBase64 = (obj: object) =>
  Buffer.from(JSON.stringify(obj)).toString("base64");

// --- React component ---

function App() {
  const [address, setAddress] = useState("");
  const [count, setCount] = useState<number | null>(null);
  const [loading, setLoading] = useState(false);

  const fetchCount = async () => {
    const response = await chainGrpcWasmApi.fetchSmartContractState(
      CONTRACT_ADDRESS,
      toBase64({ get_count: {} })
    );

    const result = JSON.parse(Buffer.from(response.data).toString());
    setCount(result.count);
  };

  useEffect(() => {
    fetchCount();
  }, []);

  const connectWallet = async () => {
    const addresses = await walletStrategy.getAddresses();
    if (addresses.length > 0) {
      setAddress(addresses[0]);
    }
  };

  const handleIncrement = async () => {
    setLoading(true);
    try {
      const msg = MsgExecuteContractCompat.fromJSON({
        contractAddress: CONTRACT_ADDRESS,
        sender: address,
        msg: { increment: {} },
      });

      await msgBroadcaster.broadcast({
        msgs: msg,
        injectiveAddress: address,
      });

      await fetchCount();
    } catch (e) {
      console.error("Increment failed:", e);
      alert("Transaction failed. Check the console for details.");
    } finally {
      setLoading(false);
    }
  };

  const handleReset = async () => {
    setLoading(true);
    try {
      const msg = MsgExecuteContractCompat.fromJSON({
        contractAddress: CONTRACT_ADDRESS,
        sender: address,
        msg: { reset: { count: 0 } },
      });

      await msgBroadcaster.broadcast({
        msgs: msg,
        injectiveAddress: address,
      });

      await fetchCount();
    } catch (e) {
      console.error("Reset failed:", e);
      alert("Transaction failed. Check the console for details.");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div style={{ textAlign: "center", marginTop: "10vh" }}>
      <h1>Counter dApp</h1>

      {!address ? (
        <button onClick={connectWallet} style={buttonStyle}>
          Connect Wallet
        </button>
      ) : (
        <p>Connected: {address}</p>
      )}

      <p style={{ fontSize: "8rem", fontWeight: "bold", margin: "0.2em 0" }}>
        {count !== null ? count : "—"}
      </p>

      <div>
        <button
          onClick={handleIncrement}
          disabled={!address || loading}
          style={buttonStyle}
        >
          {loading ? "Sending TX" : "Increment"}
        </button>
        <button
          onClick={handleReset}
          disabled={!address || loading}
          style={buttonStyle}
        >
          {loading ? "Sending TX" : "Reset to 0"}
        </button>
      </div>
    </div>
  );
}

const buttonStyle: React.CSSProperties = {
  padding: "12px 24px",
  fontSize: "1.1rem",
  cursor: "pointer",
  borderRadius: "8px",
  border: "1px solid #ccc",
  background: "#f5f5f5",
  margin: "0 8px",
};

export default App;
```

The new pieces:

- **`MsgBroadcaster`** is created with our `walletStrategy` and network. The `simulateTx: true` option makes it simulate the transaction before broadcasting — this catches errors (like `Unauthorized`) before spending gas.

- **`MsgExecuteContractCompat.fromJSON`** builds a Wasm execute message. The `msg` field is the JSON body that maps directly to our Rust `ExecuteMsg` variants. `{ increment: {} }` becomes `ExecuteMsg::Increment {}` on the contract side.

- **`msgBroadcaster.broadcast`** does three things:
  1. Simulates the transaction (gas estimation)
  2. Opens Keplr for the user to sign
  3. Broadcasts the signed transaction to the network

- **After broadcasting**, we call `fetchCount()` again to refresh the displayed value.

- **`loading` state** disables both buttons while a transaction is in-flight, preventing double-submits. The button text changes to "Sending TX" as a visual indicator.

- **Buttons are disabled** when no wallet is connected (`!address`). You can't send transactions without a connected wallet.

### Try It Out

1. Make sure `npm run dev` is running.
2. Open the page — the counter value loads from the chain immediately.
3. Click "Connect Wallet" — approve in Keplr.
4. Click "Increment" — Keplr pops up for signing. Approve, wait a few seconds for the transaction to confirm, and the counter updates.
5. Click "Reset (0)" — same flow, counter goes back to 0.

> **If you get an "Unauthorized" error on Reset:** make sure your Keplr wallet is using the same address that deployed the contract. Only the owner can reset.

---

## 14. Exercises & Wrap-Up

### What We Covered

- **Counter smart contract** — `InstantiateMsg`, `ExecuteMsg` (with owner authorization), `QueryMsg`, tested with `cw-multi-test`.
- **Deployment to Injective Testnet** — Docker optimizer, `injectived tx wasm store`, `injectived tx wasm instantiate`.
- **React + TypeScript frontend** — Vite scaffolding, project setup.
- **Wallet connection** — `BaseWalletStrategy` with `CosmosWalletStrategy` for Keplr.
- **On-chain queries** — `ChainGrpcWasmApi.fetchSmartContractState` with base64 encoding.
- **Transaction broadcasting** — `MsgBroadcaster` + `MsgExecuteContractCompat`.

### Optional Exercises (Take-Home)

1. **Decrement** — Add a `Decrement {}` variant to `ExecuteMsg` and a corresponding button in the UI. Don't forget to test the new handler.

2. **Display owner** — Add a `GetOwner {}` query to the contract that returns the owner address. Display it in the frontend below the counter.

3. **Transaction feedback** — After a successful broadcast, show the transaction hash with a link to the [Injective Testnet Explorer](https://testnet.explorer.injective.network/). Hint: `msgBroadcaster.broadcast` returns a response with a `txHash` field.

4. **Error handling** — Instead of a generic `alert`, display a user-friendly message when a transaction fails. Specifically, detect the `Unauthorized` error and show "Only the contract owner can perform this action."

---

## Quick Reference — File Structure

```
dad-counter/                     counter-dapp/
├── .cargo/                      ├── src/
│   └── config.toml              │   ├── App.tsx        ← UI + wallet + queries + transactions
├── artifacts/                   │   ├── config.ts      ← network & contract configuration
│   └── dad_counter.wasm         │   ├── main.tsx       ← React entry point
├── src/                         │   └── vite-env.d.ts
│   ├── lib.rs                   ├── index.html
│   ├── contract.rs              ├── vite.config.ts
│   ├── msg.rs                   ├── tsconfig.json
│   ├── state.rs                 └── package.json
│   └── error.rs
└── Cargo.toml
```

---

## Resources

- [Injective TypeScript SDK](https://github.com/InjectiveLabs/injective-ts) — source code and examples
- [Injective Wallet Strategy Docs](https://docs.injective.network/developers-native/wallets/strategy) — wallet connection reference
- [CosmWasm Book](https://book.cosmwasm.com/) — smart contract fundamentals
- [Vite Documentation](https://vite.dev/) — frontend build tool
- [Keplr Wallet](https://www.keplr.app/) — browser extension for Cosmos chains
- [Injective Testnet Faucet](https://faucet.injective.network/) — get testnet INJ
- [Injective Testnet Explorer](https://testnet.explorer.injective.network/) — view transactions and contracts
