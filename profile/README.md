# MultiPools

Omnichain token launchpad built on Uniswap v4 and LayerZero V2.

One `launch()` call deploys an ERC-20 token simultaneously on Ethereum, Base, and Robinhood Chain, seeds it into a Uniswap v4 AMM pool on each chain with a 1% fee hook, and wires full cross-chain bridging via a custom DVN. No bonding curve. Pure AMM pricing from block one.

Website: https://multipools.trade
X: https://x.com/multipoolstrade

---

## Repositories

| Repository | Description |
|---|---|
| [contracts](https://github.com/multipools/contracts) | Core Solidity smart contracts: factory, token, hook, DVN, executor, fee vault |
| [docs](https://github.com/multipools/docs) | Technical documentation and integration guides |

---

## How It Works

```
User calls launch()
        |
        v
MultiPoolsLaunchpadFactory (UUPS proxy, same address all chains)
        |
        +-- 1. Deploy ERC-20 via CREATE2 (same token address all chains)
        |
        +-- 2. Mine hook salt, deploy MultiPoolsHook (Uniswap v4, permission bits 0x0ACC)
        |
        +-- 3. Initialize Uniswap v4 pool, seed token-only LP (locked forever)
        |
        +-- 4. Send LayerZero message to remote chains via LZAdapter
                        |
                        v
              MultiPoolsDVN (custom DVN, same address all chains)
                        |
                        v
              verifyAndCommit() on destination chain
                        |
                        v
              MultiPoolsExecutor calls lzReceive()
                        |
                        v
              Remote chain: token deploy + pool init + LP seed
```

---

## Swap and Fee Flow

```
User swaps on any chain
        |
        v
Uniswap v4 PoolManager calls MultiPoolsHook.afterSwap()
        |
        v
1% fee accrued to MultiPoolsFeeVault
        |
        +-- 70% claimable by token creator
        |
        +-- 30% to platform: BuybackBurner + RewardsDistributor
```

---

## Token Model

- Total supply: 69,000,000,000 (69 billion)
- 42B seeded to source chain pool, 13.5B to each remote chain pool
- Standard: ERC-20 plus LayerZero OFT for native cross-chain transfers
- Liquidity locked permanently at launch
- No presale, no bonding curve, no vesting
- Pure AMM pricing from block one

---

## Deployed Contract Addresses

All contracts deployed at the same address on Ethereum (1), Base (8453), and Robinhood Chain (4663).

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

### DVN and Executor

| Contract | Address |
|---|---|
| MultiPools DVN | `0x829310352947Cc868c910DD133038Bd837EF0000` |
| DVN Configurator | `0x87BCc0dD2d3e86cffbFF2e4059D2d200C4140000` |
| MultiPools Executor | `0x8673aFB1196d09F09957F25BFBC939152b4E0000` |

### Uniswap v4 Pool Managers

| Chain | Address |
|---|---|
| Ethereum | `0x000000000004444c5dc75cB358380D2e3dE08A90` |
| Base | `0x498581fF718922c3f8e6A244956aF099B2652b2b` |
| Robinhood Chain | `0x8366a39CC670B4001A1121B8F6A443A643e40951` |

---

## Technical Stack

- Solidity 0.8.26, viaIR, optimizer 200 runs, EVM Cancun
- Foundry: unit tests, integration tests, invariant tests
- Uniswap v4 core and periphery
- LayerZero V2 OApp for cross-chain messaging
- Custom DVN with six per-route isolated wallets
- OpenZeppelin ERC-20 and UUPS upgradeable proxy

---

## Networks

| Network | Chain ID | LayerZero EID |
|---|---|---|
| Ethereum | 1 | 30101 |
| Base | 8453 | 30184 |
| Robinhood Chain | 4663 | 30416 |
