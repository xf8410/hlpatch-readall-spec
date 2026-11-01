# hlpatch 插件「全格式可读」改造规格书 v1

> 目标一句话：**让 `libhachimi_ura.so` 面对任何输入——内存、文件、SQLite、msgpack、压缩包、UTF-8/UTF-16/二进制——都能安全读出来，并且输出永远是合法 JSON，不崩、不乱码、不截断。**

本文只写「怎么改」，不写实现代码。按章节顺序执行即可。

---

## 0. 先明确三件事

### 0.1 端口 ≠ 端点

| 概念 | 现状 | 本次是否改 |
|:--|:--|:--|
| **监听端口** | `127.0.0.1:18765`，全局唯一 | ❌ **不新增、不改** |
| **端点（path）** | 已有 `/api/*`、`/il2cpp/*`、`/debug/*` | ✅ **本次只加端点** |

**不要开第二个 HTTP 监听端口。** 安卓上再起一个端口 = 再起一个线程 + 再占一个 socket，收益为零，风险为正。所有新能力都挂在 18765 的新 path 下。

### 0.2 验收标准（全部满足才算完成）

| # | 标准 | 判定方法 |
|:-:|:--|:--|
| V1 | 任意端点，非法参数返回 JSON 而非空响应/断流 | 故意传错参数，看是否 200 + `{"error":...}` |
| V2 | 任意二进制输入，输出是合法 JSON（可被 `jq` 解析） | `curl ... \| jq .` 不报错 |
| V3 | 日文/中文不乱码 | 查询含日文的路径/SQL，输出与游戏内一致 |
| V4 | 任一页未映射不导致整体失败 | 读跨页地址，返回 `gaps` 且其余数据完整 |
| V5 | 一段崩不影响其他段 | `/debug/all` 五段全部有返回 |
| V6 | 连续 100 次请求不崩、不漏内存 | 循环压测，进程存活 |

### 0.3 三条铁律（违反必崩）

```
① 指针绝不跨请求缓存 —— 对象图每帧重建
② 全量 classes / 全量 dump 禁止 —— 曾把游戏打崩（31,420 类）
③ 装机必须在游戏静止时 —— 运行中打补丁 = 无声 SIGSEGV
```

---

## 1. 总体流程：先干什么，后干什么

```
P0 冻结基线
   ↓
P1 编码基座（输出永不乱码的前提）
   ↓
P2 安全读取层（不崩的前提）
   ↓
P3 通用解码器（格式覆盖）
   ↓
P4 端点注册（对外暴露）
   ↓
P5 输出契约（统一格式）
   ↓
P6 构建 / 装机 / 验收
```

**为什么是这个顺序**：P1、P2 是后面所有东西的地基。先写端点再补编码，等于每个端点都要改两遍。

| 阶段 | 内容 | 依赖 | 预估 |
|:-:|:--|:--|:--|
| **P0** | 记录基线（commit、so 大小、sha256） | — | 5 分钟 |
| **P1** | 编码：UTF-8/UTF-16LE/BE/Latin-1 统一入口 | — | 半天 |
| **P2** | 安全读：maps 区间表 + 分页读 + jmp_buf 隔离 | — | 1 天 |
| **P3** | 格式探测 + 解码器（SQLite/msgpack/zlib/gzip/JSON） | P1 P2 | 2 天 |
| **P4** | 端点 + 路由 + 白名单 + health + not_found 四处登记 | P3 | 半天 |
| **P5** | 统一 JSON envelope + 转义 | P1 | 半天 |
| **P6** | 构建、装机、按 §9 验收 | 全部 | 1 天 |

---

## 2. 逐阶段改动清单

### P0 · 冻结基线

改任何东西之前，先记下：

```
分支/commit:  ______________________
lib.rs 行数:  ______________________
so 大小:      ______________________ 字节
so sha256:    ______________________
```

**理由**：改崩了能退回。3.28.x 线只在 `.so` 层面发布过，源码基线不完整，一旦丢掉基线就只能靠反汇编重来。

---

### P1 · 编码基座

#### 1.1 新增文件 `hachimi_ura_plugin/src/encoding.rs`

必须提供四个函数（名字可改，语义不能改）：

