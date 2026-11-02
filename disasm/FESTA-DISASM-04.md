# 剧本15 反汇编批次 04 —— 友人机制专项

> 2026-11-02。用户指正：追加の稽古 是**友人（脚本联动角色）的特殊机制**，由剧本 buff 触发，
> 效果是**给当前进度条最低的设施 +1 格**；全部满格时无效。
> **不是溢出事件**。另：**友人外出也给 1 格，但那是另一套机制，不可混用。**

---

## 1. 先撤回批次 03 的错误

| 批次 03 的说法 | 状态 |
|:--|:--|
| 「追加の稽古 = 溢出事件」 | ❌ **撤回** |
| 「溢出 → UpperTrainingEvent」因果链 | ❌ **撤回** |
| `IsOverflowActGaugeEventGain` 属于追加の稽古 | ❌ **撤回** |

`Overflow` 关键词全库只有 9 个方法，剧本15 占 4 个：

| 类 | 方法 |
|:--|:--|
| `ObscuredSingleModeFestaNotUpParameterInfo` | `get/set_IsOverflowActGaugeEventGain` |
| `ObscuredSingleModeFestaNotUpParameterInfo` | `get/set_IsOverflowActLevelEventGain` |

它们是**进度条/等级溢出时的「未加参数」标记**（NotUpParameter = 没涨到的参数），
与追加の稽古无关。

---

## 2. 剧本15 的友人相关成员（全库检索结果）

搜 `keyword=Friend`（200 命中）→ 过滤到剧本15 仅 **3 处**：

| 类 | 成员 | 含义 |
|:--|:--|:--|
| `ObscuredSingleModeFestaActInfo` | `get/set_FriendOutingGainActLevel`（+136） | **友人外出给该演目加的量** |
| `DialogSingleModeScenarioFestaOutingPartnerSelectItemViewModel` | `get_ActTypeLevelUpByFriendOutingDict` | 外出选择界面：哪些演目会因此升级 |
| 同上 | `get_AnyActTypeLevelUpByFriendOuting` | 是否有任一演目会升级 |

相关常量（`SingleModeScenarioFestaDefine`）：

```
SCENARIO_LINK_CHARA_ID_ARRAY            (+128)  联动角色 id 数组
ENDING_LIVE_SCENARIO_LINK_CHARA_ID_ARRAY(+136)  Ending Live 联动角色
```

搜 `keyword=LinkChara`（30 命中）→ **剧本15 一个都没有**，全在 Pioneer / Ramen / Breeders。
说明剧本15 的联动角色**不吃通用的 ScenarioLinkChara 通路**，只走自己的 `SCENARIO_LINK_CHARA_ID_ARRAY`。

---

## 3. 关键结论：友人机制在剧本15 只有**一个数值落点**

```
ObscuredSingleModeFestaActInfo
  +136  FriendOutingGainActLevel : ObscuredInt
```

这个字段是「**友人给某演目加进度/等级**」的通用载体。

而 `DialogSingleModeScenarioFestaOutingPartnerSelectItemViewModel` 的方法说明：
外出选择时，界面会预先算出 `ActType → 是否升级` 的字典给玩家看。

→ **友人外出 = 走 `FriendOutingGainActLevel`，与追加の稽古分开**（符合用户说明）。

---

## 4. 但目前**没有找到追加の稽古的专属字段**

搜遍以下关键词，剧本15 均无专属服务类：

| 关键词 | 命中 | 剧本15 相关 |
|:--|--:|:--|
| `TrainingEvent` | 9 | 3（全是 getter/setter） |
| `Overflow` | 9 | 4（NotUpParameter） |
| `Friend` | 200 | 3（外出相关） |
| `LinkChara` | 30 | **0** |
| `ActGauge` | 71 | 30+（全是 UI/数据容器） |
| `Random` | 200 | **0** |

**追加の稽古没有自己的类、没有自己的字段、没有自己的方法。**

它只能有两种实现方式：

| 方式 | 说明 |
|:--|:--|
| **A. 复用 ActGauge 通路** | 直接把 +1 加到某 ActType 的 `ActGauge` 上，不单独存标记 |
| **B. 服务端合成** | 服务端算完，直接把 +1 后的 `ActGauge` 下发 |

两种方式在客户端**看起来一模一样**——都只有 `ActGauge` 数值变化。

> 这也是为什么 `IsUpperTrainingEventOccurred` 是个**布尔、无 Rate**：
> 客户端只被告知"发生了/没发生"，不参与计算。

---

## 5. 下一步：从「+1 格」这个动作反推

既然效果是「给最低的设施 +1 格」，那么：
**+1 这个动作一定有一段代码在找"最低的那条"**。

目标类：

| 类 | 为什么 |
|:--|:--|
| `ActEntity`（有 `CurrentActGauge` / `ActGaugeLimit`） | 单条进度条实体，最可能含比较逻辑 |
| `SingleModeScenarioFestaGainActGaugeParameterInfo` | 名为「GainActGauge 参数信息」，像是 +1 的载体 |
| `GainedActGaugeService.GetActGaugeGainInfoDict`（static） | 结算入口 |
| `WorkSingleModeScenarioFestaDataSet.CommandGainActGaugeArray` | 存档侧的增量数组 |

**找法**：反汇编 `ActEntity` 全部 getter + `GetActGaugeGainInfoDict`，
看有没有「遍历 5 条、取最小、+1」的循环特征（比较 + 条件跳转 + 数组下标）。

---

## 6. 本批结论

| 问题 | 答案 |
|:--|:--|
| 追加の稽古 = 溢出事件？ | **否**（已撤回） |
| 友人外出的落点 | `FriendOutingGainActLevel` @ +136 |
| 追加の稽古的落点 | **不存在专属字段**，疑似复用 `ActGauge` |
| 是否有客户端概率计算 | 仍无证据；`Random` 在剧本15 命中 0 |
| 下一步 | 反汇编 `ActEntity` + `GetActGaugeGainInfoDict`，找"取最小 +1"的循环 |
