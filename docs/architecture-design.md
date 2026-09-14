# Ondo V1 合约架构设计文档

## 1. 项目概述

Ondo V1 是一个基于以太坊的**结构化金融产品协议**，核心功能是将 AMM 流动性池（LP）头寸拆分为**优先级（Senior）和劣后级（Junior）两个层级**，为投资者提供不同风险/收益特征的固定期限金库产品。

- **项目名称**: ondo-contracts v0.0.1
- **开发框架**: Hardhat + TypeScript
- **Solidity 版本**: 0.8.3（主合约）/ 0.5.16（DAO 合约）
- **依赖版本**: OpenZeppelin 4.0.0, Uniswap V2, SushiSwap
- **多链支持**: Ethereum、BSC（PancakeSwap）、Polygon（QuickSwap）

---

## 2. 系统架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Registry (核心)                              │
│  ┌───────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │ GOVERNANCE │  │   PANIC    │  │  GUARDIAN  │  │   DEPLOYER   │  │
│  │   ROLE     │  │   ROLE     │  │   ROLE     │  │    ROLE      │  │
│  └───────────┘  └────────────┘  └────────────┘  └──────────────┘  │
│  全局暂停 / 授权管理 / 参数配置                                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌──────────────┐ ┌───────────┐ ┌──────────────────┐
     │ AllPairVault │ │ Rollover  │ │   Strategy 合约    │
     │  (主金库)     │ │  Vault    │ │  (AMM 策略层)      │
     └──────┬───────┘ └─────┬─────┘ └────────┬─────────┘
            │               │                │
            ▼               ▼                ▼
     ┌──────────────┐ ┌───────────┐  ┌──────────────────┐
     │ TrancheToken │ │ Tranche   │  │ 外部 AMM/DEX      │
     │ (分级代币)    │ │ Token     │  │ Uniswap/Sushi/... │
     │ (Clones)     │ │ (Clones)  │  │ + 收益农场         │
     └──────────────┘ └───────────┘  └──────────────────┘
```

---

## 3. 合约继承体系

```
Initializable + ReentrancyGuard + Pausable
  └── OndoRegistryClientInitializable (抽象基类)
        └── OndoRegistryClient (构造器初始化版本)
              ├── Registry                        ← 中央注册表 + AccessControl
              ├── AllPairVault                    ← 主金库合约
              ├── RolloverVault                   ← 滚动金库
              ├── SampleFeeCollector              ← 手续费收集
              └── BasePairLPStrategy (抽象)       ← LP 策略基类
                    ├── UniswapStrategy
                    ├── AUniswapStrategy (抽象)
                    │     ├── ASushiswapStrategy (抽象)
                    │     │     ├── SushiStrategyLP
                    │     │     ├── SushiStakingV2Strategy
                    │     │     ├── AlchemixLPStrategy
                    │     │     ├── DopexStrategy
                    │     │     ├── EdenStrategy
                    │     │     └── QuickswapStrategyLP (Polygon)
                    │     └── BondStrategy
                    ├── PancakeStrategy (BSC)
                    └── PancakeStrategyLP (BSC)

ERC20Upgradeable + ITrancheToken + OwnableUpgradeable
  └── TrancheToken (最小代理克隆)

Ownable (自定义 ERC20)
  └── Ondo (治理代币)

AccessControl
  └── StakingPools (MasterChef 风格质押池)