| 函数 | 输入 | 输出 | 用途 |
|:--|:--|:--|:--|
| `decode_query(s: &str) -> String` | 原始 query 片段 | UTF-8 String | 解 `%XX` 与 `+` |
| `utf16le_to_string(ptr, len) -> String` | 裸指针 + 字符数 | String | 读 IL2CPP 字符串 |
| `bytes_to_text(b: &[u8]) -> (String, &'static str)` | 任意字节 | (文本, 编码名) | 自动判编码 |
| `escape_json(s: &str) -> String` | 任意 String | 可嵌入 JSON 的串 | 输出转义 |

#### 1.2 修 `url_decode`（**这行代码有前科**）

现有实现是 v3.22.89 遗留物，逐字节 `as char`（Latin-1），导致日文查询参数全乱码。

**改法**：

```
错误：%E3%81%82 → 0xE3 as char → "ã"  (Latin-1)
正确：先收集 Vec<u8> = [0xE3,0x81,0x82] → String::from_utf8_lossy → "あ"
```

顺序必须是：**`%XX` → 字节 → `+` 当空格 → UTF-8 解码**。顺序错就会吃掉合法的 `+`（base64 会坏）。

#### 1.3 `bytes_to_text` 的判定顺序

```
1. 空 → ("", "empty")
2. BOM: EF BB BF → UTF-8 BOM；FF FE → UTF-16LE；FE FF → UTF-16BE
3. 纯 ASCII 可打印 (>95%) → ("ascii")
4. UTF-8 严格校验通过 → ("utf-8")
5. 偶数长度且 NUL 出现在奇数位 → UTF-16LE 猜测
6. 以上皆否 → UTF-8 lossy，标记 "utf-8-invalid"
```

**绝不 panic**：`from_utf8_lossy` 永远返回 String，坏字节变 U+FFFD。

---

### P2 · 安全读取层

#### 2.1 可读区间表（替掉硬编码段判断）

现有问题：整段 `0x6ddca*` 被判不可读，但旁边三段都读得好 → 硬编码段逻辑有漏洞。

**改法**：启动时解析一次 `/proc/self/maps`，缓存区间表：

```
struct Range { start: u64, end: u64, perm: u8 }
fn readable(addr: u64, len: usize) -> bool   // 逐区间覆盖判断，不跨区间就整体判死
```

判定要**按地址逐区间**，不是"命中白名单前缀"。

#### 2.2 分页安全读（不让坏页拖垮整次读）

**核心手法**：不要直接 `copy_nonoverlapping`（未映射页会 SIGSEGV），改用核内异常回传：

```
for 每一页:
    pipe(&mut fds)
    write(fds[1], addr, chunk)   // 内核 copy_from_user → EFAULT，不崩
    if write 失败 { gaps.push(page_addr); continue }
    read(fds[0], &mut buf[chunk_start..], got)
```

规则：

| 情况 | 行为 |
|:--|:--|
| 起始页未映射 | ❌ 整个请求返回 `{"error":"addr_not_mapped", "readable_ranges":[...]}` |
| 中间页未映射 | ✅ 跳过，记入 `gaps`，继续读 |
| `bytes` 参数 | **必须生效**（旧版硬写 4096，忽略参数） |
| 上限 | 单次 ≤ 64 KiB，超出分页 |

#### 2.3 SIGSEGV 隔离：**顺序，不嵌套**

**根因（已实锤）**：全插件只有一个全局 `static mut SIGSEGV_JMP_BUF`。嵌套 `sigsetjmp` 会互相踩，第二个 guard 覆盖第一个的 buf，longjmp 回到错误位置。

**改法**：

```
fn section(parts: &mut Vec<String>, key: &str, f: unsafe fn() -> String)
```

- 每段开跑**前**重新 arm 自己的 `sys_sigsetjmp`
- `f` 必须是**函数指针**，不能是闭包（闭包在 longjmp 重入时会二次 drop）
- 一段失败 → 该段输出 `{"error":"sigsegv", "section":key}`，其余四段照常返回

#### 2.4 每次读对象前的三连校验

```
① 指针非空
② 对象头 klass 指针落在合法堆范围（不是 0x0 / 0xffff...）
③ 该 klass 的 name 与期望类名一致
```

