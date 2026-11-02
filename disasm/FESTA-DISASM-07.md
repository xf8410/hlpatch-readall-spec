# 剧本15 反汇编批次 07 —— 运行时类字段表 + 字典定位 + 加密体系

> 2026-11-02 10:44。用户要求：**先把结论和思考链落仓库防丢**，再继续。

---

## 1. 运行时类字段表（**[码]** `/il2cpp/dump`，实测非推断）

### `ActEntity`（Gallop.SingleMode.ScenarioFesta）— 9 字段，**全是普通 int32**

| 偏移 | 字段 | 类型 |
|--:|:--|:--|
| +16 | ActType | ActType 枚举 |
| +20 | AttachedAdviceId | int32 |
| +24 | ActMemberCharaIdList | IReadOnlyList<int> |
| +32 | MaxActTrainingLevel | int32 |
| +36 | ActTrainingState | ActTrainingState 枚举 |
| +40 | ActTrainingEffectVo | ActTrainingEffectVO |
| +48 | UrafActTrainingEffectVo | UrafActTrainingEffectVO |
| +56 | CurrentActGauge | int32 |
| +60 | LevelUpByFriendOuting | int32 |

**更正记录**：此前记录的「FriendOutingGainActLevel = +0x88（20 字节 Obscured）」属于
**master 行类 `ObscuredSingleModeFestaActInfo`**，不是运行时 `ActEntity`。
两套表不能混用：master 行走 Obscured（key^value），运行时走裸 int32。

### `TrainingCommandEntity` — 3 字段

| 偏移 | 字段 | 类型 |
|--:|:--|:--|
| +16 | CommandId | int32 |
| **+24** | **`_gainActGaugeAmountDict`** | **Dictionary** |
| +32 | IsUpperTrainingEventOccurred | bool |

`ActTrainingEffectVO` 只有 Level +16 / EffectTextList +24 —— **纯展示文案，不含数值**。

---

## 2. 字典的填充点与取用点（**[码]**）

| 角色 | 方法 | 说明 |
|:--|:--|:--|
| 填充 | `TrainingCommandRepository.CreateCommandEntity` | 构造时预生成字典 |
| 取用 | `TrainingCommandEntity.GetGainActGaugeAmount` @ **0x7547a565e0** | 只做字典查找 |
| 列表 | `TrainingCommandRepository.Get / GetList / GetByBackup` | — |

**关键事实**：`GetGainActGaugeAmount` 整段 2048 字节里
**没有 202 / 203 的立即数**（按 `movz` 编码扫过 `0x52801940`、`0x52801960`）。
⇒ 进度增量不在此处按 effect_type 分支计算。

**未完成**：字典内容一直没读到，见 §4。

---

## 3. 失败记录（不隐去）

| 尝试 | 结果 |
|:--|:--|
| `/il2cpp/call?class=TrainingCommandRepository&method=GetList` | `singleton_null`（该端点只支持 singleton 类） |
| `/il2cpp/read_mem?addr=0x76bf22d6e0`（旧会话地址） | `read_failed`，`bytes_read: 0` —— 对象已被回收 |
| `/il2cpp/disassemble` 大函数 | 只回 hex_dump 无助记符；`found_constants` 为空 |

**教训**：内存地址必须现取现用，跨操作复用必失败。

---

## 4. 当前有效地址链（2026-11-02 10:44，第 10 回合，育成主界面）

```
/scenario → dataset_obj = 0x76a35ff5a0  (WorkSingleModeScenarioFestaDataSet)
  +0x10 FestaInfo                 = 0x76a353c9c0
  +0x18 CommandInfoArray          = 0x76aad13910   (len 5)
  +0x20 EvaluationInfoArray       = 0x76a3601690
  +0x28 AdviceInfoArray           = 0x76aad13870
  +0x30 ActInfoArray              = 0x76aad13820   (len 5)
  +0x38 CheckPointResultArray     = 0x76a35f5480
  +0x40 CommandGainActGaugeArray  = 0x76aad137d0   (len 4)
  +0x48 NotUpParameterInfo        = 0x76aad1cd40
  +0x50 AutoSettingInfo           = 0x76aad1cd00
```

**下一步**：读 `0x76aad137d0`（CommandGainActGaugeArray，len 4）→ 与
`_gainActGaugeAmountDict` 的键值对做比对。**「两者是否同源」尚未验证，不作结论。**

---

## 5. 加密体系（**[码]** `/il2cpp/classes` 搜 `Obscured`）

命中 **315 个类**。全部位于 `CodeStage.AntiCheat` 体系，命名模式固定：

| 模式 | 例 |
|:--|:--|
| `Obscured<模型名>` | `ObscuredSingleModeFestaActInfo`、`ObscuredUserChara`、`ObscuredUserItem` |
| `Obscured<模型名>Extensions` | 配套的读写扩展 |

覆盖范围（按系统）：

- 用户资产：UserChara / UserCard / UserItem / UserSupportCard / UserMusic / UserCloth / PieceData
- 育成：SingleMode{Breeders, Onsen, Ramen, Festa}* 全套 DataSet / Info / Command
- 继承：Succession* / GenerateSuccession* / FactorInfo / ExtraFactorInfo
- 日程本：ScheduleBook*（7 类）
- 其他：HonorData / LovePointInfo / RaceHorseData / RewardSummaryInfo / SkillData …

### 5.1 加密强度分级（**[码]** 实测样本）

| 级别 | 字段 | 存储形态 | 解密 |
|:--|:--|:--|:--|
| 强 | 模型类字段（Obscured*） | 20 字节/字段 | `value = 第二int ^ 第一int(key)`，key 每局变化 |
| 弱 | 运行时实体（ActEntity 等） | 裸 int32 | 无加密 |

**这是两套并存**：master 行 / 存档 DTO 走 Obscured，运行时工作对象走明文。

### 5.2 布尔加密

`IsUpperTrainingEventOccurred`（TrainingCommandEntity +32）在 dump 里报告为 **bool**，
紧跟 +24 字典之后，未标注 Obscured。**该字段是否另有加密，[无证]。**

`ObscuredBool` 这一基本类型本次未在命中的 315 个名字里直接看到——
列表返回有截断，**不能据此断言它不存在**。

---

## 6. 本批结论表

| 问题 | 结论 | 证据 |
|:--|:--|:--|
| ActEntity 的字段与偏移 | 9 字段、+16…+60、明文 int32 | **[码]** dump |
| 进度增量存哪 | `_gainActGaugeAmountDict` @ +24 | **[码]** dump |
| 谁填充 | `TrainingCommandRepository.CreateCommandEntity` | **[码]** 方法表 |
| 进度增量是否按 effect_type 分支算 | **否**，GetGainActGaugeAmount 无 202/203 立即数 | **[码]** 反汇编 |
| 字典的实际键值 | 未读到 | **[无证]** |
| 加密有几套 | 至少两套（Obscured 模型层 / 明文运行时层） | **[码]** dump 对比 |
| Obscured 覆盖多少类 | 315 | **[码]** 类搜索 |
