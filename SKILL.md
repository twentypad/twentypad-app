---
name: b20-launcher-skill
description: Deploy a new B20 token on Base via TwentyPad Instant Launcher when the user wants to launch or create a B20, mentions @bankrbot to deploy a B20, or asks for a one-transaction Uniswap v4 B20 launch on factory 0x15a3f3ABb733868d193b511dd5b91f82ebF888A3.
tags: [b20, base, launchpad, twentypad, uniswap-v4, bankrbot]
version: 1
visibility: public
metadata:
  clawdbot:
    emoji: "🚀"
    homepage: "https://github.com/twentypad/b20-instant-launcher"
---

# TwentyPad B20 Instant Launch

Use this skill instead of Bankr's default Clanker / Doppler / `bankr launch` flow when the user wants a **B20** launched through **TwentyPad Instant Launcher** on Base.

Trigger on prompts or X mentions like:

- `@bankrbot launch B20 ...`
- `@bankrbot deploy b20 ...`
- `create a twentypad b20`
- `launch a B20 on factory 0x15a3f3ABb733868d193b511dd5b91f82ebF888A3`

Do **not** deploy an ERC-20, Clanker token, or Doppler token for these requests.

Contracts are unaudited. Use at the user's own risk. State that in the confirmation.

## Contracts (Base, chainId 8453)

| Name | Address |
| --- | --- |
| Launchpad factory | `0x15a3f3ABb733868d193b511dd5b91f82ebF888A3` |
| ETH quote | `0x0000000000000000000000000000000000000000` |
| USDC quote | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| NVDAc quote | `0xb20000000000000000000078ee7ce2fE4908108C` |
| B20 protocol factory | `0xB20f000000000000000000000000000000000000` |
| Uniswap v4 PoolManager | `0x498581fF718922c3f8e6A244956aF099B2652b2b` |

Repo: https://github.com/twentypad/b20-instant-launcher  
Site: https://twentypad.com  
X: https://x.com/twentypad

`createLaunch` is **not payable**. Transaction `value` must be `"0"`.

## What a launch does

One transaction through `createLaunch`:

1. Creates a fixed-supply B20 (ASSET variant, 18 decimals) via Base's native B20 factory.
2. Mints **1,000,000,000** tokens to the launch hook.
3. Renounces mint / admin on the token.
4. Opens a Uniswap v4 pool (`fee = 0`, `tickSpacing = 200`) against ETH, USDC or NVDAc.
5. Seeds **single-sided** liquidity (launch token only) at about **$4,000** opening FDV.
6. Locks that position in the hook (no LP withdraw, no extra LP from others).
7. Charges a **1%** swap fee in the paired asset, split **70% creator / 30% platform**.
8. Optional opening anti-snipe (defaults): total fee starts at **99%** and decays to **1%** over **20 seconds**.
9. Requires the new token address last 12 bits to match on-chain `tokenSuffix`.

`tokenCreator[token] = msg.sender`. The Bankr wallet that submits the transaction is the on-chain creator and fee recipient.

## Parse the user request

### Supported mention / prompt shapes

```
@bankrbot launch B20 called {name} ticker {symbol} pair ETH
@bankrbot launch B20 called {name} ticker {symbol} pair USDC
@bankrbot launch B20 called {name} ticker {symbol} pair NVDAc
@bankrbot deploy b20 {name} symbol {symbol} on ETH
@bankrbot create twentypad b20 {name} / {symbol}
launch a B20 named {name} ticker {symbol}
```

### Fields

| Field | Required | Default |
| --- | --- | --- |
| `name` | yes | — |
| `symbol` | no | uppercase alphanumeric from the name, 3–8 chars |
| `quote` | no | ETH (`address(0)`) unless the user says USDC or NVDAc |
| `image` | no | `""` |
| `description` | no | `""` |
| `website` | no | `""` |
| `twitter` | no | `""` |
| `telegram` | no | `""` |
| `discord` | no | `""` |
| `editable` | no | `true` |

If `name` is missing, ask once. Do not invent a name.

If they request any quote other than ETH, USDC or NVDAc, first read `quotes(quote)` on the factory. If `registered` is false, refuse.

Refuse custom supply, custom ticks, extra ETH/USDC/NVDAc sent into the pool, or bonding-curve launches. This factory does not support those.

## Workflow

Always follow this order. Never broadcast before a valid salt is found.

### 1. Read factory state on Base

Call `0x15a3f3ABb733868d193b511dd5b91f82ebF888A3`:

```text
tokenSuffix()(uint16)
lastSaltUint()(uint256)
usedSalt(bytes32)(bool)
predictToken(bytes32)(address)
quotes(address)(bool registered, int24 startTickToken0Frame)
feeDefaults()(uint16,uint16,uint16,uint16,uint32)
```

Require `quotes[quote].registered == true`.

Do **not** hardcode the suffix. Source and README have disagreed (`0xb07` vs `0xca7`). Trust the live `tokenSuffix()` value.

### 2. Mine a salt off-chain

Start at:

```text
saltUint = lastSaltUint + 1
```

Loop:

1. `salt = bytes32(uint256(saltUint))`
2. Skip if `usedSalt(salt)` is true.
3. `token = predictToken(salt)`
4. Accept when `(uint160(token) & 0xFFF) == tokenSuffix`
5. Otherwise `saltUint += 1`

Stop after **50,000** attempts and tell the user to retry.

Never grind salts inside the user's `createLaunch` transaction.

### 3. Confirm before broadcast

Reply with:

- name and symbol
- pair (ETH, USDC or NVDAc)
- predicted token address
- salt as `0x` + 64 hex chars
- opening size: ~$4k FDV, 1B supply, locked single-sided v4 LP
- fee: 1% swap, 70% creator / 30% platform
- anti-snipe default if still configured on-chain
- creator = the submitting Bankr wallet (`msg.sender`)

