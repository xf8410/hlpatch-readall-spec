# 剧本15 反汇编批次 01

> 开始 2026-11-02。分批落盘，每批记录：dump 了什么、取到什么地址、读出什么结论。
> 反汇编端点只回 **ARM64 原始字节**（hex），不回助记符；本文件记录地址与结论，原始字节可用端点复现。

---

## 0. 复现方式

```bash
# 单方法反汇编（返回 2KB hex）
GET /il2cpp/disassemble?class=<类名>&method=<方法名>

# 类字段
GET /il2cpp/dump?name=<类名>

# 方法检索（支持子串）
GET /il2cpp/search_methods?keyword=<关键词>
```

方法地址在返回的 `method_addr` 里，可用来判断哪些方法**共用同一段代码**（getter 常被内联复用）。

---

## 1. 本批 dump 的类

| 类 | 命名空间 | 状态 |
|:--|:--|:--|
| `TrainingCommandEntity` | Gallop.SingleMode.ScenarioFesta | ✅ 字段+方法+2 方法字节 |
| `ObscuredSingleModeFestaCommandInfo` | Gallop | ✅ 字段 |
| `AdviceEntity` | Gallop.SingleMode.ScenarioFesta | ✅ 字段 |
| `AdviceInfoEntity` | Gallop.SingleMode.ScenarioFesta | ✅ 字段+方法 |
| `AttachAdviceEntity` | Gallop.SingleMode.ScenarioFesta | ✅ 字段 |
| `AdviceVO` | Gallop.SingleMode.ScenarioFesta | ✅ 字段 |
| `ActTrainingEffectVO` | Gallop.SingleMode.ScenarioFesta | ✅ 字段 |
| `SingleModeScenarioFestaDefine` | Gallop.SingleMode.ScenarioFesta | ✅ 字段 |

---

## 2. 核心发现

### 2.1 `AdviceEntity` 同时持有两个 VO —— 直接印证「旧 buff 不删除」

```
AdviceEntity
  +16  MainAdviceVO    : AdviceVO     ← 原有 buff
  +24  FusionAdviceVO  : AdviceVO     ← 転成后的 buff
```

**一个 advice 实体里并存两份**。这与用户说明的机制完全一致：

- 第 1 年 12 月下：5 选 2 転成
- 第 2 年 12 月下：剩 3 个転成
- **旧的不删除，两份同时生效**

→ 同型效果两条并存，运行时合并方式只能是**相加**。这是「加算」在对象结构层的证据。

### 2.2 追加の稽古 的代码名是 `UpperTrainingEvent`

| 类 | 方法 |
|:--|:--|
| `ObscuredSingleModeFestaCommandInfo` | `get/set_IsUpperTrainingEventOccurred` |
| `TrainingCommandEntity` | `get_IsUpperTrainingEventOccurred` |

`search_methods?keyword=TrainingEvent` 全库只命中 9 个方法，其中与剧本15 相关的就这 3 个。

### 2.3 关键：结构里**没有任何 Rate 字段**

`ObscuredSingleModeFestaCommandInfo` 全部 5 个字段：

| 偏移 | 字段 | 类型 |
|--:|:--|:--|
| +16 | CommandType | ObscuredInt |
| +36 | CommandId | ObscuredInt |
| +56 | ParamsIncDecInfoArray | `SingleModeParamsIncDecInfo[]` |
| +64 | PredictGainActGaugeArray | `ObscuredSingleModeFestaPredictGainActGauge[]` |
| +72 | **IsUpperTrainingEventOccurred** | **ObscuredBool** |

**只有布尔结果，没有概率浮点/整数。** 说明「会不会触发」这个中间量不落字段——
要么服务端算完下发，要么客户端算完即弃。

这解释了为什么在 MDB 和类名里都搜不到「概率算法」。

### 2.4 `TrainingCommandEntity`

| 偏移 | 字段 | 类型 |
|--:|:--|:--|
| +16 | CommandId | TrainingCommandId |
| +24 | _gainActGaugeAmountDict | `IReadOnlyDictionary<ActType, int>` |
| +32 | IsUpperTrainingEventOccurred | bool |

方法 16 个，业务方法 3 个：

| 方法 | 地址 | 作用 |
|:--|:--|:--|
| `get_IsUpperTrainingEventOccurred` | `0x754a27c2fc` | 读 `+0x20` 布尔 |
| `GetGainActGaugeAmount(ActType)` | `0x754a2745e0` | 查字典取稽古进度增量 |
| `get_CommandId` | — | 读 `+0x10` |

### 2.5 稽古设施升级链路（与用户说明对齐）

`GetGainActGaugeAmount` = 每次训练给某个演目（ActType）涨多少**进度条**。

`SingleModeScenarioFestaDefine` 里的常量组印证了这条 UI 链路：

```
ACT_GAUGE_GAIN_BADGE_POSITION_HOLIDAY        ← 休息时进度条增量徽章位置
ACT_GAUGE_GAIN_BADGE_POSITION_OUTING
ACT_GAUGE_GAIN_BADGE_POSITION_OUTING_SUMMER
ACT_GAUGE_GAIN_BADGE_POSITION_HOSPITAL
ACT_GAUGE_GAIN_BADGE_POSITION_RACE_NORMAL
ACT_GAUGE_GAIN_BADGE_POSITION_RACE_RESERVE
ACT_GAUGE_GAIN_BADGE_POSITION_RACE_ROUTE
```

机制：**训练/休息/外出/比赛都会涨对应 ActType 的进度条 → 5 条攒满 → 触发稽古设施训练 → 清空 → 该设施 +1 级**。

→ 所以「buff 等级随稽古涨」= **设施等级**随进度条涨，不是教え自身有独立等级。

### 2.6 `AdviceVO` 字段

| 偏移 | 字段 |
|--:|:--|
| +16 | AdviceId : int |
| +20 | AdviceFusionType : FusionType（枚举） |
| +24 | AdviceName : string |
| +32 | AdviceEffectCategoryText : string |
| +40 | FusionAdviceEffectText : string |
| +48 | AddDegreeType : DegreeType |

全是**展示用**（名称、效果文本），无数值计算字段。

`ActTrainingEffectVO` 同样只有 `Level` + `EffectTextList` —— **纯显示对象**。

---

## 3. 结论

| 问题 | 当前答案 | 强度 |
|:--|:--|:--|
| 旧 buff 是否与転成 buff 并存 | **是**，`MainAdviceVO` + `FusionAdviceVO` 双字段 | 硬证据 |
| 三条以上同型效果如何合并 | **相加**（否则无法并存） | 强推断 |
| 追加の稽古 代码名 | `UpperTrainingEvent` | 硬证据 |
| 概率中间值是否可读 | **不落字段**（只有 ObscuredBool 结果） | 硬证据 |
| 概率合成算法 | **仍未定位** | 未解 |

---

## 4. 下一批计划

| # | 目标 | 目的 |
|:-:|:--|:--|
| 1 | `CanExecActTrainingService` 全部方法 | 稽古执行条件，可能含事件判定 |
| 2 | `SingleModeScenarioFestaMainTrainingMassageService.GetTrainingResultMassage` | 训练结果组装，布尔从哪来 |
| 3 | `ObscuredSingleModeFestaCommandInfo` 的 setter | 布尔由谁写入 |
| 4 | `AdviceAutoSettingService` / `AdviceInfoRepository` | buff 状态来源 |
| 5 | `ActTrainingEffectVO` 生产方（谁 new 出来的） | 效果聚合点 |
