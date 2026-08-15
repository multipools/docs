# Deployed Addresses

All contracts are deployed at the same address on Ethereum (chain ID 1), Base (chain ID 8453), and Robinhood Chain (chain ID 4663) unless noted.

Last updated: August 2026.

---

## Core Protocol

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

## DVN and Executor

| Contract | Address |
|---|---|
| MultiPools DVN | `0x829310352947Cc868c910DD133038Bd837EF0000` |
| DVN Configurator | `0x87BCc0dD2d3e86cffbFF2e4059D2d200C4140000` |
| MultiPools Executor | `0x8673aFB1196d09F09957F25BFBC939152b4E0000` |

## Hook Init Hashes

The hook init hash is used off-chain to mine a valid hook salt. It depends on the factory address and is therefore chain-specific.

| Chain | Hook Init Hash |
|---|---|
| Ethereum | `0x71a99e0fb505619dbc7fcd3e99ecacc44b26cca89678252ce49eb492d54cb764` |
| Base | `0x0dfb2f36ef9f8a65e2d37710ee8e3a37ee43a7e84532504e4c43c40ed6eb4cfa` |
| Robinhood Chain | `0x563864893beb9011fdf0a78724d32d3d28aacf409f71d9546c7cd854b7508ad5` |

## Uniswap v4 Pool Managers

| Chain | Address |
|---|---|
| Ethereum | `0x000000000004444c5dc75cB358380D2e3dE08A90` |
| Base | `0x498581fF718922c3f8e6A244956aF099B2652b2b` |
| Robinhood Chain | `0x8366a39CC670B4001A1121B8F6A443A643e40951` |

## LayerZero Endpoints

| Chain | EID | Address |
|---|---|---|
| Ethereum | 30101 | `0x1a44076050125825900e736c501f859c50fE728c` |
| Base | 30184 | `0x1a44076050125825900e736c501f859c50fE728c` |
| Robinhood Chain | 30416 | `0x6F475642a6e85809B1c36Fa62763669b1b48DD5B` |

## Block Explorers

| Chain | Explorer |
|---|---|
| Ethereum | https://etherscan.io |
| Base | https://basescan.org |
| Robinhood Chain | https://robinhoodchain.blockscout.com |
