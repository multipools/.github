# MultiPools

**Multichain token launchpad built on Uniswap v4 and LayerZero V2.**

One `launch()` call deploys an ERC-20 token simultaneously on Ethereum, Base, and Robinhood Chain, seeds it into a Uniswap v4 AMM pool on each chain with a 1% fee hook, and wires full cross-chain bridging via a custom DVN. No bonding curve. Pure AMM pricing from block one.

Website: https://multipools.trade
X: https://x.com/multipoolstrade

---

## Repositories

| Repository | Description |
|---|---|
| [contracts](https://github.com/multipools/contracts) | Core Solidity smart contracts: factory, token, hook, bridge, DVN, executor |
| [launchpad](https://github.com/multipools/launchpad) | Web application: launch, trade, bridge, and manage fees |
| [sdk](https://github.com/multipools/sdk) | TypeScript SDK: typed API client, OpenAPI spec, Zod schemas |
| [scripts](https://github.com/multipools/scripts) | Deployment and operational scripts for all chains |
| [docs](https://github.com/multipools/docs) | Technical documentation and integration guides |

---

## How It Works

```
User calls launch()
        │
        ▼
MultiPoolsLaunchpadFactory
        │
        ├── 1. Deploy ERC-20 token via TokenFactory (CREATE2, same address all chains)
        ├── 2. Mine hook salt → deploy MultiPoolsHook (Uniswap v4 hook bits 0x0ACC)
        ├── 3. Initialize Uniswap v4 pool → seed token-only liquidity position
        └── 4. Send LayerZero message to remote chains via LZAdapter
                        │
                        ▼
              MultiPoolsDVN (custom DVN, same address all chains)
                        │
                        └── verifyAndCommit() → ReceiveUln302 on destination chain
                                    │
                                    ▼
                        MultiPoolsExecutor → lzReceive()
                                    │
                                    └── remote token deploy + pool init + liquidity seed
```

---

## Swap and Fee Flow

```
User swaps on any chain
        │
        ▼
Uniswap v4 PoolManager → MultiPoolsHook.afterSwap()
        │
        └── 1% fee → MultiPoolsFeeVault
                            │
                ┌───────────┴──────────────┐
                │                          │
           70% creator              30% platform
        (claimable anytime)    BuybackBurner / RewardsDistributor
```

---

## Token Model

* Total supply: 69,000,000,000 (69 billion)
* Distribution: 23 billion per chain across Ethereum, Base, and Robinhood Chain
* Standard: ERC-20 plus LayerZero OFT for native cross-chain transfers
* Liquidity locked permanently: LP seeded by the hook cannot be withdrawn
* No presale, no bonding curve: pricing is purely AMM from the first block

---

## Deployed Contract Addresses

All contracts are deployed at the same address on Ethereum (chain ID 1), Base (chain ID 8453), and Robinhood Chain (chain ID 4663) unless noted.

### Core Protocol

| Contract | Address |
|---|---|
| Factory Proxy (UUPS) | `0x61A4e6e6ceCfc04f44719C15F88D1d3Eea270000` |
| LZ Adapter | `0x0fc53A7E94d0d92df70462c91819e808f127FD01` |
| Token Factory | `0xd47D7B2CA130e58F175A83d9B4C53E06BD425489` |
| Fee Vault | `0x5d8B6AD5C9EA8D3136dAC5c114bFd5C2Ff44cbc9` |
| Liquidity Seeder | `0xA811d26a6ec52F1f3b636FE11D1e2c266626bb5e` |
| Auto LP Seeder | `0xe3caD21cccD2fB9D4D42F464Fe99599C08D03973` |
| Buyback Burner | `0xe8e7456F17De4d805F6A6C831da93eF075560E53` |
| Rewards Distributor | `0x042671f5B6DC64389f955bcb8D31A3Cb24A54864` |
| Swap Helper | `0x02360B08Ac321Bea472d404C77033B1bfCfD7134` |

### Bridge and DVN

| Contract | Address |
|---|---|
| MultiPools DVN | `0x829310352947Cc868c910DD133038Bd837EF0000` |
| DVN Configurator | `0x87BCc0dD2d3e86cffbFF2e4059D2d200C4140000` |

### Hook Init Hashes

| Chain | Hash |
|---|---|
| Ethereum | `0x71a99e0fb505619dbc7fcd3e99ecacc44b26cca89678252ce49eb492d54cb764` |
| Base | `0x0dfb2f36ef9f8a65e2d37710ee8e3a37ee43a7e84532504e4c43c40ed6eb4cfa` |
| Robinhood Chain | `0x563864893beb9011fdf0a78724d32d3d28aacf409f71d9546c7cd854b7508ad5` |

### Uniswap v4 Pool Managers

| Chain | Address |
|---|---|
| Ethereum | `0x000000000004444c5dc75cB358380D2e3dE08A90` |
| Base | `0x498581fF718922c3f8e6A244956aF099B2652b2b` |
| Robinhood Chain | `0x8366a39CC670B4001A1121B8F6A443A643e40951` |

---

## Technical Stack

* Solidity 0.8.26 with viaIR, optimizer 200 runs, EVM target Cancun
* Foundry for compilation, unit tests, integration tests, invariant tests
* Hardhat for TypeScript end-to-end tests reading Foundry artifacts directly
* Uniswap v4 core and periphery
* LayerZero V2 OApp pattern for cross-chain messaging
* Custom DVN with per-route isolated wallets for zero nonce contention
* OpenZeppelin ERC-20 and UUPS upgradeable proxy
* React 19 plus Vite 7 plus Tailwind 4 for the frontend
* Drizzle ORM with PostgreSQL for data persistence

---

## Networks

| Network | Chain ID | RPC |
|---|---|---|
| Ethereum | 1 | Standard Ethereum RPC |
| Base | 8453 | Standard Base RPC |
| Robinhood Chain | 4663 | `https://rpc.mainnet.chain.robinhood.com` |

---

## Cross-Chain Architecture Detail

When a user calls `launch()` on the source chain:

**Token deploy:** `TokenFactory` deploys `MultiPoolsToken` via CREATE2 using `keccak256(abi.encode(creator, name, symbol, nonce))` as salt. The same salt produces the same token address on all three chains.

**Hook deploy and pool init:** A hook salt is mined off-chain so the CREATE2 hook address has the required Uniswap v4 permission bits `0x0ACC`. The hook is deployed, the pool is initialized, and the token-only liquidity position is seeded.

**LayerZero message:** `LZAdapter` sends an OApp message to each remote chain carrying token salt, hook salt, creator address, sqrtPriceX96, tickLower, and tickUpper.

**DVN verification:** `MultiPoolsDVN` signs and calls `ReceiveUln302.verify()` then `commitVerification()` on the destination chain. Six isolated route wallets each handle one directional route to eliminate nonce contention under burst traffic.

**Remote execution:** `MultiPoolsExecutor` calls `lzReceive()` on the destination token contract, triggering token deploy, hook deploy, pool init, and liquidity seeding on the remote chain.