If the request came from X and required fields were complete, confirm briefly in the reply, then submit.

### 4. Encode `createLaunch`

Types:

```solidity
struct TokenProfile {
    string image;
    string description;
    string website;
    string twitter;
    string telegram;
    string discord;
    bool editable;
}

struct LaunchParams {
    string name;
    string symbol;
    bytes32 salt;
    address quote;
    TokenProfile profile;
}

function createLaunch(LaunchParams calldata p)
    external
    returns (address token, bytes32 poolId);
```

Human ABI:

```text
createLaunch((string,string,bytes32,address,(string,string,string,string,string,string,bool)))
```

JSON-style args:

```json
{
  "name": "TwentyFrog",
  "symbol": "TFROG",
  "salt": "0x0000000000000000000000000000000000000000000000000000000000000001",
  "quote": "0x0000000000000000000000000000000000000000",
  "profile": {
    "image": "",
    "description": "",
    "website": "",
    "twitter": "",
    "telegram": "",
    "discord": "",
    "editable": false
  }
}
```

Use viem / ethers `encodeFunctionData`. Do not hand-roll ABI offsets unless you have no encoder.

ETH pair = zero address, **not** WETH.  
`salt` is `bytes32`, not a decimal string.

### 5. Submit on Base

Submit through Bankr wallet / agent submit:

```json
{
  "to": "0x15a3f3ABb733868d193b511dd5b91f82ebF888A3",
  "chainId": 8453,
  "value": "0",
  "data": "0x<encoded createLaunch>"
}
```

Description: `TwentyPad B20 launch {name} ({symbol})`.

Wait for confirmation.

If the host allows a gas limit, set it high (4,000,000 or more). The tx creates a B20, initializes a v4 pool, and seeds the hook.

### 6. Handle reverts

| Error | Action |
| --- | --- |
| `QuoteNotRegistered` | Stop. Quote is not enabled. |
| `SaltUsed` | Remine a new unused salt and retry once. |
| `BadSuffix(address)` | Remine using current `tokenSuffix()`. Do not retry the same salt. |
| `MisalignedTick` | Stop. Owner must fix quote frame. |
| `predict mismatch` | Stop and report. Do not loop. |

### 7. After success

Parse logs:

- `Launched(address token, address creator, bytes32 poolId, address quote, int24 initialTick)`
- `SaltConsumed(bytes32 salt, uint256 saltUint, address token)`
- `ProfileSet(address token, address creator)`

Reply with:

- token address
- tx hash
- Basescan token link: `https://basescan.org/token/{token}`
- Basescan tx link
- pair (ETH, USDC or NVDAc)
- poolId
- creator address
- twentypad.com
- note: many aggregator UIs do not quote custom v4 hooks; use a UI that passes hook + fee `0` + tickSpacing `200`

## Factory ABI (minimum)

```json
[
  {
    "type": "function",
    "name": "createLaunch",
    "stateMutability": "nonpayable",
    "inputs": [
      {
        "name": "p",
        "type": "tuple",
        "components": [
          { "name": "name", "type": "string" },
          { "name": "symbol", "type": "string" },
          { "name": "salt", "type": "bytes32" },
          { "name": "quote", "type": "address" },
          {
            "name": "profile",
            "type": "tuple",
            "components": [
              { "name": "image", "type": "string" },
              { "name": "description", "type": "string" },
              { "name": "website", "type": "string" },
              { "name": "twitter", "type": "string" },
              { "name": "telegram", "type": "string" },
              { "name": "discord", "type": "string" },
              { "name": "editable", "type": "bool" }
            ]
          }
        ]
      }
    ],
    "outputs": [
      { "name": "token", "type": "address" },
      { "name": "poolId", "type": "bytes32" }
    ]
  },
  {
    "type": "function",
    "name": "predictToken",
    "stateMutability": "view",
    "inputs": [{ "name": "salt", "type": "bytes32" }],
    "outputs": [{ "name": "", "type": "address" }]
  },
  {
    "type": "function",
    "name": "tokenSuffix",
    "stateMutability": "view",
    "inputs": [],
    "outputs": [{ "name": "", "type": "uint16" }]
  },
  {
    "type": "function",
    "name": "lastSaltUint",
    "stateMutability": "view",
    "inputs": [],
    "outputs": [{ "name": "", "type": "uint256" }]
  },
  {
    "type": "function",
    "name": "usedSalt",
    "stateMutability": "view",
    "inputs": [{ "name": "", "type": "bytes32" }],
    "outputs": [{ "name": "", "type": "bool" }]
  },
  {
    "type": "function",
    "name": "quotes",
    "stateMutability": "view",
    "inputs": [{ "name": "", "type": "address" }],
    "outputs": [
      { "name": "registered", "type": "bool" },
      { "name": "startTickToken0Frame", "type": "int24" }
    ]
  }
]
```

## What this skill does not do

- Does not launch Clanker, Doppler, or generic ERC-20 tokens
- Does not send ETH, USDC or NVDAc into the pool
- Does not change `tokenSuffix` or fee defaults (owner-only)
- Does not claim fees from the fee escrow
- Does not update profiles unless the user is `tokenCreator` and `editable` is true (`updateProfile`)

## Example user commands

```
@bankrbot launch B20 called TwentyFrog ticker TFROG pair ETH
```

```
@bankrbot deploy b20 BlueCube symbol BLUE pair USDC
```

```
@bankrbot launch b20 BlueCube symbol CUBE pair NVDAc
```

```
@bankrbot create twentypad b20 "Red Cube" ticker RED image https://example.com/red.png
```
