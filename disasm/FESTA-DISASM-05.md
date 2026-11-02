# 剧本15 反汇编批次 05

> 2026-11-02。本批回答用户的问题：**追加の稽古算不算「友人的非外出触发事件」？**
> 同时拿到了完整的 182 个剧本15 类清单，做了分层。

---

## 1. 直接回答

**代码层面：没有「友人事件」这个分类，所以无从判定。**

依据：全库搜 `FriendEvent` → **0 命中**。

剧本15 里带 `Friend` 的只有 3 个成员，**全部属于外出**：

| 类 | 成员 |
|:--|:--|
| `ObscuredSingleModeFestaActInfo` | `FriendOutingGainActLevel`（+136） |
| `DialogSingleModeScenarioFestaOutingPartnerSelectItemViewModel` | `get_ActTypeLevelUpByFriendOutingDict` |
| 同上 | `get_AnyActTypeLevelUpByFriendOuting` |

**追加の稽古在客户端没有任何「友人」标记。** 它表现出来就是一次 `GainActGauge` 参数变化：

```
ChangeParameterInfo
  Type      = (GainActGauge 相关枚举)
  Value     = 1
  ValueList = [...]
```

与普通进度增加**结构完全同构**，无法在客户端区分"这也是友人做的"。

> 如果游戏设计上把它归为「友人的非外出触发事件」，这个归类**只存在于策划表或服务端**，
> **客户端没有对应的代码落点**——这也解释了为什么搜 `Friend` 只有外出的那三处。

---

## 2. 剧本15 全类清单：182 个

按职责分层：

### 逻辑层（约 41 个）

| 层 | 类 |
|:--|:--|
| **Entity** | `ActEntity`、`ActListEntity`、`ActMemberInfoEntity`、`AdviceEntity`、`AdviceInfoEntity`、`AttachAdviceEntity`、`AdviceAutoSettingEntity`、`CheckPointEntity`、`CheckPointListEntity`、`CommandEntity`、`PeriodEntity`、`TrainingCommandEntity` |
| **Repository** | `ActListRepository`、`ActMemberInfoRepository`、`AdviceInfoRepository`、`AdviceAutoSettingRepository`、`CheckPointListRepository`、`CommandRepository`、`PeriodRepository`、`TrainingCommandRepository` |
| **Service** | `AcquiredAdviceService`、`AdviceAutoSettingService`、`CanExecActTrainingService`、`CheckPointProgressService`、**`GainedActGaugeService`**、**`LevelUpActService`**、`SingleModeScenarioFestaAttachAdviceAutoSelectService`、两个 `...EffectTextService` |
| **VO** | `AdviceVO`、`ActTrainingEffectVO`、`ActTrainingEffectVOList`、`CheckPointEffectVO`、`UrafActTrainingEffectVO` |
| **参数/消息** | `SingleModeScenarioFestaAcquiredAdviceParameterInfo`、`...ActivateActTrainingParameterInfo`、`...CheckPointProgressUpParameterInfo`、**`...GainActGaugeParameterInfo`**、`...GainActLevelParameterInfo`、`...JoinMemberParameterInfo`（每个都有 `*A2U` 变体） |
| **其他** | `SingleModeScenarioFestaDefine`、`SingleModeScenarioFestaTrainingDefineExtensions`、`SingleModeScenarioFestaMainTrainingMassageService`、`FestaEndLiveHelper`、`TrainingFooterState(Utility)` |

### UI 层（约 141 个）

全部是 `Dialog...` / `Parts...` / `...View` / `...Controller` / `...ViewModel`，不做数值计算。

---

## 3. 两个结算入口（各只有一个方法）

| 类 | 方法 | 签名 |
|:--|:--|:--|
| `GainedActGaugeService` | **`GetActGaugeGainInfoDict`** | `static`，返回 `Dictionary` |
| `LevelUpActService` | **`GetLevelUpActList`** | 返回升级演目列表 |

**这两个类各有且仅有 1 个业务方法，其余全是 Object 样板方法。**

→ 整个「涨进度 / 升级」的判定，就压缩在这两个 static 方法里。
**它们是最值得反汇编的两个目标。**

---

## 4. `ActMemberInfoEntity` —— 演目成员（4 字段）

| 偏移 | 字段 | 类型 |
|--:|:--|:--|
| +16 | TargetId | int |
| +20 | CharaId | int |
| +24 | TrainingCommandId | TrainingCommandId |
| +28 | EvaluationValue | int |

这是「哪个角色挂在哪个演目训练上」。`CharaId` 可以用来识别**友人角色**——
如果追加の稽古真的由友人触发，`CharaId` 会是判定入口。

---

## 5. `...GainActGaugeParameterInfo` 继承自 `ChangeParameterInfo`

基类 16 个字段：

| 偏移 | 字段 |
|--:|:--|
| +16 | Type : ParameterType |
| +20 | Value : int |
| +24 | ValueList : List\<int\> |
| +32 | ParamStr : string |
| +40 | ParamValue : int |
| +44 | ParamValue2 : int |
| +48 | ParamValue3 : int |
| +52 | Place : GroupPlace |
| +56 | OnParamChangeStart : Action |
| +64 | ForcedPlayTime : float |
| +68 | ForcedFlashNone : bool |

子类追加：

| 偏移 | 字段 |
|--:|:--|
| +72 | MessageText : string |
| +80 | IsNeedPlayOutScenarioParts : bool |
| +88 | ActListEntitySet : (ActListEntity, ActListEntity) |
| +104 | InfoIndex : (int, int) |
| +112 | ActGaugeAnimationState |

**`ActListEntitySet` 是一对** —— 正好是「变化前 / 变化后」两份进度快照。
这就是 UI 上进度条动画的输入。

---

## 6. 下一步（批次 06）

按价值排序：

| # | 目标 | 找什么 |
|:-:|:--|:--|
| 1 | `GainedActGaugeService.GetActGaugeGainInfoDict` 反汇编 | **遍历 + 取最小 + 赋 1** 的循环 |
| 2 | `LevelUpActService.GetLevelUpActList` 反汇编 | 升级条件（`>= NeedActGauge`） |
| 3 | `ActEntity` 全部 getter | `CurrentActGauge` 的比较逻辑 |
| 4 | 搜 `keyword=CharaId` 过滤剧本15 | 友人的 `CharaId` 是否参与判定 |

**如果第 1 项里出现「遍历 ActList、比较 gauge、取最小、+1」的循环，就直接锁死了 +1 格的代码位置。**