```

---

## 4. 核心合约详解

### 4.1 Registry — 中央注册表

**文件**: `contracts/Registry.sol`
**职责**: 全局访问控制、暂停机制、参数管理

- 继承 `OndoRegistryClient` + `IRegistry` + `AccessControl`
- 维护全局 `paused` 状态，所有协议合约通过 `OndoRegistryClientInitializable.paused()` 检查
- 管理授权合约列表（Vault、Strategy、Rollover）
- 存储全局参数：`denominator`（精度分母）、`weth` 地址
- 控制分级代币的启用/禁用与回收

### 4.2 AllPairVault — 主金库

**文件**: `contracts/AllPairVault.sol`
**职责**: 金库生命周期管理、用户存取、投资收益分配

核心功能：
- **创建金库** (`createVault`): 指定交易对、策略、容量参数，通过 `Clones.cloneDeterministic()` 部署 Senior/Junior 分级代币
- **存款** (`deposit` / `depositETH` / `depositLp`): 用户存入资产，Senior 存款受 `seniorCap` 上限约束
- **投资** (`invest`): 策略师调用，通过 Strategy 合约向 AMM 添加流动性
- **赎回** (`redeem`): 到期后策略师调用，Strategy 移除流动性
- **提取** (`withdraw` / `withdrawETH` / `claim` / `claimETH`): 用户按比例提取本金+收益
- **手续费**: 对 Junior 超额收益收取绩效费，通过 `IFeeCollector` 处理

### 4.3 RolloverVault — 滚动金库

**文件**: `contracts/RolloverVault.sol`
**职责**: 自动续投机制，将到期收益滚动投入下一轮金库

- 通过 `newRollover()` 创建滚动实例
- 按 "轮次"（Round）追踪每个层级的金库周期
- `migrate()` 由策略师调用，在轮次间转移资金
- 每轮独立部署分级代币克隆
- 支持 `SlippageSettings` 滑点保护

### 4.4 TrancheToken — 分级代币

**文件**: `contracts/TrancheToken.sol`
**职责**: 代表金库中 Senior/Junior 层级的 ERC-20 代币

- 继承 `ERC20Upgradeable`，通过 EIP-1167 最小代理部署
- 支持 `mint` / `burn` / `destroy` 操作
- 由 Vault 合约通过 `VAULT_ROLE` / `ROLLOVER_ROLE` 权限控制铸造/销毁

### 4.5 OndoRegistryClientInitializable — 协议基类

**文件**: `contracts/OndoRegistryClientInitializable.sol`
**职责**: 所有协议合约的公共基础

- 存储 `registry` 引用和 `denominator` 常量
- `isAuthorized` 修饰器：检查调用者是否通过 Registry 授权
- `paused()` 覆写：合并 Registry 全局暂停和本地暂停状态
- `rescueTokens()`: Guardian 紧急代币救援

---

## 5. 策略模式

### 5.1 策略接口 (IStrategy)

**文件**: `contracts/interfaces/IStrategy.sol`

定义统一的策略接口，核心结构：

```solidity
struct Vault {
    address origin;      // 金库来源地址
    address pool;        // AMM 池地址
    address senior;      // 优先级代币
    address junior;      // 劣后级代币
    uint256 shares;      // LP 份额
    uint256 seniorExcess;// 优先级超额
    uint256 juniorExcess;// 劣后级超额
}
```

核心方法：`addVault` / `addLp` / `removeLp` / `invest` / `redeem` / `withdrawExcess`

### 5.2 策略继承层次

| 层级 | 合约 | 职责 |
|------|------|------|
| 抽象基类 | `BasePairLPStrategy` | Vault 映射管理、LP 操作、`onlyOrigin` 访问控制 |
| Uniswap 抽象 | `AUniswapStrategy` | Uniswap V2 风格 AMM 交互（路由、储备量、金额计算） |
| Sushi 抽象 | `ASushiswapStrategy` | 在 Uniswap 基础上增加 farming/compounding（质押、收割、复投） |
| 具体实现 | 各链具体策略 | 对接特定协议的外部合约 |

### 5.3 多链策略部署

| 链 | DEX | 策略合约 | 收益农场 |
|----|-----|---------|---------|
| Ethereum | Uniswap V2 | `UniswapStrategy` | — |
| Ethereum | SushiSwap | `SushiStrategyLP` | MasterChef + xSUSHI |
| Ethereum | SushiSwap V2 | `SushiStakingV2Strategy` | MasterChef V2 + IRewarder |
| Ethereum | BarnBridge | `BondStrategy` | BarnBridge Staking |
| Ethereum | Alchemix | `AlchemixLPStrategy` | Alchemix StakingPools |
| Ethereum | Dopex | `DopexStrategy` | Dopex StakingRewards |
| Ethereum | Eden | `EdenStrategy` | Eden RewardsManager |
| BSC | PancakeSwap | `PancakeStrategy` / `PancakeStrategyLP` | PancakeSwap MasterChef |
| Polygon | QuickSwap | `QuickswapStrategyLP` | QuickSwap Staking + dQUICK |

---

## 6. 金库生命周期状态机

```
  Inactive ──→ Deposit ──→ Live ──→ Withdraw
     │            │           │          │
     │ 创建金库    │ 用户存款   │ 投资AMM  │ 赎回/提取
     │ 部署代币    │ Senior上限 │ 策略执行  │ 分配收益
