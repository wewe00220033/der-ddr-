# ⚔️ RuneLite Microbot F2P Automation & Progression Suite

A modular, event-driven suite of plugins and scripts for RuneLite with Microbot integration, designed for autonomous task management, quest completion, and early-game skill milestone progression in Old School RuneScape (OSRS).

---

## 📦 Directory Structure

```
der-ddr-/
├── net/                               # Unified RuneLite client plugin package tree
│   └── runelite/client/plugins/microbot/
│       ├── stronghold/                # Stage 0.1: Stronghold of Security navigation & rewards
│       ├── tradeunlock/               # Stage 0.2: 12 QP Trade Unlock quests suite
│       └── totallevel/                # Stage 0.3: Total Level 100 safe skill progression
│
├── microbot_stage0_quests/            # Standalone package for F2P Trade-Restriction Quests
│   ├── TradeUnlockConfig.java
│   ├── TradeUnlockOverlay.java
│   ├── TradeUnlockPlugin.java
│   └── TradeUnlockScript.java
│
├── microbot_stage0_stronghold/        # Standalone package for Stronghold traversal
│   ├── StrongholdConfig.java
│   ├── StrongholdOverlay.java
│   ├── StrongholdPlugin.java
│   └── StrongholdScript.java
│
├── microbot_stage0_totallevel/        # Standalone package for Total Level progression
│   ├── TotalLevelChecks.java          # 122 offline verification assertions
│   ├── TotalLevelConfig.java
│   ├── TotalLevelOverlay.java
│   ├── TotalLevelPlugin.java
│   └── TotalLevelScript.java
│
├── OSRS/                              # Complete OSRS strategic workspace & documentation
│   ├── 01_دليل_طرق_الربح_والمصادر_المعتمدة.md
│   ├── OSRS_Economic_Modeling_Study_AR.md
│   ├── README.md
│   ├── osrs_ge_radar.py               # Live Grand Exchange margin & market scanner
│   ├── جدول_الخطوات_وبروتوكول_العمل_المشترك.md
│   ├── شاطر.md
│   └── جبان.md
│
└── index.html                         # Web portal / dashboard
```

---

## 🧩 Stage 0 Modules Overview

### 1. `stronghold` (Security Stronghold Navigator)
- **Objective**: Autonomous traversal of all 4 floors of the Stronghold of Security (Vault of War, Catacomb of Famine, Pit of Pestilence, Cradle of Life).
- **Core Features**:
  - Deterministic door question answering based on upstream question tables.
  - Non-blocking pathfinding (`walkStep`) with corridor bounds.
  - Verification of 10,000 GP reward and emote unlocks.

### 2. `tradeunlock` (F2P Trade-Restriction Unlock Quests)
- **Objective**: Automated completion of the core 4 F2P quests awarding 12 Quest Points total:
  - *Cook's Assistant* (1 QP)
  - *Romeo & Juliet* (5 QP)
  - *Sheep Shearer* (1 QP)
  - *Goblin Diplomacy* (5 QP)
- **Core Features**:
  - Pre-sourcing item verification (zero mid-quest resource gathering).
  - Varp and Varbit milestone tracking (`varp 29`, `varp 144`, `varp 179`, `varbit 2378`).
  - Dialogue state machine with dynamic widget option resolving.

### 3. `totallevel` (Total Level 100 Progression)
- **Objective**: Structured, low-risk skill progression reaching Total Level 100.
- **Core Features**:
  - Balanced multi-skill rotation (Woodcutting, Firemaking, Fishing, Cooking).
  - Game-tick total-level latch to preempt background actions immediately upon reaching target.
  - Localized area safety bounds (Lumbridge Castle & riverbanks) avoiding hostile NPC corridors.

---

## 🛠️ Build & Installation

### Requirements
- Java Development Kit (JDK) 11 or higher
- RuneLite client core with Microbot API binaries

### Integration
Place the packages into your RuneLite client plugin tree under:
`net.runelite.client.plugins.microbot`

---

## 📜 Compliance & Security
- **Bolla Constitution Law 8 Compliance**: Strictly zero hardcoded credentials, secret keys, or environment tokens are committed to this repository. All private keys are managed via local `.gitignore` rules.
