# Multi-Ledger-Token

# Multi-Token Ledger Standard (MTLS) - Developer Guide

A comprehensive guide to understanding, implementing, and using multi-token ledgers on the Internet Computer Protocol (ICP).

## Table of Contents

1. [What is a Multi-Token Ledger?](#what-is-a-multi-token-ledger)
2. [Architecture Overview](#architecture-overview)
3. [Core Concepts](#core-concepts)
4. [Implementation Guide](#implementation-guide)
5. [Factory-of-Factories Pattern](#factory-of-factories-pattern)
6. [ICRC Compliance](#icrc-compliance)
7. [Deployment & Usage](#deployment--usage)
8. [Proxy Pattern for Compatibility](#proxy-pattern-for-compatibility)
9. [Best Practices](#best-practices)

## What is a Multi-Token Ledger?

Traditional token systems on ICP deploy one canister per token. While this works, it creates overhead:
- High cycles cost (each canister reserves cycles)
- Management complexity (upgrading multiple canisters)
- Deployment friction for small or experimental tokens

A **Multi-Token Ledger** solves this by allowing multiple fungible tokens to coexist within a single canister, each identified by a unique `TokenId`.

### Benefits

- **Cost Efficiency**: Share cycles across multiple tokens
- **Simplified Management**: One canister to upgrade and maintain
- **Atomic Operations**: Execute multi-token transfers in single calls
- **Scalability**: Create hundreds of tokens without deployment overhead
- **Shared Infrastructure**: Common logic for transfers, allowances, metadata

## Architecture Overview

```
┌─────────────────────────┐
│   MasterFactory         │  ← Spawns isolated ledgers
│   (singleton canister)  │
└─────────────────────────┘
            │
            ├─ createLedger(owner=UserA)
            │
            v
┌─────────────────────────┐
│   LedgerCanisterA       │  ← UserA's isolated multi-token ledger
│   Owner: UserA          │
│   ├─ Token[0]: APE      │
│   ├─ Token[1]: KONG     │
│   └─ Token[2]: DEGEN    │
└─────────────────────────┘
            │
            ├─ createLedger(owner=UserB)  
            │
            v
┌─────────────────────────┐
│   LedgerCanisterB       │  ← UserB's isolated multi-token ledger
│   Owner: UserB          │
│   ├─ Token[0]: MOON     │
│   └─ Token[1]: ROCKET   │
└─────────────────────────┘
```

## Core Concepts

### TokenId
Each token within a ledger is identified by a unique `TokenId` (typically `Nat`):
```motoko
public type TokenId = Nat;
```

### Token Metadata
Standard token information:
```motoko
public type Metadata = {
  name : Text;        // "AstroApe Token"
  symbol : Text;      // "APE"
  decimals : Nat;     // 8
};
```

### Multi-Token State Management
Each ledger maintains:
```motoko
// Map of TokenId -> Token data
tokens : HashMap<TokenId, Token>

// Token structure
public type Token = {
  metadata : Metadata;
  totalSupply : Nat;
  balances : HashMap<Principal, Nat>;
  allowances : HashMap<(Principal, Principal), Nat>;
  creator : Principal;
};
```

### Method Signatures
Multi-token methods include `tokenId` as the first parameter:

**Single-token ICRC-1:**
```motoko
icrc1_balance_of(account : Principal) : async Nat
```

**Multi-token version:**
```motoko
icrc1_balance_of(tokenId : TokenId, account : Principal) : async Nat
```

## Implementation Guide

### Step 1: Master Factory Canister

The `MasterFactory` spawns isolated ledger canisters for users:

```motoko
// MasterFactory.mo
import Management "canister:management";

actor MasterFactory {
  
  stable var ledgers : [Principal] = [];
  
  public shared(msg) func createLedger() : async Principal {
    let owner = msg.caller;
    
    // Create new canister
    let settings = {
      controllers = ?[owner, Principal.fromActor(this)];
      compute_allocation = null;
      memory_allocation = null;
      freezing_threshold = null;
    };
    
    let { canister_id } = await Management.create_canister({ settings = ?settings });
    
    // Install ledger WASM
    let wasmModule : Blob = /* your compiled ledger wasm */;
    await Management.install_code({
      arg = to_candid(owner);
      wasm_module = wasmModule;
      mode = #install;
      canister_id = canister_id;
    });
    
    // Track created ledgers
    ledgers := Array.append(ledgers, [canister_id]);
    canister_id
  };
  
  public query func listLedgers() : async [Principal] {
    ledgers
  };
}
```

### Step 2: Multi-Token Ledger Implementation

Each spawned ledger canister manages multiple tokens:

```motoko
// Ledger.mo
actor class Ledger(owner: Principal) = this {
  
  public type TokenId = Nat;
  
  public type Metadata = {
    name : Text;
    symbol : Text;
    decimals : Nat;
  };
  
  private stable var tokens = HashMap<TokenId, Token>(16, Nat.equal, Nat.hash);
  private stable var nextTokenId : TokenId = 0;
  
  // Create new token (owner-only)
  public shared(msg) func createToken(
    metadata : Metadata, 
    initialSupply : Nat
  ) : async TokenId {
    assert(msg.caller == owner);
    
    let tokenId = nextTokenId;
    nextTokenId += 1;
    
    let balances = HashMap<Principal, Nat>(16, Principal.equal, Principal.hash);
    if (initialSupply > 0) {
      balances.put(owner, initialSupply);
    };
    
    let token : Token = {
      metadata = metadata;
      totalSupply = initialSupply;
      balances = balances;
      allowances = HashMap<(Principal, Principal), Nat>(16, allowanceEqual, allowanceHash);
      creator = msg.caller;
    };
    
    tokens.put(tokenId, token);
    tokenId
  };
  
  // ICRC-1 Methods
  public query func icrc1_balance_of(tokenId : TokenId, account : Principal) : async Nat {
    switch (tokens.get(tokenId)) {
      case null { 0 };
      case (?t) {
        switch (t.balances.get(account)) {
          case null 0;
          case (?b) b;
        }
      };
    }
  };
  
  public shared(msg) func icrc1_transfer(
    tokenId : TokenId, 
    from : Principal, 
    to : Principal, 
    amount : Nat
  ) : async Bool {
    assert(msg.caller == from);
    
    switch (tokens.get(tokenId)) {
      case null { false };
      case (?t) {
        let fromBal = switch (t.balances.get(from)) { case null 0; case (?b) b };
        if (fromBal < amount) { return false };
        
        t.balances.put(from, fromBal - amount);
        let toBal = switch (t.balances.get(to)) { case null 0; case (?b) b };
        t.balances.put(to, toBal + amount);
        true
      };
    }
  };
  
  // ICRC-2 Methods (allowances)
  public shared(msg) func icrc2_approve(
    tokenId : TokenId,
    owner : Principal,
    spender : Principal,
    amount : Nat
  ) : async Bool {
    assert(msg.caller == owner);
    
    switch (tokens.get(tokenId)) {
      case null { false };
      case (?t) {
        t.allowances.put((owner, spender), amount);
        true
      };
    }
  };
  
  public shared(msg) func icrc2_transfer_from(
    tokenId : TokenId,
    spender : Principal,
    owner : Principal,
    to : Principal,
    amount : Nat
  ) : async Bool {
    assert(msg.caller == spender);
    
    switch (tokens.get(tokenId)) {
      case null { false };
      case (?t) {
        let allowance = switch (t.allowances.get((owner, spender))) {
          case null 0; case (?a) a
        };
        if (allowance < amount) { return false };
        
        let ownerBal = switch (t.balances.get(owner)) { case null 0; case (?b) b };
        if (ownerBal < amount) { return false };
        
        // Execute transfer
        t.balances.put(owner, ownerBal - amount);
        let toBal = switch (t.balances.get(to)) { case null 0; case (?b) b };
        t.balances.put(to, toBal + amount);
        
        // Update allowance
        t.allowances.put((owner, spender), allowance - amount);
        true
      };
    }
  };
  
  // Utility functions
  public query func supported_tokens() : async [TokenId] {
    var result : [TokenId] = [];
    for ((id, _) in tokens.entries()) {
      result := Array.append(result, [id]);
    };
    result
  };
}
```

## Factory-of-Factories Pattern

This architecture provides:

1. **Isolation**: Each user gets their own ledger canister
2. **Ownership**: Users control their ledger (create tokens, mint, burn)
3. **Scalability**: Unlimited users, each with unlimited tokens
4. **Security**: No shared state between different users' tokens

## ICRC Compliance

### ICRC-1 Methods (with TokenId)
- `icrc1_balance_of(tokenId, account)` → balance
- `icrc1_transfer(tokenId, from, to, amount)` → success
- `icrc1_total_supply(tokenId)` → total supply
- `icrc1_metadata(tokenId)` → metadata
- `icrc1_supported_standards()` → standards array

### ICRC-2 Methods (Allowances)
- `icrc2_approve(tokenId, owner, spender, amount)` → success
- `icrc2_allowance(tokenId, owner, spender)` → allowance amount
- `icrc2_transfer_from(tokenId, spender, owner, to, amount)` → success

## Deployment & Usage

### Deploy Master Factory
```bash
dfx deploy MasterFactory
```

### Create Personal Ledger
```bash
dfx canister call MasterFactory createLedger
# Returns: (principal "rdmx6-jaaaa-aaaah-qcaiq-cai")  // Your ledger canister ID
```

### Create Tokens in Your Ledger
```bash
dfx canister call rdmx6-jaaaa-aaaah-qcaiq-cai createToken '(
  record { name="AstroApe"; symbol="APE"; decimals=8 }, 
  1000000000:nat
)'
# Returns: (0 : nat)  // TokenId

dfx canister call rdmx6-jaaaa-aaaah-qcaiq-cai createToken '(
  record { name="MoonToken"; symbol="MOON"; decimals=18 }, 
  500000000:nat
)'
# Returns: (1 : nat)  // TokenId
```

### Transfer Tokens
```bash
dfx canister call rdmx6-jaaaa-aaaah-qcaiq-cai icrc1_transfer '(
  0:nat,                                    # TokenId (APE)
  principal "aaaaa-aa",                     # from
  principal "bbbbb-bb",                     # to
  1000:nat                                  # amount
)'
```

### Check Balances
```bash
dfx canister call rdmx6-jaaaa-aaaah-qcaiq-cai icrc1_balance_of '(
  0:nat,                                    # TokenId
  principal "aaaaa-aa"                      # account
)'
```

## Proxy Pattern for Compatibility

Some wallets and explorers expect one canister per token. You can create proxy canisters that forward calls to your multi-token ledger:

```motoko
// TokenProxy.mo
actor TokenProxy(factoryPrincipal : Principal, tokenId : Nat) {
  
  // Standard ICRC-1 interface (no tokenId required)
  public func icrc1_balance_of(account : Principal) : async Nat {
    let factory : actor {
      icrc1_balance_of : (Nat, Principal) -> async Nat;
    } = actor(Principal.toText(factoryPrincipal));
    
    await factory.icrc1_balance_of(tokenId, account)
  };
  
  public func icrc1_transfer(from : Principal, to : Principal, amount : Nat) : async Bool {
    let factory : actor {
      icrc1_transfer : (Nat, Principal, Principal, Nat) -> async Bool;
    } = actor(Principal.toText(factoryPrincipal));
    
    await factory.icrc1_transfer(tokenId, from, to, amount)
  };
}
```

Deploy proxy for each token:
```bash
dfx deploy TokenProxy --argument '(principal "rdmx6-jaaaa-aaaah-qcaiq-cai", 0:nat)'
```

## Best Practices

### Security
- Validate all inputs (non-zero amounts, valid principals)
- Implement proper access controls (owner-only operations)
- Use stable variables for persistence across upgrades
- Consider rate limiting to prevent abuse

### Performance
- Use efficient data structures (HashMap with good hash functions)
- Implement pagination for large token lists
- Consider memory usage with many tokens/users

### Upgrades
- Design stable variable layout for smooth upgrades
- Test upgrade scenarios thoroughly
- Consider versioning for breaking changes

### Error Handling
- Return meaningful error codes/messages
- Log important operations for debugging
- Handle edge cases (zero amounts, self-transfers)

### Integration
- Follow ICRC standards for maximum compatibility
- Provide clear documentation and examples
- Consider event logging for indexers/explorers

## Use Cases

- **DEX Platforms**: Manage trading pairs in single canisters
- **Token Launchpads**: Create new tokens without deployment overhead  
- **Wrapped Assets**: Issue cross-chain tokens (ckBTC, ckETH) efficiently
- **Gaming**: Manage in-game currencies and items
- **DeFi Protocols**: Handle multiple yield-bearing tokens
- **Meme Token Factories**: Quick deployment for experimental tokens

---

With this multi-token ledger standard, you can build scalable, cost-efficient token systems on ICP while maintaining compatibility with existing tooling through proxy patterns.