**目的**：主界面停留时，`get_SingleMode` 会返回已释放对象，判空过了照样崩。

---

### P3 · 通用解码器

#### 3.1 格式探测（按 magic bytes，先长后短）

| 优先级 | Magic（hex） | 格式 | 解码器 |
|:-:|:--|:--|:--|
| 1 | `53 51 4C 69 74 65 20 66 6F 72 6D 61 74 20 33 00` | SQLite 3 | `rusqlite` 只读打开 |
| 2 | `7F 45 4C 46` | ELF | 只读头 + 段表 |
| 3 | `04 22 4D 18` | LZ4 frame | 解压后再判 |
| 4 | `1F 8B` | gzip | 解压后再判 |
| 5 | `78 01 / 78 9C / 78 DA` | zlib | 解压后再判 |
| 6 | `50 4B 03 04` | ZIP | 列目录 |
| 7 | `55 6E 69 74 79 46 53` / `55 6E 69 74 79 43 46 53` | UnityFS/CAB | 标 `asset-bundle`，不深解 |
| 8 | `AF 1B B1 FA` | IL2CPP metadata | 只报版本号 + 段数 |
| 9 | `89 50 4E 47` / `4F 67 67 53` | PNG / OGG | `binary` |
| 10 | `FF FE` / `FE FF` | UTF-16 LE/BE | 转 UTF-8 |
| 11 | `EF BB BF` | UTF-8 BOM | 去 BOM → UTF-8 |
| 12 | 首个非空白 ∈ `{ [ "` | JSON | `serde_json` 校验 |
| 13 | 首字节 `0x80–0x8F` / `0x90–0x9F` | msgpack map/array | 现有 `protocol_msgpack::decode_to_json` |
| 14 | 其余 | 未知 | 文本判定（§1.3）+ hex 预览 |

**注意**：msgpack 的 `fixint` 与二进制数据首字节高度重叠（`0x00–0x7F` 既是正整数也是裸字节）。**不要**用首字节判 msgpack，只认 `fixmap(0x80–0x8F)` / `fixarray(0x90–0x9F)` / `0xD9–0xDB` 且**解码不报错**才算。

#### 3.2 解压防护

| 防护 | 阈值 |
|:--|:--|
| 解压后总大小 | ≤ 64 MiB（超限中止） |
| 压缩比 | ≤ 1000:1（防 zip bomb） |
| 解压层数 | ≤ 3 |
| 单次响应 | ≤ 256 KiB（超出改为落盘 + 返回摘要） |

#### 3.3 递归深度

msgpack / JSON 递归上限 **8 层**，超限返回 `{"error":"too_deep","at":N}`，绝不爆栈。

---

### P4 · 端点注册（**四处都要改，少一处就是"端点不存在"**）

新端点必须在以下 4 处登记，缺一不可：

| # | 位置（锚点） | 参考行号 | 改什么 |
|:-:|:--|:-:|:--|
| 1 | 路由分发 `} else if path == "/api/sniff/clear" {` 之后 | ~8366 | 加 `else if` 分支 |
| 2 | `DL_ALLOWED` 数组 `"/api/sniff/clear",` | ~8213 | 加路径，否则下载端点 403 |
| 3 | `/health` 的 endpoints 列表 | ~8267 | 加路径，否则文档/自检看不到 |
| 4 | `not_found` 的 `available` 数组 | ~9983 | 加路径，否则 404 提示是旧列表 |

**参考行号会随改动漂移，永远按字符串锚点定位。**

同时：**不要**加进 `BOOT_SAFE_*`——这些端点摸活内存，必须游戏初始化后才可用。

---

### P5 · 统一输出契约

#### 5.1 Envelope

所有端点统一返回：

```json
{
  "ok": true,
  "endpoint": "/debug/decode",
  "elapsed_ms": 12,
  "data": { },
  "gaps": [],
  "warnings": []
}
```

失败：

```json
{
  "ok": false,
  "endpoint": "/debug/decode",
  "error": "addr_not_mapped",
  "code": "E_ADDR",
  "hint": "起始页未映射，见 readable_ranges",
  "readable_ranges": ["0x6dd0000000-0x6dd8000000"]
}
```

#### 5.2 错误码表（固定，不要自由发挥）

