# 剧本15 反汇编批次 03

> 2026-11-02。本批把「稽古设施等级 ↔ 进度条 ↔ 溢出事件」这条链完整串起来了。

---

## 1. 找到了：`ActGauge` 与「溢出事件」

搜 `keyword=ActGauge` → 71 个方法，29 个类。核心两个字段表：

### 1.1 `ObscuredSingleModeFestaActInfo` —— 一个演目的完整状态（7 字段）

| 偏移 | 字段 | 类型 |
|--:|:--|:--|
| +16 | **ActType** | ObscuredInt |
| +36 | **ActGauge** | ObscuredInt（当前进度条） |
| +56 | **Level** | ObscuredInt（当前设施等级） |
| +76 | **NeedActGauge** | ObscuredInt（升级所需） |
| +96 | ActTrainingState | ObscuredInt |
| +116 | **AdviceId** | ObscuredInt（该演目挂的教え） |
| +136 | FriendOutingGainActLevel | ObscuredInt |

**偏移 16 → 36 → 56 → 76 → 96 → 116 → 136，每次 +20。**

→ **证实 `ObscuredInt` 占 20 字节**（4 值 + 16 key），批次 02 那条推断成立。

这张表就是「5 个演目各自的进度条与等级」，与用户描述完全对应。

### 1.2 `ObscuredSingleModeFestaNotUpParameterInfo` —— 溢出事件标记（3 字段）

| 偏移 | 字段 | 类型 |
|--:|:--|:--|
| +16 | **IsOverflowActGaugeEventGain** | ObscuredBool |
| +28 | **IsOverflowActLevelEventGain** | ObscuredBool |
| +40 | NotLevelUpActTypeArray | ObscuredInt[] |

**两个关键布尔：**

- `IsOverflowActGaugeEventGain` —— **进度条溢出时的事件奖励**
- `IsOverflowActLevelEventGain` —— **等级溢出时的事件奖励**

> 这就是「5 个进度条攒满 → 触发稽古设施的训练」的代码落点。
> 溢出（overflow）触发事件，事件再结算奖励。

---

## 2. `ActGauge` 相关类清单（按职责分层）

| 层 | 类 | 方法 |
|:--|:--|:--|
| **逻辑实体** | `ActEntity` | `get_CurrentActGauge` / `get_ActGaugeLimit` / `get_IsActGaugeLimit` |
| **网络结构** | `ObscuredSingleModeFestaActInfo` | get/set `ActGauge` / `NeedActGauge` |
| **网络结构** | `ObscuredSingleModeFestaPredictGainActGauge` | get/set `GainActGauge` |
| **数据集合** | `WorkSingleModeScenarioFestaDataSet` | `CommandGainActGaugeArray` |
| **服务** | `GainedActGaugeService` | `GetActGaugeGainInfoDict`（静态） |
| **命令** | `CommandEntity` | `get_ActGaugeGainValue` |
| **命令** | `TrainingCommandEntity` | `GetGainActGaugeAmount(ActType)` |
| **UI** | `PartsSingleModeScenarioFestaMainViewActInfo` | `UpdateGainActGaugeAmount` / `PlayActGaugeUpAnimation` / `get_IsActGaugeLimit` |
| **UI** | `TrainingCuttFestaController` | `PlayActGaugeUp` |
| **文本** | `SingleModeScenarioFestaMainTrainingMassageService` | `GetActGaugeMessageList` |

层级非常清楚：**实体 → 网络结构 → 数据集 → UI**，**中间没有"计算器"**。

---

## 3. 又一个"没有 Random"的证据

搜 `keyword=Random` → 85 个类、200 个方法，**过滤后没有一个属于 `Gallop.SingleMode.ScenarioFesta`**。

命中的全是：

| 类别 | 例子 |
|:--|:--|
| 赛马模拟 | `RaceRandom` / `IRaceRandomGenerator` |
| 角色/尾巴动画 | `RandomTailMotionController` 等一大票 |
| 扭蛋/抽选 | `MasterCharacterSystemLottery.GetRandom` |
| 先锋剧本 | `SingleModeScenarioPioneerShimaTrainingCutDomainService` |

> **剧本15 的随机判定不在客户端任何带 Random 的类里。**

结合批次 02 的「只有 ObscuredBool 结果、无 Rate 字段」：

**结论指向：追加の稽古 是否触发，由服务端决定，客户端只接收布尔。**

---

## 4. 完整机制串联（当前认知）

```
每次训练
  └─ CommandEntity.ActGaugeGainValue          ← 该命令给多少进度
     └─ TrainingCommandEntity.GetGainActGaugeAmount(ActType)  ← 按演目查字典
        └─ ActEntity.CurrentActGauge += gain
           ├─ 未到 NeedActGauge → 继续攒
           └─ 溢出 → IsOverflowActGaugeEventGain = true
                      └─ 触发该演目设施的稽古训练（UpperTrainingEvent）
                         └─ 清空进度条 → Level +1
                            └─ 若同时等级溢出 → IsOverflowActLevelEventGain = true
```

**「追加の稽古」大概率就是 `UpperTrainingEvent`，由进度条溢出触发。**

这解释了为什么它的"发生率"是 20~80%：**溢出后还要过一次概率判定**才真正发生，
而这次判定的结果就是 `IsUpperTrainingEventOccurred` 那个 ObscuredBool。

---

## 5. 结论表（累计）

| 问题 | 答案 | 强度 |
|:--|:--|:--|
| 进度条/等级存哪 | `ObscuredSingleModeFestaActInfo`（+36 进度、+56 等级、+76 需求） | 硬证据 |
| 溢出触发事件 | `IsOverflowActGaugeEventGain` / `IsOverflowActLevelEventGain` | 硬证据 |
| ObscuredInt 步长 | **20 字节** | 硬证据（两次独立验证） |
| 追加の稽古 | = `UpperTrainingEvent`，溢出后过概率 | 强推断 |
| 概率算法在客户端？ | **没有任何 Random 命中** → 倾向服务端 | 强证据 |
| 概率公式 | 仍未定位 | 未解 |

---

## 6. 批次 04 计划

| # | 目标 | 目的 |
|:-:|:--|:--|
| 1 | `GainedActGaugeService.GetActGaugeGainInfoDict` 反汇编 | 进度条计算是否含 buff 系数 |
| 2 | `ActEntity` 全方法 + 字段 | 溢出判定条件 |
| 3 | `SingleModeFestaAPI` 全部方法 | 确认请求/响应边界 |
| 4 | 搜 `keyword=Overflow` | 溢出事件完整链路 |
| 5 | 嗅探实测一次训练请求的响应体 | **直接看服务端有没有下发概率/布尔** |