```

**状态转换条件**:
- `Inactive → Deposit`: 金库创建完成
- `Deposit → Live`: 策略师调用 `invest()`
- `Live → Withdraw`: 到期后策略师调用 `redeem()`

### 投资者记账 — 前缀和算法

`OLib` 库实现了高效的前缀和记账系统：

- `Investor` 结构体维护 `userSums[]` 和 `prefixSums[]` 数组
- `getInvestedAndExcess()` 通过二分查找 (`findUpperBound`) 在 O(log n) 内计算每个投资者的份额
- 避免遍历所有存款记录，显著提升 gas 效率

---

## 7. 访问控制体系

### 7.1 角色定义

| 角色 | 标识 | 权限范围 |
|------|------|---------|
| `GOVERNANCE_ROLE` | 治理角色 | 最高管理权限，管理 Registry 参数，管理其他角色的分配 |
| `PANIC_ROLE` | 紧急角色 | 触发全局暂停 |
| `GUARDIAN_ROLE` | 守护者角色 | 解除暂停、调用紧急函数 (`excall`)、救援代币 |
| `DEPLOYER_ROLE` | 部署者角色 | 创建金库，管理 VAULT/ROLLOVER/STRATEGY 角色分配 |
| `CREATOR_ROLE` | 创建者角色 | 金库创建权限 |
| `STRATEGIST_ROLE` | 策略师角色 | 调用 `invest()` / `redeem()`，管理策略操作 |
| `VAULT_ROLE` | 金库角色 | 授予 AllPairVault 合约，允许铸造/销毁分级代币 |
| `ROLLOVER_ROLE` | 滚动角色 | 授予 RolloverVault 合约 |
| `STRATEGY_ROLE` | 策略角色 | 授予策略合约 |

### 7.2 角色管理关系

```
GOVERNANCE_ROLE (admin)
  ├── DEPLOYER_ROLE
  ├── CREATOR_ROLE
  ├── PANIC_ROLE
  └── GUARDIAN_ROLE

DEPLOYER_ROLE (admin)
  ├── VAULT_ROLE
  ├── ROLLOVER_ROLE
  └── STRATEGY_ROLE
