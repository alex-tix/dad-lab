# DAD Lab 3 — Introduction to CosmWasm Smart Contracts

> **Prerequisites:** Lab 1 (Rust fundamentals), Lab 2 (Injective CLI, Python SDK, on-chain interactions)  
> **Duration:** ~2 hours  
> **Outcome:** Build an admin-list smart contract from scratch, test it locally, then deploy and interact with it on Injective Testnet.

---

## Table of Contents

1. [Environment Setup](#1-environment-setup)
2. [The Big Picture — How CosmWasm Works](#2-the-big-picture--how-cosmwasm-works)
3. [Create the Rust Project](#3-create-the-rust-project)
4. [Entry Points](#4-entry-points)
5. [Build the Contract (First Compile)](#5-build-the-contract-first-compile)
6. [Creating a Query](#6-creating-a-query)
7. [Testing the Query](#7-testing-the-query)
8. [Contract State](#8-contract-state)
9. [Execution Messages & Error Handling](#9-execution-messages--error-handling)
10. [Optimized Build with Docker](#10-optimized-build-with-docker)
11. [Deploy to Injective Testnet](#11-deploy-to-injective-testnet)
12. [Interact with the Live Contract](#12-interact-with-the-live-contract)
13. [Exercises & Wrap-Up](#13-exercises--wrap-up)

---

## 1. Environment Setup

You already have Rust, `cargo`, and `injectived` from previous labs. Today we need one new target so the Rust compiler can output WebAssembly binaries, plus a small validation tool.

```bash
# Add the Wasm compilation target
rustup target add wasm32-unknown-unknown

# Install the CosmWasm validity checker
cargo install cosmwasm-check
```

Verify everything is in place:

```bash
rustup target list --installed | grep wasm32
# → wasm32-unknown-unknown

cosmwasm-check --version
```

> **Why `wasm32-unknown-unknown`?** CosmWasm contracts compile to WebAssembly (Wasm) — a portable binary format that runs inside a sandboxed VM on the blockchain. The triple means: 32-bit Wasm / no specific OS / no specific ABI.

---

## 2. The Big Picture — How CosmWasm Works

Before we start coding, it helps to understand the mental model that CosmWasm is built on: the **Actor Model**.

### Contracts as Actors

Think of every deployed smart contract as an **actor** — an isolated entity that:

- Has its own **internal state** (stored on-chain, private to modify, public to read).
- Reacts to **messages** sent to it (it never actively polls the outside world).
- Can **send messages** to other actors, but only *after* finishing its own work — never during.

This is fundamentally different from Solidity's model, where Contract A can call Contract B mid-execution and B's code runs *inside* A's call stack. That pattern is the root cause of **reentrancy attacks**. CosmWasm avoids this entirely: when your contract handles a message, it runs to completion, returns a `Response`, and that response can *include* sub-messages to other contracts — but those execute separately, in sequence.

### The Three Core Entry Points

Every CosmWasm contract exposes (at minimum) three message handlers:

| Entry Point     | Purpose                                    | Can modify state? |
|-----------------|--------------------------------------------|-------------------|
| `instantiate()` | Constructor — called once at deploy time   | Yes               |
| `execute()`     | Handle state-changing actions              | Yes               |
| `query()`       | Read-only data retrieval                   | No                |

There are others (`migrate`, `sudo`, `reply`) but these three are what we'll work with today.

### The Contract Lifecycle on Injective

```
  ① Upload Wasm binary → gets a CODE_ID (reusable template)
  ② Instantiate with a message → gets a CONTRACT_ADDRESS (unique instance)
  ③ Execute / Query → interact with the live contract
```

This is different from Ethereum, where deploy = upload + instantiate in one step. On CosmWasm, the same code can be instantiated many times with different initial configurations.

> **We'll revisit the Actor Model in more depth in a future lab when we cover cross-contract communication and the `Reply` mechanism.**

---

## 3. Create the Rust Project

Smart contracts are Rust **library** crates (not binaries — there's no `main()`).

```bash
cargo new --lib ./admin-contract
cd admin-contract
```

Replace the contents of `Cargo.toml`:

```toml
[package]
name = "admin-contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
cosmwasm-std = { version = "2.2", features = ["staking"] }
serde = { version = "1", default-features = false, features = ["derive"] }
cw-storage-plus = "2"
thiserror = "1"
cosmwasm-schema = "2"

[dev-dependencies]
cw-multi-test = "2"
```

**Key points to understand:**

- **`crate-type = ["cdylib", "rlib"]`** — `cdylib` produces the Wasm binary the blockchain needs. `rlib` lets us run `cargo test` normally (unit tests don't run as Wasm).
- **`cosmwasm-std`** — the standard library for CosmWasm contracts (provides entry point macros, types like `Deps`, `Response`, `Addr`, etc.).
- **`serde`** — serialization/deserialization for JSON messages.
- **`cw-storage-plus`** — ergonomic wrappers over the raw key-value store that the blockchain provides.
- **`thiserror`** — for clean custom error types.
- **`cosmwasm-schema`** — for generating JSON schemas of your messages (useful for frontend integration).
- **`cw-multi-test`** (dev only) — a simulated blockchain for integration testing without deploying.

Also create a `.cargo/config.toml` to alias the Wasm build command:

```bash
mkdir .cargo
```

```toml
# .cargo/config.toml
[alias]
wasm = "build --target wasm32-unknown-unknown --release"
wasm-debug = "build --target wasm32-unknown-unknown"
```

Now `cargo wasm` is all you need to compile. (Though we'll use Docker for the final optimized build.)

---

## 4. Entry Points

Delete everything in `src/lib.rs` and replace it with:

```rust
use cosmwasm_std::{
    entry_point, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
};

mod contract;
mod msg;

#[entry_point]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: Empty,
) -> StdResult<Response> {
    contract::instantiate(deps, env, info, msg)
}

#[entry_point]
pub fn query(deps: Deps, env: Env, msg: Empty) -> StdResult<Binary> {
    contract::query(deps, env, msg)
}
```

Now create `src/contract.rs`:

```rust
use cosmwasm_std::{
    Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
};

pub fn instantiate(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: Empty,
) -> StdResult<Response> {
    Ok(Response::new())
}

pub fn query(_deps: Deps, _env: Env, _msg: Empty) -> StdResult<Binary> {
    unimplemented!()
}
```

And an empty `src/msg.rs` for now:

```rust
// Message types will go here shortly.
```

### Walking Through the Arguments

Let's walk through each parameter of `instantiate`:

| Parameter | Type          | What it gives you                                                              |
|-----------|---------------|--------------------------------------------------------------------------------|
| `deps`    | `DepsMut`     | Mutable access to storage, ability to query other contracts, address validation |
| `env`     | `Env`         | Block height, time, chain ID, contract's own address                           |
| `info`    | `MessageInfo` | Sender address, funds sent with the message                                    |
| `msg`     | (generic)     | The JSON message body — currently `Empty` (`{}`), we'll replace this soon      |

For `query`, note two differences:
- `deps` is `Deps` (immutable) — queries cannot write to storage.
- There is no `info` — queries are not transactions, they have no sender or attached funds.

### The `#[entry_point]` Macro

Notice that `#[entry_point]` appears in `lib.rs` but **not** in `contract.rs`. This is intentional — we're separating two concerns:

- **`lib.rs`** — the Wasm-facing entry points, decorated with `#[entry_point]`. This macro generates the raw FFI glue that the Wasm runtime needs (it can only work with primitive types, not Rust structs). These thin wrappers just forward to `contract.rs`.
- **`contract.rs`** — the actual logic, written as plain Rust functions with no Wasm-specific machinery.

This split pays off later: you can reuse the logic from `contract.rs` in tests and as a library dependency for other contracts, without dragging in the Wasm entry point generation.

---

## 5. Build the Contract (First Compile)

Let's verify everything compiles:

```bash
cargo wasm
```

You should see a successful build. Now validate it's a proper CosmWasm contract:

```bash
cosmwasm-check ./target/wasm32-unknown-unknown/release/admin_contract.wasm
```

Expected output (capabilities may vary):

```
Available capabilities: {"cosmwasm_1_1", "cosmwasm_1_2", "cosmwasm_1_3", ...}

./target/wasm32-unknown-unknown/release/admin_contract.wasm: pass
```

> **Note:** `cosmwasm-check` verifies that the Wasm binary exports the required entry points. If you forget `#[entry_point]` on `instantiate`, it will fail here.

---

## 6. Creating a Query

An empty contract isn't useful. Let's add a `Greet` query that returns a hello message.

### 6.1 Define the Messages

Replace `src/msg.rs` with:

```rust
use cosmwasm_schema::QueryResponses;
use serde::{Deserialize, Serialize};

// --- Query ---

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone, QueryResponses)]
pub enum QueryMsg {
    #[returns(GreetResp)]
    Greet {},
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct GreetResp {
    pub message: String,
}
```

**Why an enum for `QueryMsg`?**

Each variant represents a distinct query the contract supports. When serialized, `QueryMsg::Greet {}` becomes `{"greet": {}}`. Using an enum gives us a clean dispatch pattern and makes it easy to add more queries later.

**Why the empty `{}`?** Without curly braces, the variant serializes as a plain string (`"greet"`) instead of an object. Using `{}` keeps the schema consistent — every variant is `{ "variant_name": { ...params } }`.

### 6.2 Implement the Query

Update `src/contract.rs`:

```rust
use cosmwasm_std::{
    to_json_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
};

use crate::msg::{GreetResp, QueryMsg};

pub fn instantiate(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: Empty,
) -> StdResult<Response> {
    Ok(Response::new())
}

pub fn query(_deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        Greet {} => to_json_binary(&query::greet()?),
    }
}

mod query {
    use super::*;

    pub fn greet() -> StdResult<GreetResp> {
        let resp = GreetResp {
            message: "Hello World".to_owned(),
        };

        Ok(resp)
    }
}
```

And update the entry point in `src/lib.rs` to use `QueryMsg` instead of `Empty`:

```rust
use cosmwasm_std::{
    entry_point, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
};

mod contract;
mod msg;

#[entry_point]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: Empty,
) -> StdResult<Response> {
    contract::instantiate(deps, env, info, msg)
}

#[entry_point]
pub fn query(deps: Deps, env: Env, msg: msg::QueryMsg) -> StdResult<Binary> {
    contract::query(deps, env, msg)
}
```

**Note the project structure pattern:** entry points live in `lib.rs` (thin wrappers), dispatch logic in `contract.rs`, message types in `msg.rs`. This keeps responsibilities clean as the contract grows.

Run `cargo wasm` again to make sure it compiles.

---

## 7. Testing the Query

### 7.1 Unit Test (Quick Sanity Check)

Add at the bottom of `src/contract.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::msg::GreetResp;

    #[test]
    fn greet_query() {
        let resp = query::greet().unwrap();
        assert_eq!(
            resp,
            GreetResp {
                message: "Hello World".to_owned()
            }
        );
    }
}
```

```bash
cargo test
```

This tests the handler function directly — fast but doesn't simulate the full contract lifecycle.

### 7.2 Integration Test with `cw-multi-test`

`cw-multi-test` simulates a blockchain in-memory. It tests the contract the way the real chain would use it: upload code → instantiate → send messages.

First, we need a stub `execute` entry point (multitest requires all three). Add to `src/contract.rs`, between the `query` function and the `query` module:

```rust
#[allow(dead_code)]
pub fn execute(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: Empty,
) -> StdResult<Response> {
    unimplemented!()
}
```

Now replace the entire test module at the bottom of `src/contract.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::msg::GreetResp;
    use cosmwasm_std::Addr;
    use cw_multi_test::{App, ContractWrapper, Executor};

    #[test]
    fn greet_query() {
        let resp = query::greet().unwrap();
        assert_eq!(
            resp,
            GreetResp {
                message: "Hello World".to_owned()
            }
        );
    }

    #[test]
    fn greet_query_via_multitest() {
        // 1. Create a simulated blockchain
        let mut app = App::default();

        // 2. "Upload" the contract code
        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        // 3. Instantiate the contract
        let contract_addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &Empty {},
                &[],              // no funds
                "Admin Contract", // label
                None,             // no admin for migrations
            )
            .unwrap();

        // 4. Query it
        let resp: GreetResp = app
            .wrap()
            .query_wasm_smart(contract_addr, &QueryMsg::Greet {})
            .unwrap();

        assert_eq!(
            resp,
            GreetResp {
                message: "Hello World".to_owned()
            }
        );
    }
}
```

```bash
cargo test
```

**Notice how the multitest flow mirrors the real on-chain lifecycle:**
1. `App::default()` = a fresh simulated chain
2. `store_code()` = `injectived tx wasm store` → returns a `code_id`
3. `instantiate_contract()` = `injectived tx wasm instantiate` → returns a contract address
4. `query_wasm_smart()` = `injectived query wasm contract-state smart`

---

## 8. Contract State

Time to make the contract actually useful. We'll store a list of **admin addresses** that gets set on instantiation and can be queried.

### 8.1 Define the State

Create `src/state.rs`:

```rust
use cosmwasm_std::Addr;
use cw_storage_plus::Item;

pub const ADMINS: Item<Vec<Addr>> = Item::new("admins");
```

Register it in `src/lib.rs` (add `mod state;`):

```rust
mod contract;
mod msg;
mod state;
```

> **How does state work in CosmWasm?** The blockchain provides a key-value store. Each contract gets its own isolated namespace. `Item<Vec<Addr>>` is not the data itself — it's an **accessor** that knows how to serialize/deserialize a `Vec<Addr>` under the key `"admins"`. The actual read/write happens through the `deps.storage` handle passed to your entry points.

### 8.2 Create an InstantiateMsg

Update `src/msg.rs` — add the instantiate message and an admins-list query:

```rust
use cosmwasm_std::Addr;
use cosmwasm_schema::QueryResponses;
use serde::{Deserialize, Serialize};

// --- Instantiate ---

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct InstantiateMsg {
    pub admins: Vec<String>,
}

// --- Query ---

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone, QueryResponses)]
pub enum QueryMsg {
    #[returns(GreetResp)]
    Greet {},
    #[returns(AdminsListResp)]
    AdminsList {},
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct GreetResp {
    pub message: String,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct AdminsListResp {
    pub admins: Vec<Addr>,
}
```

> **Why `Vec<String>` in the message but `Vec<Addr>` in state?** Messages come from the outside world — you can't trust that a string is a valid blockchain address. We accept `String` and validate with `deps.api.addr_validate()` before storing as `Addr`.

### 8.3 Update the Contract Logic

Update `src/contract.rs`:

```rust
use cosmwasm_std::{
    to_json_binary, Binary, Deps, DepsMut, Empty, Env, MessageInfo, Response, StdResult,
};

use crate::msg::{AdminsListResp, GreetResp, InstantiateMsg, QueryMsg};
use crate::state::ADMINS;

pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    // Validate every address string before storing
    let admins: StdResult<Vec<_>> = msg
        .admins
        .into_iter()
        .map(|addr| deps.api.addr_validate(&addr))
        .collect();
    ADMINS.save(deps.storage, &admins?)?;

    Ok(Response::new())
}

pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        Greet {} => to_json_binary(&query::greet()?),
        AdminsList {} => to_json_binary(&query::admins_list(deps)?),
    }
}

#[allow(dead_code)]
pub fn execute(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: Empty,
) -> StdResult<Response> {
    unimplemented!()
}

mod query {
    use super::*;

    pub fn greet() -> StdResult<GreetResp> {
        let resp = GreetResp {
            message: "Hello World".to_owned(),
        };

        Ok(resp)
    }

    pub fn admins_list(deps: Deps) -> StdResult<AdminsListResp> {
        let admins = ADMINS.load(deps.storage)?;
        let resp = AdminsListResp { admins };
        Ok(resp)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use cosmwasm_std::Addr;
    use cw_multi_test::{App, ContractWrapper, Executor};

    #[test]
    fn instantiation_with_empty_admins() {
        let mut app = App::default();

        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg { admins: vec![] },
                &[],
                "Contract",
                None,
            )
            .unwrap();

        let resp: AdminsListResp = app
            .wrap()
            .query_wasm_smart(addr, &QueryMsg::AdminsList {})
            .unwrap();

        assert_eq!(resp, AdminsListResp { admins: vec![] });
    }

    #[test]
    fn instantiation_with_admins() {
        let mut app = App::default();

        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg {
                    admins: vec!["admin1".to_owned(), "admin2".to_owned()],
                },
                &[],
                "Contract 2",
                None,
            )
            .unwrap();

        let resp: AdminsListResp = app
            .wrap()
            .query_wasm_smart(addr, &QueryMsg::AdminsList {})
            .unwrap();

        assert_eq!(
            resp,
            AdminsListResp {
                admins: vec![Addr::unchecked("admin1"), Addr::unchecked("admin2")],
            }
        );
    }
}
```

Update `src/lib.rs` to use the new `InstantiateMsg`:

```rust
use cosmwasm_std::{
    entry_point, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult,
};

mod contract;
mod msg;
mod state;

#[entry_point]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: msg::InstantiateMsg,
) -> StdResult<Response> {
    contract::instantiate(deps, env, info, msg)
}

#[entry_point]
pub fn query(deps: Deps, env: Env, msg: msg::QueryMsg) -> StdResult<Binary> {
    contract::query(deps, env, msg)
}
```

Run the tests:

```bash
cargo test
```

Both should pass. Notice how we instantiate the **same code** twice with different admin lists — each gets its own independent state. This is the code vs. instance separation in action.

---

## 9. Execution Messages & Error Handling

Now we add state-changing functionality: admins can add new members or remove themselves.

### 9.1 Define a Custom Error Type

Create `src/error.rs`:

```rust
use cosmwasm_std::{Addr, StdError};
use thiserror::Error;

#[derive(Error, Debug, PartialEq)]
pub enum ContractError {
    #[error("{0}")]
    StdError(#[from] StdError),

    #[error("{sender} is not a contract admin")]
    Unauthorized { sender: Addr },
}
```

Register it in `src/lib.rs` (add `mod error;` to the module list).

> **Why not just use `StdError`?** `StdError` covers generic problems (serialization failures, not-found, etc.). Domain-specific errors like "not an admin" deserve their own variant — it makes tests cleaner and gives better error messages to users.

### 9.2 Add Execute Messages

Update `src/msg.rs` — add the `ExecuteMsg` enum:

```rust
use cosmwasm_std::Addr;
use cosmwasm_schema::QueryResponses;
use serde::{Deserialize, Serialize};

// --- Instantiate ---

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct InstantiateMsg {
    pub admins: Vec<String>,
}

// --- Execute ---

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub enum ExecuteMsg {
    AddMembers { admins: Vec<String> },
    Leave {},
}

// --- Query ---

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone, QueryResponses)]
pub enum QueryMsg {
    #[returns(GreetResp)]
    Greet {},
    #[returns(AdminsListResp)]
    AdminsList {},
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct GreetResp {
    pub message: String,
}

#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
pub struct AdminsListResp {
    pub admins: Vec<Addr>,
}
```

### 9.3 Implement the Execute Handlers

Replace the entire contents of `src/contract.rs`:

```rust
use cosmwasm_std::{
    to_json_binary, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult,
};

use crate::error::ContractError;
use crate::msg::{AdminsListResp, ExecuteMsg, GreetResp, InstantiateMsg, QueryMsg};
use crate::state::ADMINS;

pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    let admins: StdResult<Vec<_>> = msg
        .admins
        .into_iter()
        .map(|addr| deps.api.addr_validate(&addr))
        .collect();
    ADMINS.save(deps.storage, &admins?)?;

    Ok(Response::new())
}

pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        Greet {} => to_json_binary(&query::greet()?),
        AdminsList {} => to_json_binary(&query::admins_list(deps)?),
    }
}

pub fn execute(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    use ExecuteMsg::*;

    match msg {
        AddMembers { admins } => exec::add_members(deps, info, admins),
        Leave {} => exec::leave(deps, info).map_err(Into::into),
    }
}

mod exec {
    use super::*;

    pub fn add_members(
        deps: DepsMut,
        info: MessageInfo,
        admins: Vec<String>,
    ) -> Result<Response, ContractError> {
        let mut curr_admins = ADMINS.load(deps.storage)?;

        // Only existing admins can add new members
        if !curr_admins.contains(&info.sender) {
            return Err(ContractError::Unauthorized {
                sender: info.sender,
            });
        }

        // Validate and append new addresses
        let admins: StdResult<Vec<_>> = admins
            .into_iter()
            .map(|addr| deps.api.addr_validate(&addr))
            .collect();

        curr_admins.append(&mut admins?);
        ADMINS.save(deps.storage, &curr_admins)?;

        Ok(Response::new())
    }

    pub fn leave(deps: DepsMut, info: MessageInfo) -> StdResult<Response> {
        ADMINS.update(deps.storage, move |admins| -> StdResult<_> {
            let admins = admins
                .into_iter()
                .filter(|admin| *admin != info.sender)
                .collect();
            Ok(admins)
        })?;

        Ok(Response::new())
    }
}

mod query {
    use super::*;

    pub fn greet() -> StdResult<GreetResp> {
        let resp = GreetResp {
            message: "Hello World".to_owned(),
        };

        Ok(resp)
    }

    pub fn admins_list(deps: Deps) -> StdResult<AdminsListResp> {
        let admins = ADMINS.load(deps.storage)?;
        let resp = AdminsListResp { admins };
        Ok(resp)
    }
}

// ============================================================
// TESTS
// ============================================================

#[cfg(test)]
mod tests {
    use super::*;
    use crate::error::ContractError;
    use cosmwasm_std::Addr;
    use cw_multi_test::{App, ContractWrapper, Executor};

    #[test]
    fn instantiation_with_admins() {
        let mut app = App::default();
        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg {
                    admins: vec!["admin1".to_owned(), "admin2".to_owned()],
                },
                &[],
                "Contract",
                None,
            )
            .unwrap();

        let resp: AdminsListResp = app
            .wrap()
            .query_wasm_smart(addr, &QueryMsg::AdminsList {})
            .unwrap();

        assert_eq!(
            resp,
            AdminsListResp {
                admins: vec![Addr::unchecked("admin1"), Addr::unchecked("admin2")],
            }
        );
    }

    #[test]
    fn add_members_by_admin() {
        let mut app = App::default();
        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg {
                    admins: vec!["admin1".to_owned()],
                },
                &[],
                "Contract",
                None,
            )
            .unwrap();

        // admin1 adds admin2
        app.execute_contract(
            Addr::unchecked("admin1"),
            addr.clone(),
            &ExecuteMsg::AddMembers {
                admins: vec!["admin2".to_owned()],
            },
            &[],
        )
        .unwrap();

        let resp: AdminsListResp = app
            .wrap()
            .query_wasm_smart(addr, &QueryMsg::AdminsList {})
            .unwrap();

        assert_eq!(
            resp,
            AdminsListResp {
                admins: vec![Addr::unchecked("admin1"), Addr::unchecked("admin2")],
            }
        );
    }

    #[test]
    fn unauthorized_add_members() {
        let mut app = App::default();
        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg { admins: vec![] },
                &[],
                "Contract",
                None,
            )
            .unwrap();

        let err = app
            .execute_contract(
                Addr::unchecked("intruder"),
                addr,
                &ExecuteMsg::AddMembers {
                    admins: vec!["intruder".to_owned()],
                },
                &[],
            )
            .unwrap_err();

        // Downcast from anyhow::Error back to our ContractError
        assert_eq!(
            ContractError::Unauthorized {
                sender: Addr::unchecked("intruder")
            },
            err.downcast().unwrap()
        );
    }

    #[test]
    fn admin_can_leave() {
        let mut app = App::default();
        let code = ContractWrapper::new(execute, instantiate, query);
        let code_id = app.store_code(Box::new(code));

        let addr = app
            .instantiate_contract(
                code_id,
                Addr::unchecked("owner"),
                &InstantiateMsg {
                    admins: vec!["admin1".to_owned(), "admin2".to_owned()],
                },
                &[],
                "Contract",
                None,
            )
            .unwrap();

        // admin1 leaves
        app.execute_contract(
            Addr::unchecked("admin1"),
            addr.clone(),
            &ExecuteMsg::Leave {},
            &[],
        )
        .unwrap();

        let resp: AdminsListResp = app
            .wrap()
            .query_wasm_smart(addr, &QueryMsg::AdminsList {})
            .unwrap();

        assert_eq!(
            resp,
            AdminsListResp {
                admins: vec![Addr::unchecked("admin2")],
            }
        );
    }
}
```

Update `src/lib.rs` with the `execute` entry point (final version):

```rust
use cosmwasm_std::{
    entry_point, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult,
};

use error::ContractError;

mod contract;
mod error;
mod msg;
mod state;

#[entry_point]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: msg::InstantiateMsg,
) -> StdResult<Response> {
    contract::instantiate(deps, env, info, msg)
}

#[entry_point]
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: msg::ExecuteMsg,
) -> Result<Response, ContractError> {
    contract::execute(deps, env, info, msg)
}

#[entry_point]
pub fn query(deps: Deps, env: Env, msg: msg::QueryMsg) -> StdResult<Binary> {
    contract::query(deps, env, msg)
}
```

Run the full test suite:

```bash
cargo test
```

All four tests should pass. A few things worth noting:

- **`unwrap_err()` + `downcast()`** — multitest wraps errors in `anyhow::Error`; `downcast()` recovers our original `ContractError` for assertion.
- **`info.sender`** — the address that signed the transaction. Already validated by the chain, so it's `Addr` (not `String`).
- **`ADMINS.update()`** — an atomic read-modify-write. More efficient than separate `load` + `save`.

---

## 10. Optimized Build with Docker

For testnet/mainnet deployment, we use the CosmWasm Rust Optimizer Docker image. This produces a small, deterministic Wasm binary (anyone building the same source gets the exact same output).

```bash
# From the project root (admin-contract/)

# For x86/amd64 machines:
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/optimizer:0.16.1

# For Apple Silicon / ARM64:
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/optimizer-arm64:0.16.1
```

> ⚠️ **ARM64 builds produce different Wasm hashes than x86 builds.** For production releases, always use x86. For testnet learning purposes, ARM64 is fine.

If you encounter an `Unable to update registry crates-io` error, add this to `Cargo.toml`:

```toml
[net]
git-fetch-with-cli = true
```

This creates an `artifacts/` directory containing:

```
artifacts/
├── admin_contract.wasm    ← the optimized binary
└── checksums.txt          ← SHA-256 hash
```

Verify the artifact:

```bash
cosmwasm-check artifacts/admin_contract.wasm
```

---

## 11. Deploy to Injective Testnet

We'll now take our tested contract live. Make sure you have:
- `injectived` installed (from Lab 2)
- A testnet wallet with funds (use the [Injective Faucet](https://faucet.injective.network/))

### 11.1 Set Up Environment Variables

```bash
export INJ_ADDRESS=<your_inj_testnet_address>
export NODE="https://testnet.sentry.tm.injective.network:443"
export CHAIN_ID="injective-888"
```

### 11.2 Upload (Store) the Wasm Code

```bash
injectived tx wasm store artifacts/admin_contract.wasm \
  --from=$INJ_ADDRESS \
  --chain-id=$CHAIN_ID \
  --yes \
  --fees=1000000000000000inj \
  --gas=2000000 \
  --node=$NODE
```

Note the `txhash` from the output, then query it to find your `code_id`:

```bash
injectived query tx <TXHASH> --node=$NODE
```

Look for `code_id` in the event attributes. Export it:

```bash
export CODE_ID=<your_code_id>
```

> You can also find your code on the [Injective Testnet Explorer](https://testnet.explorer.injective.network/smart-contracts/code/).

### 11.3 Instantiate the Contract

```bash
INIT='{"admins":["'$INJ_ADDRESS'"]}'

injectived tx wasm instantiate $CODE_ID "$INIT" \
  --label="AdminContractLab3" \
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

---

## 12. Interact with the Live Contract

### 12.1 Query — List Admins

```bash
QUERY='{"admins_list":{}}'

injectived query wasm contract-state smart $CONTRACT "$QUERY" \
  --node=$NODE \
  --output json
```

Expected output:

```json
{"data":{"admins":["inj1your_address_here"]}}
```

### 12.2 Query — Greet

```bash
QUERY='{"greet":{}}'

injectived query wasm contract-state smart $CONTRACT "$QUERY" \
  --node=$NODE \
  --output json
```

Expected:

```json
{"data":{"message":"Hello World"}}
```

### 12.3 Execute — Add a Member

Ask a classmate for their `inj1...` address, or use a second address you control:

```bash
ADD='{"add_members":{"admins":["inj1_classmate_address_here"]}}'

injectived tx wasm execute $CONTRACT "$ADD" \
  --from=$INJ_ADDRESS \
  --chain-id=$CHAIN_ID \
  --yes \
  --fees=1000000000000000inj \
  --gas=2000000 \
  --node=$NODE
```

Then query `admins_list` again to confirm the new admin was added.

### 12.4 Execute — Leave

```bash
LEAVE='{"leave":{}}'

injectived tx wasm execute $CONTRACT "$LEAVE" \
  --from=$INJ_ADDRESS \
  --chain-id=$CHAIN_ID \
  --yes \
  --fees=1000000000000000inj \
  --gas=2000000 \
  --node=$NODE
```

Query `admins_list` one more time — your address should be gone.

### 12.5 Test Unauthorized Access

Have a classmate (with a different address not in the admin list) try to call `add_members` on your contract. They should get an **Unauthorized** error — the same one we tested locally!

---

## 13. Exercises & Wrap-Up

### What We Covered

- **Actor Model** — contracts as message-driven actors with no reentrancy.
- **Project structure** — `lib.rs` (entry points), `contract.rs` (logic), `msg.rs` (types), `state.rs` (storage), `error.rs` (custom errors).
- **Three entry points** — `instantiate`, `execute`, `query`.
- **State management** — `cw-storage-plus` `Item`, `save`/`load`/`update`.
- **Testing** — unit tests, black-box tests with `cw-multi-test`.
- **Deployment to Injective Testnet** — store code, instantiate, query, execute.

### Optional Exercises (Take-Home)

1. **`GreetAdmin`** — Add a query variant that takes an admin name as input and returns a personalized greeting. If the name is not in the admin list, return an error.

2. **`RemoveMember`** — Add an execute message that allows an admin to remove *another* admin from the list (not just themselves). Think about edge cases: can the last admin be removed?

3. **Events** — Modify `add_members` and `leave` to emit CosmWasm events/attributes on the `Response` (e.g., `.add_attribute("action", "add_members")`). These show up as transaction logs on the explorer. See the [CosmWasm book's events chapter](https://book.cosmwasm.com/basics/events.html) for guidance.

4. **Schema generation** — Create an `examples/schema.rs` that uses `cosmwasm_schema::write_api!` to generate JSON schemas for your messages. Run `cargo schema` and inspect the output.

---

## Quick Reference — File Structure

```
admin-contract/
├── .cargo/
│   └── config.toml          ← build aliases
├── artifacts/
│   └── admin_contract.wasm  ← optimized build output
├── src/
│   ├── lib.rs               ← entry points (#[entry_point] wrappers)
│   ├── contract.rs           ← dispatch + handler logic + tests
│   ├── msg.rs               ← InstantiateMsg, ExecuteMsg, QueryMsg, responses
│   ├── state.rs             ← ADMINS: Item<Vec<Addr>>
│   └── error.rs             ← ContractError enum
└── Cargo.toml
```

---

## Resources

- [CosmWasm Book](https://book.cosmwasm.com/) — the full tutorial this lab is based on
- [Injective CosmWasm Docs](https://docs.injective.network/developers-cosmwasm/) — Injective-specific deployment and modules
- [cw-storage-plus API](https://docs.rs/cw-storage-plus/latest/cw_storage_plus/) — `Item`, `Map`, and more
- [cw-multi-test API](https://docs.rs/cw-multi-test/latest/cw_multi_test/) — testing framework reference
- [Injective Testnet Faucet](https://faucet.injective.network/) — get testnet INJ
- [Injective Testnet Explorer](https://testnet.explorer.injective.network/) — view transactions and contracts