| code | error | 含义 |
|:--|:--|:--|
| `E_ARG` | `missing_param` | 缺参数 |
| `E_ARG` | `bad_param` | 参数非法（hex/数字解析失败） |
| `E_ADDR` | `addr_not_mapped` | 起始地址未映射 |
| `E_ADDR` | `addr_not_readable` | 有映射但无读权限 |
| `E_OBJ` | `object_invalid` | klass 校验失败 |
| `E_FMT` | `unknown_format` | 探测不出格式 |
| `E_FMT` | `decode_failed` | 解码器报错 |
| `E_LIM` | `too_large` | 超体积上限 |
| `E_LIM` | `too_deep` | 超递归深度 |
| `E_IO` | `io_error` | 文件/管道错误 |
| `E_SIG` | `sigsegv` | 段错误已隔离 |
| `E_SQL` | `sql_denied` | 只读守卫拦截 |

---

## 3. 端点规格

> 下面每个端点都不新增监听端口，全挂 `127.0.0.1:18765`。
> **参数一律要求 URL 编码（`percent-encoding`）**，服务端按 §1.2 顺序解。

### 3.1 编码自检 `/debug/encoding/selftest`

| 项 | 值 |
|:--|:--|
| 参数 | 无 |
| 返回 | 固定 6 组用例的往返结果 |

```
"あ"(%E3%81%82) → "あ"        UTF-8
"応"            → "応"        UTF-16LE
"\u0000ab"      → "\u0000ab"  JSON 转义
0x80 0x81 0x82  → "..." lossy  
""              → ""          
0xE3 0x81 (截断) → "�"        不 panic
```

**这个端点必须先做**——它是编码层唯一的自动化验证，后面所有端点都依赖它。

### 3.2 内存读 `/il2cpp/read_mem`（改造）

| 参数 | 必填 | 说明 |
|:--|:-:|:--|
| `addr` | ✅ | `0x` 前缀十六进制 |
| `bytes` | ❌ | 默认 4096，**上限 65536**（旧版忽略此参） |

返回新增 `gaps[]`：未映射页地址列表。

### 3.3 文件读 `/debug/read_file`（新增）

| 参数 | 必填 | 说明 |
|:--|:-:|:--|
| `path` | ✅ | **URL 编码的绝对路径** |
| `as` | ❌ | `auto`（默认）/`text`/`hex`/`base64` |
| `offset` / `bytes` | ❌ | 分片读，默认 0 / 65536 |

**安全**：只允许白名单根目录下的路径，解析后必须做 `..` 折叠，拒绝逃逸。

### 3.4 格式探测 `/debug/sniff`（新增）

| 参数 | 必填 | 说明 |
|:--|:-:|:--|
| `path` 或 `addr` | ✅ | 二选一 |
| `bytes` | ❌ | 探测窗口，默认 512 |

返回：`{"format":"sqlite","magic":"53514C697465...","confidence":"high","notes":[]}`

### 3.5 统一解码 `/debug/decode`（新增，核心）

| 参数 | 必填 | 说明 |
|:--|:-:|:--|
| `path` 或 `addr` | ✅ | 二选一 |
| `as` | ❌ | `auto` 默认，或强制指定格式 |
| `depth` | ❌ | 默认 8 |

流程：**读字节 → §3.1 探测 → 解码 → 统一 envelope**。这一个端点吃掉所有格式，不用为每种格式建一个。

### 3.6 SQLite 只读 `/mdb/raw`（已存在，需加固）

| 坑 | 说明 |
|:--|:--|
| 只读守卫生效时**表名含 `update` 会被当成写操作拒绝**（`T008`） | 用别名或 `sqlite_master` 间接查 |
| `%` 在 URL 里会被吃掉 | 模糊匹配改用 `GLOB 'xxx*'` |
| 查询串含日文会乱码 | 走 §1.2 修好的 `decode_query` |

### 3.7 剧本15 专用（已有，保持）

```
/debug/training_entities    5 个 CommandEntity 全字段
/debug/acts                 5 行 act_type/level/gauge/advice_id
/debug/params?cmd=106|?all=1
```

---

## 4. 偏移常量全表

### 4.1 IL2CPP 通用布局