```

### 7.3 暂停机制

- **全局暂停**: `Registry.paused()` 被所有协议合约检查
- **本地暂停**: 单个合约可通过 `Pausable` 独立暂停
- `PANIC_ROLE` 可暂停，`GUARDIAN_ROLE` 可解除暂停
- 暂停时所有状态修改操作被 `isAuthorized` 修饰器阻止

### 7.4 紧急响应

- `AllPairVault.excall()`: Guardian 可执行任意外部调用
- `rescueTokens()`: Guardian 可回收卡在合约中的代币

---

## 8. 升级与扩展模式

### 8.1 EIP-1167 最小代理

- `TrancheToken` 作为实现合约部署一次
- 每个金库的 Senior/Junior 代币通过 `Clones.cloneDeterministic()` 部署
- 大幅降低金库创建的 gas 成本
- 使用 CREATE2 确保地址确定性

### 8.2 可升级代币

- `TrancheToken` 继承 `ERC20Upgradeable` + `OwnableUpgradeable`
- 使用 `Initializable` 模式替代构造器

### 8.3 策略可插拔

- 新 AMM 集成只需实现 `IStrategy` 接口并注册
- 策略在金库创建时设定，创建后不可更改
- 支持通过继承复用公共逻辑

### 8.4 DAO 治理升级

- `GovernorBravo` + `Timelock` 实现去中心化治理
- 提案 → 投票 → 排队 → 执行的时间延迟流程
- DAO 可通过治理提案升级协议合约

---

## 9. 外部集成

### 9.1 DEX/AMM 集成

| 协议 | 链 | 外部合约 |
|------|-----|---------|
| Uniswap V2 | Ethereum | `IUniswapV2Router02`, `UniswapV2Library` |
| SushiSwap | Ethereum | Router, `IMasterChef`, `ISushiBar` (xSUSHI) |
| SushiSwap V2 | Ethereum | `IMasterChefV2`, `IRewarder` |
| BarnBridge | Ethereum | `Staking`, `YieldFarmLP` |
| Alchemix | Ethereum | `IStakingPools` |
| Dopex | Ethereum | `IStakingRewards` |
| Eden | Ethereum | `IRewardsManager` |
| PancakeSwap | BSC | Router, `IPancakeMasterChef` |
| QuickSwap | Polygon | Router, `IStakingRewards`, `IDragonLair` (dQUICK) |

### 9.2 代币集成

- **WETH**: 原生 ETH 通过 WETH 包装后入金库
- **USDT 兼容**: `OndoSaferERC20.ondoSafeIncreaseAllowance()` 处理需要先重置授权额的代币

### 9.3 无预言机依赖

协议不使用价格预言机（如 Chainlink），定价通过 AMM 池的隐含比率完成。

---

## 10. DAO 治理架构

| 合约 | 文件 | 说明 |
|------|------|------|
| `Timelock` | `contracts/dao/Timelock.sol` | Compound 风格时间锁，延迟执行治理操作 |
| `GovernorBravoDelegate` | `contracts/dao/GovernorBravoDelegate.sol` | 治理逻辑实现 |
| `GovernorBravoDelegator` | `contracts/dao/GovernorBravoDelegator.sol` | 治理代理，委托至 Delegate |
| `GovernorBravoInterfaces` | `contracts/dao/GovernorBravoInterfaces.sol` | 接口定义 |

治理代币 `Ondo` (`contracts/tokens/Ondo.sol`) 提供投票权，`StakingPools` (`contracts/tokens/StakingPools.sol`) 提供 MasterChef 风格质押激励。

---

## 11. 部署架构

### 11.1 部署脚本序列

| 序号 | 脚本 | 部署内容 |
|------|------|---------|
| 001 | `deploy_core.ts` | Registry, TrancheToken 实现合约 |
| 002 | `deploy_base.ts` | AllPairVault 基础部署 |
| 003 | `deploy_uniswap_strategy.ts` | Uniswap V2 策略 |
| 004 | `deploy_sushiswap_strategy.ts` | SushiSwap 策略 |
| 005 | `deploy_alchemix.ts` | Alchemix 策略 |
| 006 | `deploy_rollover.ts` | RolloverVault |
| 007 | `deploy_ondo_token.ts` | ONDO 治理代币 |
| 008 | `deploy_staking_pools.ts` | ONDO 质押池 |
| 009 | `deploy_sushi_v2.ts` | SushiSwap MasterChef V2 策略 |
| 010 | `deploy_reward_helper.ts` | 奖励辅助合约 |
| 011 | `deploy_rewarder.ts` | Rewarder 合约 |
| 013 | `deploy_bond_strategy.ts` | BarnBridge 策略 |
| 014 | `deploy_dao.ts` | GovernorBravo + Timelock |
| 015 | `deploy_eden_strategy.ts` | Eden 策略 |

### 11.2 多链部署

- `scripts/utils/multichain.ts` 管理各链的部署路径、合约目录和测试目录
- 各链独立目录：`contracts/`（Ethereum）、`bsc/contracts/`（BSC）、`polygon/contracts/`（Polygon）
- 生产环境部署脚本位于 `deploy/production/`

### 11.3 编译配置

- Solidity 0.8.3（主合约）+ 0.5.16（DAO 合约）
- 优化器：启用，100 次运行
- 网络：hardhat（主网分叉）、ropsten、rinkeby、mainnet、bsc、bsc-testnet、polygon、mumbai

---

## 12. 测试架构

- **框架**: Hardhat + `@nomiclabs/hardhat-waffle` (Chai)
- **语言**: TypeScript
- **测试文件**: 60+ 个 `.spec.ts` 文件
- **主网分叉**: 使用 Hardhat forking 模式进行真实集成测试

核心测试覆盖：
- 金库生命周期 (`vault.spec.ts`)
- 注册表访问控制 (`registry.spec.ts`)
- 滚动金库 (`rollover.spec.ts`)
- 分级代币 (`token.spec.ts`)
- 质押池 (`staking.spec.ts`)
- 手续费 (`fee.spec.ts`)
- 提取/申领 (`claim.spec.ts`)
- 紧急救援 (`rescue.spec.ts`)
- CREATE2 地址 (`create2.spec.ts`)
- 各策略独立测试

---

## 13. 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 分级代币部署 | EIP-1167 最小代理 | 大幅降低 gas，确定性地址 |
| 投资者记账 | 前缀和 + 二分查找 | O(log n) 查询，避免线性遍历 |
| 策略扩展 | 继承层次 + 接口抽象 | 复用 AMM 公共逻辑，减少重复代码 |
| 访问控制 | OpenZeppelin AccessControl | 成熟的 RBAC 方案，支持角色层级 |
| 暂停机制 | 全局 + 本地双层 | Registry 全局急停 + 合约级细粒度控制 |
| 多链支持 | 独立目录 + 共享基类 | 各链策略独立演进，共享核心逻辑 |
| 定价机制 | AMM 隐含比率 | 无需预言机，降低外部依赖和攻击面 |
| 治理 | GovernorBravo + Timelock | Compound 验证过的治理方案 |
