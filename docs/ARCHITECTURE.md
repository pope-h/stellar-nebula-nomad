# Nebula Nomad: Architecture & System Design

## System Architecture Diagram

```
╔════════════════════════════════════════════════════════════════════════════╗
║                         NEBULA NOMAD ECOSYSTEM                             ║
╚════════════════════════════════════════════════════════════════════════════╝

                           ┌─ Player Wallets ─┐
                           │  (Stellar PKC)   │
                           └────────┬──────────┘
                                    │
                  ┌─────────────────┼─────────────────┐
                  │                 │                 │
                  ▼                 ▼                 ▼
        ┌──────────────────┐ ┌───────────────┐ ┌───────────────┐
        │ Web Frontend     │ │  Mobile App   │ │    CLI        │
        │  (React/Vue)     │ │  (Native)     │ │  (Soroban)    │
        └────────┬─────────┘ └────────┬──────┘ └───────┬───────┘
                 │                    │                │
                 └────────────────────┼────────────────┘
                                      │ HTTPS/JSON-RPC
                    ┌─────────────────▼──────────────────┐
                    │   Stellar RPC Endpoint             │
                    │  (rpc-futurenet.stellar.org)       │
                    └────────────────┬───────────────────┘
                                     │
        ╔════════════════════════════▼════════════════════════════╗
        ║          STELLAR SOROBAN LAYER-1 CONTRACTS             ║
        ╠════════════════════════════════════════════════════════╣
        ║                                                        ║
        ║  ┌──────────────────────────────────────────────────┐  ║
        ║  │ NEBULA EXPLORER CONTRACT                         │  ║
        ║  │ ─────────────────────────────────────────────────│  ║
        ║  │ scan_nebula(region_id: u64) → NebulaScan         │  ║
        ║  │                                                  │  ║
        ║  │ Logic:                                           │  ║
        ║  │  • seed = region_id ⊕ ledger_sequence            │  ║
        ║  │  • density = seed % 100                          │  ║
        ║  │  • color = ["violet", "cyan", ...][seed % 4]     │  ║
        ║  │  • resources classification based on density     │  ║
        ║  └──────────────────────────────────────────────────┘  ║
        ║                           ▲                            ║
        ║                           │                            ║
        ║  ┌──────────────────────────────────────────────────┐  ║
        ║  │ RESOURCE MINTER CONTRACT                         │  ║
        ║  │ ─────────────────────────────────────────────────│  ║
        ║  │ mint_resource(player, type, qty) → Resource      │  ║
        ║  │                                                  │  ║
        ║  │ Features:                                        │  ║
        ║  │  • Sequential resource IDs                       │  ║
        ║  │  • Owner tracking (immutable ledger)             │  ║
        ║  │  • Tradeable on DEXs                             │  ║
        ║  │  • Multiple resource types supported             │  ║
        ║  └──────────────────────────────────────────────────┘  ║
        ║                           ▲                            ║
        ║                           │                            ║
        ║  ┌──────────────────────────────────────────────────┐  ║
        ║  │ SHIP REGISTRY CONTRACT (NFT)                     │  ║
        ║  │ ─────────────────────────────────────────────────│  ║
        ║  │ register_ship(owner, name) → Ship                │  ║
        ║  │ upgrade_ship(ship_id, type) → Ship (upgraded)    │  ║
        ║  │                                                  │  ║
        ║  │ Features:                                        │  ║
        ║  │  • Unique NFT per ship                           │  ║
        ║  │  • Owner-based access control                    │  ║
        ║  │  • Level & capability progression                │  ║
        ║  │  • Transfer/trade support                        │  ║
        ║  └──────────────────────────────────────────────────┘  ║
        ║                           ▲                            ║
        ║                           │ Cross-contract calls       ║
        ║  ┌──────────────────────────────────────────────────┐  ║
        ║  │ SHARED STATE (Soroban Persistent Storage)        │  ║
        ║  │ ─────────────────────────────────────────────────│  ║
        ║  │ • Player profiles (ownership, stats)             │  ║
        ║  │ • Ship registry (metadata, upgrades)             │  ║
        ║  │ • Resource ledger (immutable history)            │  ║
        ║  │ • Leaderboards (top explorers)                   │  ║
        ║  └──────────────────────────────────────────────────┘  ║
        ║                                                        ║
        ║  ┌──────────────────────────────────────────────────┐  ║
        ║  │ SOROBAN RUNTIME                                  │  ║
        ║  │ ─────────────────────────────────────────────────│  ║
        ║  │ • WebAssembly (WASM) execution environment       │  ║
        ║  │ • Deterministic contract behavior                │  ║
        ║  │ • Ledger seeding for verifiable randomness       │  ║
        ║  └──────────────────────────────────────────────────┘  ║
        ║                                                        ║
        ╚════════════════════════════════════════════════════════╝
                                  ▲
                                  │ Transactions
                    ┌─────────────┴──────────────┐
                    │                            │
        ┌───────────▼──────────┐    ┌──────────▼─────────┐
        │ Stellar Mainnet      │    │  Stellar Futurenet │
        │ (Production)         │    │  (Testnet)         │
        │                      │    │                    │
        │ • Public ledger      │    │ • Development      │
        │ • Real assets        │    │ • Safe testing     │
        │ • Mainnet consensus  │    │ • Futurenet tokens │
        └──────────────────────┘    └────────────────────┘


╔════════════════════════════════════════════════════════════════════════════╗
║                      GAME STATE FLOW DIAGRAM                               ║
╚════════════════════════════════════════════════════════════════════════════╝

Player Starts
    │
    ▼
1. REGISTRATION PHASE
    ├─ Create Stellar account
    ├─ Receive default ship NFT
    └─ 0 resources collected

    │
    ▼
2. EXPLORATION PHASE
    ├─ Select region to scan
    ├─ scan_nebula(region_id) called
    ├─ Contract generates: density, color, resource_level
    └─ Player observes nebula characteristics

    │
    ▼
3. COLLECTION PHASE
    ├─ Decide to collect resources
    ├─ mint_resource(resource_type, quantity) called
    ├─ New resource NFT created
    └─ Ledger records ownership

    │
    ▼
4. UPGRADE PHASE
    ├─ Collect multiple resources
    ├─ Trade for upgrade materials
    ├─ upgrade_ship(upgrade_type) called
    ├─ Ship stats improved (scan_range, level)
    └─ Return to EXPLORATION with better ship

    │
    ▼
5. TRADING PHASE
    ├─ List resources/ships on DEX
    ├─ Exchange with other players
    ├─ Participate in secondary market
    └─ Value creation from gameplay

    │
    ▼
Loop: Steps 2-5 repeat indefinitely


╔════════════════════════════════════════════════════════════════════════════╗
║                    DATA FLOW: SCANNING A NEBULA                            ║
╚════════════════════════════════════════════════════════════════════════════╝

User Action                Input                Contract Processing
─────────────              ─────                ──────────────────

Player clicks
"Scan Nebula"  ──────► region_id: 42 ──────┐
                                             │
                                             ▼
                                    ┌─────────────────┐
                                    │ Get ledger data │
                                    │ seq: 1739609600 │
                                    └────────┬────────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
                                    │ Mix inputs:      │
                                    │ seed = 42 ⊕      │
                                    │ 1739609600       │
                                    └────────┬─────────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
                                    │ Generate props:  │
                                    │ density = %100   │
                                    │ color = %4       │
                                    └────────┬─────────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
                                    │ Classify:        │
                                    │ if 0-33: sparse  │
                                    │ if 34-66: mod    │
                                    │ if 67-100: abund │
                                    └────────┬─────────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
Return to                           │ Store in ledger  │
Player:                             │ (immutable)      │
"Magenta nebula" ◄───────────────┤                    │
"Abundant resources"                │ Return to caller │
"Scan confirmed"                    └──────────────────┘


╔════════════════════════════════════════════════════════════════════════════╗
║                     CONTRACT INTERACTION MATRIX                            ║
╚════════════════════════════════════════════════════════════════════════════╝

                 │ Nebula   │ Resource │ Ship     │ Ledger
                 │ Explorer │ Minter   │ Registry │ State
─────────────────┼──────────┼──────────┼──────────┼─────────
Nebula Explorer  │    •     │    ◄─    │          │    ◄─
Resource Minter  │          │    •     │          │    ◄─
Ship Registry    │          │          │    •     │    ◄─
─────────────────┼──────────┼──────────┼──────────┼─────────

Legend:
• = Primary module logic
◄─ = Read from state
→ = Write to state


╔════════════════════════════════════════════════════════════════════════════╗
║                    DEPLOYMENT ARCHITECTURE                                 ║
╚════════════════════════════════════════════════════════════════════════════╝

Development Environment
┌─────────────────────────┐
│ Local Machine           │
│ • cargo build           │
│ • cargo test            │
│ • soroban invoke        │
└────────────┬────────────┘
             │
             ▼
Integration Testing
┌─────────────────────────┐
│ Stellar Futurenet       │
│ • GitHub Actions CI/CD  │
│ • Run full test suite   │
│ • Contract optimization │
└────────────┬────────────┘
             │
             ▼
Staging Environment
┌─────────────────────────┐
│ Stellar Testnet         │
│ • Pre-deployment check  │
│ • User acceptance tests │
│ • Load testing          │
└────────────┬────────────┘
             │
             ▼
Production Environment
┌─────────────────────────┐
│ Stellar Mainnet         │
│ • Live deployment       │
│ • Real assets           │
│ • Player data           │
└─────────────────────────┘

```