| 项 | 值 | 备注 |
|:--|:-:|:--|
| 对象头 | 16 B | klass@+0，monitor@+8；**字段从 +0x10 开始** |
| `List<T>` count | 0x18 | |
| `List<T>` items 指针 | 0x10 | 指向底层数组 |
| `List<T>` 元素 | `array + 0x20 + i*8` | |
| `T[]` 长度 | 0x18 | |
| `T[]` 元素 | `array + 0x20 + i*8` | |
| ObscuredInt 步长 | **0x14** | 相邻字段间隔 0x14，别按 4/8 算 |
| ObscuredInt 解码 | `key@off`，`hidden@off+4`，`值 = hidden ^ key` | **key 每局变，禁止硬编码** |

### 4.2 场景链

```
WorkDataManager._instance + 0x60
  → WorkSingleModeData + 0x58
  → WorkSingleModeCharaData + 0x4D0   (ScenarioFesta；+0x4F8 = Ramen)
  → Scenario + 0x10
  → FestaDataSet
```

### 4.3 FestaDataSet 字段

| 字段 | 偏移 |
|:--|:-:|
| FestaInfo | 0x10 |
| CommandInfoArray | 0x18 |
| EvaluationInfoArray | 0x20 |
| AdviceInfoArray | 0x28 |
| ActInfoArray | 0x30 |
| CheckPointResultArray | 0x38 |
| CommandGainActGaugeArray | 0x40 |
| NotUpParameterInfo | 0x48 |
| AutoSettingInfo | 0x50 |

### 4.4 ActInfo 元素（5 个）

| 字段 | 偏移 |
|:--|:-:|
| ActType | 0x10 |
| ActGauge | 0x24 |
| **Level** | **0x38** |
| NeedActGauge | 0x4C |
| ActTrainingState | 0x60 |
| AdviceId | 0x74（**每回合动态值**） |
| FriendOutingGainActLevel | 0x88 |

### 4.5 训练命令实体

| 字段 | 偏移 | 类型 |
|:--|:-:|:--|
| CommandType | 0x10 | i32 |
| CommandId | 0x14 | i32 |
| IsEnable | 0x18 | **u8（1 字节）** |
| TrainingLevel | 0x20 | i32 |
| TrainingFailureRate | 0x24 | i32（单位未定） |
| 下层 GainParameterList | 0x28 | ptr |
| 上层 GainBonusParameterList | 0x30 | ptr |

参数项：`ParameterType@0x10`、`Value@0x14`，**均普通 int32**。

`ParameterType`：1速 2耐 3力 4毅 5智 10体力 30SP

### 4.6 队友支援

`TrainingPartnerId@0x10`、`EffectArray@0x28`；元素 `EffectType@0x10`、`EffectValue@0x18`（Obscured 布局未验证 → 三种解释都输出）

---

## 5. 编码与转义规则（最容易翻车的部分）

### 5.1 输入侧

| 环节 | 规则 |
|:--|:--|
| URL 解析 | 先 `%XX` → 字节，再把 `+` 当空格，再 UTF-8 |
| base64 参数 | **不要**经过 `+`→空格那步；用 base64url 或强制 `%2B` |
| 路径参数 | 必须 `percent-encode`；反斜杠统一成正斜杠；折叠 `..` |
| hex 参数 | 允许 `0x` 前缀/大写；解析失败 → `E_ARG`，不要默默当 0 |

### 5.2 IL2CPP 字符串

| 项 | 规则 |
|:--|:--|
| 布局 | 长度 `i32@+0x10`（**字符数，不是字节数**），字符 `UTF-16LE@+0x14` |
| 空指针 | 返回 `""`，**不要返 `null`** |
| 长度异常 | `len <= 0` 或 `len > 4096` → 丢弃返回 `""` + warning |
| 转换 | `il2cpp_string_new_utf16` 或自写 `u16 → char` |

### 5.3 输出侧（JSON）

