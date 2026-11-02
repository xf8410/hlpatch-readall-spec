# 剧本15 反汇编批次 02

> 2026-11-02。承接批次 01。本批重点：**谁写入 IsUpperTrainingEventOccurred**、転成自动选择逻辑、buff 状态来源。

---

## 1. 决定性突破：那个布尔是 **Obscured（反作弊加密）存储**

### 反汇编 `ObscuredSingleModeFestaCommandInfo.set_IsUpperTrainingEventOccurred`

方法地址 `0x754a513258`，前两条指令即写完：

```asm
0x0000  01 24 00 f9    STR  X1, [X0, #0x48]    ; 值  → +0x48 (72)
0x0004  02 50 00 b9    STR  W2, [X0, #0x50]    ; key → +0x50 (80)
0x0008  c0 03 5f d6    RET
```

**对应字段偏移 +72，与 `get_fields` 报的 `IsUpperTrainingEventOccurred @ 72` 完全吻合。**

### 这意味着什么

| 项 | 值 |
|:--|:--|
| 类型 | `ObscuredBool` |
| 布局 | `{ value : int/X @ +0x48, cryptoKey : int @ +0x50 }` |
| 真实值 | `value XOR key`（CodeStage AntiCheat 规则） |

**所以你在内存里直接搜 0/1 是搜不到的** —— 存的是加密后的随机值。
这解释了为什么读训练状态时字段看起来是乱的。

> 另外：偏移是 **0x48 = 72**，但 `get_fields` 报的 `CommandType @16 / CommandId @36` 是 ObscuredInt，
> 说明 **ObscuredInt 占 20 字节**（4 值 + 16 key），不是 8。这个步长必须记住，否则算偏移全错。

---

## 2. `CanExecActTrainingService`

全 13 个方法，**只有 1 个是业务方法**：

| 方法 | 作用 |
|:--|:--|
| `GetActivatedActTypeList` | 返回当前已激活的演目（ActType）列表 |

其余全是 Object 继承的样板方法。

→ 这个类**只管"哪些演目开着"，不含事件判定**。排除嫌疑。

---

## 3. `AdviceInfoRepository`

| 方法 | 作用 |
|:--|:--|
| `Get` | 取 AdviceInfoEntity |
| `GetByBackup` | 取备份 |
| `get_FestaDataSet` | 数据集合 |
| `get_ChangeParameterInfoScenarioFesta` | 参数变更信息 |

**只有 4 个业务方法，全是 getter 风格。** 没有计算逻辑 → 排除。

---

## 4. `AdviceAutoSettingService` —— 找到了「転成选择」的自动化

| 方法 | 作用 |
|:--|:--|
| **`AutoSelectFusionSet`** | **自动选择転成组合** ← 転成逻辑入口 |
| `AutoSelectAttachSetDeck` | 自动选择装备组 |
| `get_BaseAdviceVoList` | 基础 buff 列表（5 个） |
| `GetAttachSetDeckDegreeId` ×2 | 装备组等级 id |
| `GetAttachSetDeckDegreeIdForDisplay` | 显示用等级 id |

**`AutoSelectFusionSet` 是唯一和「第一年 12 月下选 2 个转成」直接对应的代码。**

不过注意：它是 **AutoPlay（自动育成）用的**，属于"帮玩家选"，不是"算效果"。
手动玩时走的是 `DialogSingleModeScenarioFestaFusionAdvice` 那条 UI 链。

> 两条链最终都要调用同一个提交接口 → 效果计算在那之后。

---

## 5. 累计排除表

| 类 | 是否含效果/概率计算 | 结论 |
|:--|:-:|:--|
| `TrainingCommandEntity` | ✗ | 只是数据容器（3 字段） |
| `ObscuredSingleModeFestaCommandInfo` | ✗ | 网络传输结构 |
| `AdviceEntity` | ✗ | 两个 VO 的容器 |
| `AdviceInfoEntity` | ✗ | 一个 List |
| `AttachAdviceEntity` | ✗ | List + Dict |
| `AdviceVO` / `ActTrainingEffectVO` | ✗ | 纯显示 |
| `SingleModeScenarioFestaDefine` | ✗ | UI 坐标/颜色/文本 id |
| `CanExecActTrainingService` | ✗ | 只返回激活演目 |
| `AdviceInfoRepository` | ✗ | 只做 Get |
| `AcquiredAdviceService` | ✗ | 只做 GetList |
| `SingleModeScenarioFestaAdviceFusionEffectTextService` | ✗ | 只生成文本 |
| `SingleModeScenarioFestaAdviceLevelEffectTextService` | ✗ | 只生成文本 |

**12 个类全部排除。** 客户端「按类名能想到的地方」已经没有候选。

---

## 6. 新的判断：计算可能在 **Work 数据层**

`ObscuredSingleModeFestaCommandInfo` 的字段由**序列化过程**填充（`ParamsIncDecInfoArray` 也是它）。
真正算出 `ParamsIncDec` 的，应该是 `Work*` 系列：

线索类（批次 03 目标）：

| 类 | 为什么怀疑 |
|:--|:--|
| `WorkSingleModeCharaData` | 有 `get_TrainingEventType` |
| `WorkIdleSingleModeData` | 同上 |
| `SingleModeLogGroupBase` | `<SetEventIcon>g__SetTrainingEventIcon` |
| `StoryViewController` | `SetTrainingEventId` |

`Work*` 是存档数据结构，**训练结果算完直接写进去**。概率判定如果发生在客户端，大概率就在写 Work 之前那一步。

---

## 7. 批次 03 计划

| # | 目标 | 目的 |
|:-:|:--|:--|
| 1 | `WorkSingleModeCharaData` 字段 + `get_TrainingEventType` | 训练事件类型从哪来 |
| 2 | `WorkIdleSingleModeData` 同名字段 | 交叉验证 |
| 3 | 搜 `keyword=ActGauge` 全部方法 | 进度条写入点 ≈ 训练结算点 |
| 4 | 搜 `keyword=Random` 在 ScenarioFesta 相关类 | 找随机数调用 |
| 5 | `SingleModeFestaAPI` 全部方法 | 确认是否服务端下发 |

---

## 8. 本批结论

| 问题 | 答案 |
|:--|:--|
| 布尔由谁写入 | setter 直接写 ObscuredBool，**加密存储** |
| 加密布局 | value @ +0x48，key @ +0x50，真值 = XOR |
| ObscuredInt 步长 | **20 字节**（不是 8） |
| 転成选择在哪 | `AdviceAutoSettingService.AutoSelectFusionSet`（自动育成用） |
| 效果计算在哪 | **不在这 12 个类里**，指向 `Work*` 层 |