## Key Architectural Decisions

### 1. Ledger-Seeded Procedural Generation

- **Why**: Deterministic + verifiable (no server manipulation)
- **How**: `seed = region_id ⊕ ledger_sequence`
- **Benefit**: Players can always verify nebula properties

### 2. Modular Contract Design

- **Why**: Separation of concerns, easier to audit
- **How**: Nebula Explorer, Resource Minter, Ship Registry as separate modules
- **Benefit**: Each contract can be upgraded/replaced independently

### 3. Immutable Ledger Records

- **Why**: Guarantees fairness and prevents exploits
- **How**: All discoveries and creations are permanent
- **Benefit**: Players trust the system

### 4. Asset-Native Design

- **Why**: Leverage Stellar's native token & NFT support
- **How**: Resources and Ships are Soroban tokens
- **Benefit**: DEX integration, marketplace compatibility

### 5. No Combat/PvP

- **Why**: Reduces toxicity, enables cooperation
- **How**: Only exploration & resource collection mechanics
- **Benefit**: Inclusive community, lower barrier to entry

---

## Security Considerations

| Risk                       | Mitigation                                       |
| -------------------------- | ------------------------------------------------ |
| Ledger manipulation        | Deterministic seeding makes it impossible        |
| Double-spending resources  | Immutable ledger + ownership tracking            |
| Unauthorized ship upgrades | Function access control via owner validation     |
| Contract bugs              | Comprehensive test suite + audits before mainnet |
| Network attacks            | Stellar Foundation's infrastructure security     |
