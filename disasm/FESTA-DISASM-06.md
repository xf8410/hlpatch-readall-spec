# 剧本15 反汇编批次 06 —— 友人 = 吾妻 的身份落地

> 2026-11-02。用户给出候选 id **30323**。本批把它查穿了。

---

## 1. 30323 是什么：**抽卡支援卡，不是 chara id**

`chara_data` 的 id 范围是 **1001–9052，共 174 行**，`id=30323` 不存在（`row_count 0`）。

查 `support_card_data` 命中：

| 字段 | 值 |
|:--|:--|
| id | **30323** |
| **chara_id** | **9052** |
| rarity | **3（SSR）** |
| support_card_type | 2 |
| **outing_max** | **5** |
| start_date | **1792983600 = 2026-10-26 12:00 JST** |

**start_date 与剧本15 开档时刻完全相同**（`single_mode_scenario` id 15 也是 1792983600）。

---

## 2. chara 9052 就是吾妻 —— 文本表直读

`chara_data` where id=9052：`sex 1`、`chara_category 1`、`love_rank_limit 10`。

`text_data` 里 `index=9052` 的 16 条，关键几条：

| category | text |
|:-:|:--|
| **6** | **吾妻** |
| 7 | 三石琴乃（CV） |
| 163 | 駿大祭の巫女、吾妻と申します。／ウマ娘と人の絆を守り、尊び… |
| 170 | 吾妻 |
| 173 | 巫女 |
| 182 | アヅマ |
| 372 | Azuma |

`character_system_text` 里 `character_id=9052` 出现 **`snd_voi_festa_905200`** 语音组，
start_date = 1792983600 → **剧本15（Festa）专属语音**。

**结论：chara 9052 = 吾妻（駿大祭の巫女），其 SSR 支援卡 = 30323。**

---

## 3. 「她外出也给 1 格」—— 数据表原文

`single_mode_15_outing_effect` 共 **10 行**：

| id | support_card_id | story_step | add_level |
|--:|--:|:-:|:-:|
| 1–5 | **30323（吾妻）** | 1,2,3,4,5 | **1** |
| 6–10 | 10149 | 1,2,3,4,5 | **1** |

**这张表只按 `support_card_id` 索引**（索引名 `_SupportCardId`），
说明它是「**外出时**该支援卡带来的演目等级增量」。

→ 每次外出 `add_level = 1`，共 5 个 story_step 档。
**这就是用户说的「她外出也给 1 格」，而且它是独立的表，与追加の稽古无关。**

---

## 4. 吾妻不是演目担当 —— 与「剧本 NPC」彻底分开

`single_mode_15_act_member` 只有 5 行：

| act_type | chara_id |
|:-:|--:|
| 1 | 1009 |
| 2 | 1096 |
| 3 | 1024 |
| 4 | 1029 |
| 5 | 1128 |

**9052 不在其中。** 她是祭典巫女（全局角色），不是 5 个演目的担当。
所以「友人（吾妻）」与「演目上的剧本 NPC」在数据里本就不是同一类东西。

---

## 5. ActLevel 相关 API 清单（34 个方法）

逻辑侧：

| 类 | 成员 |
|:--|:--|
| `ActEntity` | `get_IsActLevelMax` |
| `ActListEntity` | `get_MaxActLevel` |
| `CheckPointEntity` | `get_NeedActLevel` |
| `ObscuredSingleModeFestaInfo` | `get_MaxActLevel` |
| `ObscuredSingleModeFestaActInfo` | `get/set_FriendOutingGainActLevel` |

消息/UI 侧：`GetActLevelUpMessageList`、`GetNotUpActLevelUpMessageList`、`PlayActLevelUp`、
`PlayActLevelExec`、`IsGainActLevelUp`、`UpdateActLevelNext`、`ResetActLevelNext`、
`SetParameterInfoByScenarioFestaActLevel` —— 全是展示。

**没有任何一处是「取最低那条 +1」的计算。**

---

## 6. 本批结论

| 问题 | 答案 | 依据 |
|:--|:--|:--|
| 30323 是什么 | 支援卡 id，chara_id=9052 | `support_card_data` |
| 9052 是谁 | **吾妻**（駿大祭の巫女） | `text_data` cat 6/163/170/182 |
| 她外出给 1 格 | `15_outing_effect` add_level=1 × 5 步 | 数据表 |
| 吾妻是演目担当吗 | **不是**（act_member 只有 1009/1096/1024/1029/1128） | 数据表 |
| 「取最低 +1」的代码 | **仍未找到** | 34 个 ActLevel 方法里无计算 |

---

## 7. 批次 07 计划

有了确定身份 **9052 / 30323**，可以做**常量扫描**：

| # | 做法 | 目的 |
|:-:|:--|:--|
| 1 | 反汇编 34 个 ActLevel 方法，扫立即数 `0x235C`(9052) / `0x7673`(30323) | 找硬编码引用点 |
| 2 | 扫 `single_mode_15_advice` / `advice_effect` 全表 | 确认 701 的 type 8 十档与条件列 |
| 3 | 读 `15_outing_effect` 在代码里的消费点（`GetSelectQuery...SupportCardId`） | 确认外出通路与追加稽古是否共用 add_level |
| 4 | 反汇编 `Model.GetActLevel(int)` | 唯一带参数的等级查询，可能是复用点 |