| 字符 | 处理 |
|:--|:--|
| `"` `\` | 转义 |
| `\n` `\r` `\t` | 转义为 `\n` `\r` `\t` |
| `< 0x20` 控制字符 | `\uXXXX` |
| 孤立代理项（U+D800–DFFF 单独出现） | 替换为 `\uFFFD` |
| 非法 UTF-8 字节 | lossy → U+FFFD + `warnings:["invalid_utf8@offset"]` |
| 二进制 | `base64`（标准，带 padding）+ 可选 hex 预览 |
| hex dump | 每行 16 字节，`%04X: XX XX ... |ascii|` |

**禁止**：把原始二进制直接塞进 JSON 字符串（会产出非法 JSON，客户端解析崩）。

### 5.4 数值

| 项 | 规则 |
|:--|:--|
| i32/i64 | 按实际宽度读，**不要一律 `read_int_at`** |
| hex 输出 | 小写、带 `0x` |
| NaN / Inf | 输出 `null`，不输出 `NaN`（非法 JSON） |
| 未解析字段 | `-1` 或 `null`，**不要填 0 冒充真值** |

### 5.5 字节序

| 数据 | 字节序 |
|:--|:--|
| IL2CPP 结构（arm64 原生） | 小端 |
| SQLite | 固定小端 |
| UTF-16 | 看 BOM；无 BOM 默认 LE |
| ELF | 看 `EI_DATA`（`01`=LE / `02`=BE） |
| msgpack | 自带类型标记，无字节序问题 |

---

## 6. 安全硬规则（写进 review checklist）

| # | 规则 | 后果 |
|:-:|:--|:--|
| 1 | 所有触摸游戏内存的端点必须 `READ_MUTEX` + `sys_sigsetjmp` + `catch_unwind(AssertUnwindSafe)` | 不加则一崩全崩 |
| 2 | `f` 用函数指针，不用闭包 | 闭包 longjmp 重入 = 双 drop |
| 3 | 单次响应 ≤ 256 KiB，超出改落盘 | 大响应会断流 |
| 4 | 递归 ≤ 8 层 | 爆栈 |
| 5 | 解压 ≤ 64 MiB、压缩比 ≤ 1000:1 | zip bomb |
| 6 | 文件端点必须有根目录白名单 | 任意读 = 提权 |
| 7 | 不缓存对象指针 | 已释放对象 |
| 8 | 不加进 `BOOT_SAFE_*` | 初始化前访问未就绪运行时 |

---

## 7. 构建与版本

| 项 | 值 |
|:--|:--|
| Cargo 版本 | 每次改动 bump（`3.28.5` → `3.28.6` …） |
| 版本串 | `VERSION_SUFFIX = if cfg!(feature="gadget") {"x.y.z+g"} else {"x.y.z+c"}` |
| 工具链 | NDK r26c / clang 17.0.2 |
| 命令 | `cargo build --release --target aarch64-linux-android --features gadget --offline` |
| profile | `lto=true, codegen-units=1, opt-level=3, strip=true, panic="unwind"` |
| 产物 | `libhachimi_ura.so` + 记录字节数 + sha256 |
| CI | 只在 `v*` tag / `workflow_dispatch` 构建；**workbench 分支 push 不自动出包** |

> `panic="unwind"` 是必须的——`catch_unwind` 在 `panic=abort` 下不生效。

---

## 8. 装机步骤

```
1. 备份旧 so（改名为 .so.bak，留在原目录）
2. 替换 lib/arm64-v8a/libhachimi_ura.so
3. 重签 APK → 覆盖安装
4. 启动游戏 → 停在主界面 20~30 秒不动
5. 再进育成
```

**装机竞态是历史事故第一来源**：游戏运行中途打补丁 → 撞上正在执行的函数 → 无声 SIGSEGV。等够时间再操作。

---

## 9. 验收清单

按顺序做，前一步不通不进下一步。

| 步 | 端点 | 期望 | 通过 |
|:-:|:--|:--|:-:|
| 1 | `/health` | 版本号 = 本次 bump | ☐ |
| 2 | `/debug/encoding/selftest` | 6 组全绿 | ☐ |
| 3 | `/debug/sniff?path=<日文路径>` | 正确识别 + 不乱码 | ☐ |
| 4 | `/il2cpp/read_mem?addr=0x...&bytes=16` | 只回 16 字节（不是 4096） | ☐ |
| 5 | 同端点跨页地址 | 返回 + `gaps` 非空，不报错 | ☐ |
| 6 | `/debug/decode?path=<sqlite>` | 识别 `sqlite` + 表数正确 | ☐ |
| 7 | `/debug/decode?path=<msgpack>` | 解出 JSON，中文正常 | ☐ |
| 8 | `/debug/training_entities` | 5 行，106 号下层 25 / 上层 6 | ☐ |
| 9 | `/debug/acts` | 5 行，各 offset 合理 | ☐ |
| 10 | `/debug/all` | **5 段全部有返回**（允许个别 `sigsegv`） | ☐ |
| 11 | 压测 100 次 | 进程存活、无内存增长 | ☐ |

---

## 10. 历史踩过的坑（别重复）

| 坑 | 表现 | 根因 |
|:--|:--|:--|
| Latin-1 乱码 | 日文查询变 `ã�‚` | `url_decode` 逐字节 `as char` |
| `read_mem` 参数无效 | 永远回 4096 字节 | 硬编码，忽略 `bytes` |
| 整段不可读 | `0x6ddca*` 判死但邻段正常 | 硬编码段白名单 |
| `/debug/all` 段错误 | `sigsegv_recovered_in_debug_all` | 全局单例 `SIGSEGV_JMP_BUF` 嵌套互踩 |
| 全量 dump 崩游戏 | 响应中途断流 + 进程消失 | 一次 dump 31,420 类 |
| 进育成闪退 | 无 crashlog | 运行中装机（懒补装） |
| SQL 被拒 | `T008` | 只读守卫把含 `update` 的表名当写操作 |
| 指针缓存失效 | 读数全是垃圾 | 对象图每帧重建 |
| `panic=abort` 编译 | `catch_unwind` 失效 | profile 配置错 |

---

## 11. 禁止事项

| ❌ 不要 | 原因 |
|:--|:--|
| 新开 HTTP 监听端口 | 无收益，只增加线程/端口风险 |
| CI 里跑脚本改源码 | 锚点断言一改就静默失效 |
| 用首字节判 msgpack | 与 `fixint` / 二进制重叠，误判率极高 |
| 把二进制塞进 JSON 字符串 | 产出非法 JSON |
| 未解析字段填 0 | 0 是合法值，无法区分"没读到" |
| 硬编码 ObscuredInt key | 每局变 |
| 缓存跨请求指针 | 已释放 |

---

## 附录 A · 源码锚点速查

> 行号会漂移，**永远按字符串定位**。

| 用途 | 锚点字符串 | 参考行 |
|:--|:--|:-:|
| 版本串 | `VERSION_SUFFIX`（`src/lib.rs` 顶部） | ~27 |
| 插件版本常量 | `const PLUGIN_VERSION: &str = env!("CARGO_PKG_VERSION");` | — |
| 路由分发 | `} else if path == "/api/sniff/clear" {` | ~8366 |
| 下载白名单 | `            "/api/sniff/clear",`（`DL_ALLOWED`） | ~8213 |
| health 列表 | `/health` 的 endpoints 数组 | ~8267 |
| 404 列表 | `not_found` 的 `available` | ~9983 |
| 字符串读取 | `read_il2cpp_string` | — |
| 指针读取 | `read_ptr_at` / `read_int_at` | — |
| API 结构 | `&*super::API`（字段 `il2cpp_get_method_addr_fn` / `interceptor`） | — |
| U16 字符串 | `resolve_il2cpp_symbol("il2cpp_string_new_utf16")` | — |
| 互斥 | `READ_MUTEX` | — |
| 崩溃兜底 | `sys_sigsetjmp` / `sys_siglongjmp` | — |
| 分段执行 | `debug_all_section` | — |

---

## 附录 B · 一页速查

```
先编码(P1) → 再安全读(P2) → 再解码(P3) → 再注册(P4) → 再统一输出(P5)

新端点必须在 4 处登记：路由 / DL_ALLOWED / health / not_found
端口不动：127.0.0.1:18765

读内存：分页 pipe 读，只判起始页
读对象：指针→klass校验→名字比对，三步都要
读字符串：长度@+0x10（字符数），字符 UTF-16LE@+0x14
出 JSON：控制字符转义，二进制转 base64，非法 UTF-8 走 lossy
```

---

*本文件描述规格，不含实现。任何一步改动后请更新 §9 的勾选状态。*
