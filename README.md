# hlpatch 插件改造手册（单文件版）

> 原 `docs/READALL-SPEC-v1.md` 全部并入本文件，后续只维护这一份。
> 本文件只写「你要改什么、改哪里、改成什么」，代码由你自己落。

---

## 0. 怎么用这份文件

1. 从上往下做，**不要跳**（2.2 → 2.3 有依赖）
2. 每项都给了**锚点字符串**，用 grep 定位，别信行号（会漂移）
3. 每项末尾有「验证」，做完立刻验，别攒着
4. 一次只改一项，改完能编译再动下一项

---

## 1. 改动总览

| # | 文件 | 改动 | 必需 | 影响面 |
|:-:|:--|:--|:-:|:--|
| 2.1 | `hachimi_ura_plugin/Cargo.toml` | 版本 bump | ✅ | 无 |
| 2.2 | `src/lib.rs` | 修 `url_decode` 的 Latin-1 问题 | ✅ | 查询参数乱码 |
| 2.3 | `src/lib.rs` | 字段偏移改成 **symbol 优先 + 常量兜底** | ✅ | 抗字段重排 |
| 2.4 | `src/lib.rs` | `read_mem` 支持 `bytes`、分页安全读 | ✅ | 读内存不再整段失败 |
| 2.5 | `src/lib.rs` | 可读区间改用 `/proc/self/maps` | ✅ | 判映射不再误杀 |
| 2.6 | `src/lib.rs` | SIGSEGV 隔离改**顺序**不嵌套 | ✅ | `/debug/all` 不再整体崩 |
| 2.7 | `src/lib.rs` | 编码 / 转义 helper | ✅ | 输出永远合法 JSON |
| 2.8 | `src/lib.rs` | 新端点 + **4 处注册** | ✅ | 端点能被访问 |

**不动的东西**：监听端口（固定 18765）、hook 机制、读值方式（保持裸内存读）。

---

## 2. 逐项改动

### 2.1 `Cargo.toml`

| 项 | 值 |
|:--|:--|
| 锚点 | `version = "3.28.` |
| 改成 | 手动 +1，例如 `3.28.5` → `3.28.6` |
| 同时改 | `src/lib.rs` 里的 `VERSION_SUFFIX`（文件顶部） |

```rust
VERSION_SUFFIX = if cfg!(feature="gadget") {"3.28.6+g"} else {"3.28.6+c"}
```

新增依赖：**不需要**。msgpack / SQLite / JSON 已有（`rmpv`、`rusqlite`、`serde_json`）。
解压（zlib/gzip）如要做，加 `miniz_oxide = "0.8"`（纯 Rust，无需 C 工具链）。

**验证**：`/health` 返回的 version 与 Cargo.toml 一致。

---

### 2.2 修 `url_decode`（日文乱码根因）

**锚点**：`fn url_decode`

**现状（错的）**：逐字节 `as char`，等于按 Latin-1 解释 → `%E3%81%82`（あ）解出来是 `ã`。

**改成**：先收字节，最后统一 UTF-8。

```rust
fn hexval(c: u8) -> Option<u8> {
    match c {
        b'0'..=b'9' => Some(c - b'0'),
        b'a'..=b'f' => Some(c - b'a' + 10),
        b'A'..=b'F' => Some(c - b'A' + 10),
        _ => None,
    }
}

fn url_decode(s: &str) -> String {
    let b = s.as_bytes();
    let mut out: Vec<u8> = Vec::with_capacity(b.len());
    let mut i = 0usize;
    while i < b.len() {
        if b[i] == b'%' && i + 2 < b.len() {
            if let (Some(h), Some(l)) = (hexval(b[i + 1]), hexval(b[i + 2])) {
                out.push((h << 4) | l);
                i += 3;
                continue;
            }
        }
        if b[i] == b'+' {
            out.push(b' ');
        } else {
            out.push(b[i]);
        }
        i += 1;
    }
    String::from_utf8_lossy(&out).into_owned()
}
```

**注意顺序**：`%XX` → 字节 → `+` 变空格 → UTF-8。顺序错了会把 base64 里的 `+` 吃掉。

**验证**：`/mdb/raw?sql=SELECT text FROM text_data WHERE text LIKE '%あ%'` 能查到日文。

---

### 2.3 字段偏移：symbol 优先 + 常量兜底

**这是最值钱的一项。** 现在偏移写死在代码里（`0x4D0`、`0x38`…），游戏一改字段顺序就静默读错 → 崩。

**锚点**：`fn find_field_offset`（已有实现，遍历 `FieldInfo` 逐个比名字）

**改成**：先请宿主按名字查，查不到再用常量。

```rust
// ⚠ 宿主 API 表里没有 il2cpp_field_get_offset，两种取法任选其一：

// 取法 A：解符号（最稳）
let f_get_offset = resolve_il2cpp_symbol("il2cpp_field_get_offset")
    .map(|p| std::mem::transmute::<_, unsafe extern "C" fn(*mut FieldInfo) -> i32>(p));

// 取法 B：直读 FieldInfo 布局（快，需实测确认）
// FieldInfo: name@0x00, type@0x08, parent@0x10, offset@0x18 (i32)
// read_int_at(field as usize + 0x18)
```

```rust
unsafe fn field_off(
    klass: *mut Il2CppClass,
    name: &std::ffi::CStr,
    fallback: i64,   // 原来硬编码的值，例如 0x4D0
) -> i64 {
    // 1) 宿主 API 按名字查
    if let Some(get_field_from_name) = (*API).il2cpp_get_field_from_name_fn {
        if !klass.is_null() {
            let f = get_field_from_name(klass, name.as_ptr());
            if !f.is_null() {
                let off = field_get_offset_impl(f); // 上面 A 或 B
                if off >= 0 {
                    return off as i64;
                }
            }
        }
    }
    // 2) 兜底 + 留痕
    log_once(&format!("field_off fallback: {:?} -> {:#x}", name, fallback));
    fallback
}
```

**调用侧改法**（逐一替换）：

```rust
// 旧
let festa = read_ptr_at(chara, 0x4D0);
let level = read_int_at(elem + 0x38);

// 新
let off_festa = field_off(k_chara, c"_scenarioFesta", 0x4D0);
let festa = read_ptr_at(chara, off_festa as usize);
let off_level = field_off(k_act, c"_actLevel", 0x38);
let level = read_int_at(elem + off_level as usize);
```

**缓存**：`(klass, name) → offset` 用 `Mutex<HashMap>` 存，一次查永久用。
**缓存偏移可以，缓存对象指针不行**（对象图每帧重建）。

**边界**：`ObscuredInt` 的**内部**布局（key@+0，hidden@+4）仍是常量，symbol 帮不上，保留实测值。

**验证**：`/debug/acts` 的 `act_level` 与游戏内演目等级一致。

---

### 2.4 `read_mem`：支持 `bytes` + 分页安全读

**锚点**：`"/il2cpp/read_mem"`、`addr_not_mapped`

**两个现状 bug**：
1. 忽略 `bytes` 参数，永远返回 4096 字节
2. 任何一页未映射 → 整个请求失败

**改法**：

```rust
/// 返回 (数据, 未映射页地址)；仅起始页未映射才算失败
fn read_bytes_safe(addr: usize, len: usize) -> Result<(Vec<u8>, Vec<usize>), &'static str> {
    let mut buf = vec![0u8; len];
    let mut gaps: Vec<usize> = Vec::new();
    let mut done = 0usize;
    let mut first = true;

    while done < len {
        let cur = addr + done;
        let page_left = 4096 - (cur & 0xfff);
        let chunk = page_left.min(len - done);

        // pipe + write：内核 copy_from_user，坏页返回 EFAULT 而非 SIGSEGV
        let mut fds = [0i32; 2];
        if unsafe { libc::pipe(fds.as_mut_ptr()) } != 0 { return Err("pipe_failed"); }
        let w = unsafe { libc::write(fds[1], cur as *const libc::c_void, chunk) };
        if w > 0 {
            unsafe {
                libc::read(fds[0], buf[done..].as_mut_ptr() as *mut libc::c_void, w as usize);
                libc::close(fds[0]);
                libc::close(fds[1]);
            }
            done += w as usize;
        } else {
            unsafe { libc::close(fds[0]); libc::close(fds[1]); }
            if first { return Err("addr_not_mapped"); }
            gaps.push(cur);
            done += chunk;
        }
        first = false;
    }
    Ok((buf, gaps))
}
```

参数解析：

```rust
let size = q.get("bytes").or_else(|| q.get("size"))
    .and_then(|v| v.parse::<usize>().ok())
    .unwrap_or(4096)
    .min(65536);
```

输出新增：`"bytes_read": N`、`"gaps": ["0x..."]`。

**验证**：`?bytes=16` 只回 16 字节；跨页地址带 `gaps` 但不报错。

---

### 2.5 可读区间：改用 `/proc/self/maps`

**锚点**：任何判断"这段地址能不能读"的地方（关键词 `addr_not_mapped`）

**现状问题**：硬编码段白名单，导致 `0x6ddca*` 整段被判死，但相邻三段都读得动。

```rust
#[derive(Clone, Copy)]
struct Rng { start: usize, end: usize }

static MAPS_CACHE: std::sync::Mutex<Option<Vec<Rng>>> = std::sync::Mutex::new(None);

fn readable_ranges() -> Vec<Rng> {
    let mut g = MAPS_CACHE.lock().unwrap();
    if let Some(v) = g.as_ref() { return v.clone(); }
    let mut out = Vec::new();
    if let Ok(s) = std::fs::read_to_string("/proc/self/maps") {
        for line in s.lines() {
            let mut it = line.split_whitespace();
            let (Some(range), Some(perm)) = (it.next(), it.next()) else { continue };
            if !perm.starts_with('r') { continue }
            let mut p = range.split('-');
            let (Some(a), Some(b)) = (p.next(), p.next()) else { continue };
            if let (Ok(a), Ok(b)) = (usize::from_str_radix(a, 16), usize::from_str_radix(b, 16)) {
                out.push(Rng { start: a, end: b });
            }
        }
    }
    out.sort_by_key(|r| r.start);
    *g = Some(out.clone());
    out
}

fn covered(addr: usize, len: usize) -> bool {
    let (s, e) = (addr, addr.saturating_add(len));
    readable_ranges().iter().any(|r| r.start <= s && e <= r.end)
}
```

**注意**：进程加载新 so 后 maps 会变。缓存要在**每次装机后失效**，或干脆每次请求重读（几毫秒，可接受）。

**验证**：`?addr=0x6ddca350` 不再整体报错。

---

### 2.6 SIGSEGV 隔离：顺序，不嵌套

**锚点**：`SIGSEGV_JMP_BUF`、`debug_all`

**根因**：只有一个全局 jmp_buf。嵌套 `sigsetjmp` 互相覆盖，longjmp 跳到错误位置。

```rust
fn section(parts: &mut Vec<String>, key: &str, f: unsafe fn() -> String) {
    unsafe {
        if sys_sigsetjmp(SIGSEGV_JMP_BUF.as_ptr(), 1) == 0 {
            let s = f();
            parts.push(format!("{{\"{}\":{}}}", key, s));
        } else {
            parts.push(format!("{{\"{}\":{{\"error\":\"sigsegv\"}}}}", key));
        }
    }
}

// debug_all 里
let mut parts = Vec::new();
section(&mut parts, "summary",   read_summary_inner);
section(&mut parts, "scenario",  read_scenario_detail);
section(&mut parts, "storydata", debug_storydata_inner);
section(&mut parts, "cmdinfo",   debug_cmdinfo);
section(&mut parts, "rameninfo", read_ramen_info);
```

**两条硬要求**：
1. `f` 是**函数指针**，不能是闭包——闭包在 longjmp 重入时会二次 drop
2. 顺序执行，**绝不允许**在 `section` 里再调 `section`

另：`catch_unwind` 只在 `panic="unwind"` 时生效，`[profile.release]` 必须是 `panic = "unwind"`。

**验证**：`/debug/all` 返回 5 个 key，即使其中一个含 `"sigsegv"`。

---

### 2.7 编码 / 转义 helper

**锚点**：`fn read_il2cpp_string`

补三个函数：

```rust
// 1) IL2CPP 字符串：长度是「字符数」不是字节数
unsafe fn il2cpp_str(p: usize) -> String {
    if p == 0 { return String::new(); }
    let len = read_int_at(p + 0x10);          // i32
    if len <= 0 || len > 4096 { return String::new(); }
    let chars = p + 0x14;                      // UTF-16LE
    let mut v = Vec::with_capacity(len as usize);
    for i in 0..len as usize {
        v.push(read_u16_at(chars + i * 2));
    }
    String::from_utf16_lossy(&v)
}

// 2) JSON 转义
fn escape_json(s: &str) -> String {
    let mut o = String::with_capacity(s.len() + 8);
    for ch in s.chars() {
        match ch {
            '"'  => o.push_str("\\\""),
            '\\' => o.push_str("\\\\"),
            '\n' => o.push_str("\\n"),
            '\r' => o.push_str("\\r"),
            '\t' => o.push_str("\\t"),
            c if (c as u32) < 0x20 => o.push_str(&format!("\\u{:04x}", c as u32)),
            c => o.push(c),
        }
    }
    o
}

// 3) 数值：NaN/Inf 必须变 null，否则不是合法 JSON
fn num(f: f64) -> String {
    if f.is_finite() { format!("{}", f) } else { "null".into() }
}
```

**二进制**：一律 base64，不要塞原始字节；另给 `%04X: XX XX ... |ascii|` 的 hex 预览。

**验证**：`curl ... | jq .` 对任意端点都不报错。

---

### 2.8 新端点 + **4 处注册**

#### 要加的端点

| path | 参数 | 作用 |
|:--|:--|:--|
| `/debug/encoding/selftest` | 无 | 6 组编码往返自检 |
| `/debug/read_file` | `path`(URL编码) `as` `offset` `bytes` | 读文件，白名单根目录 |
| `/debug/sniff` | `path` 或 `addr` | 只报格式，不解码 |
| `/debug/decode` | `path` 或 `addr` `as` `depth` | 探测 + 解码一体 |

#### 4 处注册（少一处 = 端点"不存在"）

| # | 锚点 | 加什么 |
|:-:|:--|:--|
| 1 | `} else if path == "/api/sniff/clear" {` | 后面加 `else if` 分支 |
| 2 | `DL_ALLOWED` 里的 `"/api/sniff/clear",` | 加路径，否则下载端点 403 |
| 3 | `/health` 的 endpoints 数组 | 加路径 |
| 4 | `not_found` 的 `available` 数组 | 加路径，否则 404 提示是旧列表 |

```rust
} else if path == "/debug/decode" {
    debug_decode_endpoint(&full_uri)
} else if path == "/debug/sniff" {
    debug_sniff_endpoint(&full_uri)
} else if path == "/debug/read_file" {
    debug_read_file_endpoint(&full_uri)
} else if path == "/debug/encoding/selftest" {
    debug_encoding_selftest()
}
```

**不要**加进 `BOOT_SAFE_*`——这些端点摸活内存，必须游戏初始化后才可用。

#### 格式探测优先级（sniff / decode 共用）

| 序 | Magic | 格式 |
|:-:|:--|:--|
| 1 | `53 51 4C 69 74 65 20 66 6F 72 6D 61 74 20 33 00` | SQLite 3 |
| 2 | `7F 45 4C 46` | ELF |
| 3 | `04 22 4D 18` | LZ4 |
| 4 | `1F 8B` | gzip |
| 5 | `78 01/9C/DA` | zlib |
| 6 | `50 4B 03 04` | ZIP |
| 7 | `55 6E 69 74 79 46 53` | UnityFS |
| 8 | `AF 1B B1 FA` | IL2CPP metadata |
| 9 | `FF FE` / `FE FF` | UTF-16 LE/BE |
| 10 | `EF BB BF` | UTF-8 BOM |
| 11 | 首非空白 ∈ `{ [ "` | JSON |
| 12 | 首字节 `0x80–0x8F` / `0x90–0x9F` **且解码成功** | msgpack |
| 13 | 其余 | 文本 / hex 预览 |

**msgpack 那条必须"解码成功才算"**——`fixint` 范围与裸二进制完全重叠，只看首字节误判率极高。

**解压防护**：解压后 ≤ 64 MiB、压缩比 ≤ 1000:1、嵌套 ≤ 3 层、递归 ≤ 8 层。

---

## 3. 偏移与布局速查

### 3.1 IL2CPP 通用

| 项 | 值 |
|:--|:-:|
| 对象头 | 16 B（字段从 **+0x10** 起） |
| `List<T>` count | 0x18 |
| `List<T>` items 指针 | 0x10 |
| `List<T>` 元素 | `array + 0x20 + i*8` |
| `T[]` 长度 / 元素 | 0x18 / `array + 0x20 + i*8` |
| ObscuredInt 步长 | **0x14** |
| ObscuredInt 解码 | `值 = hidden ^ key`；key@+0，hidden@+4；**key 每局变** |

### 3.2 场景链

```
WorkDataManager._instance + 0x60
  → WorkSingleModeData + 0x58
  → WorkSingleModeCharaData + 0x4D0    (Festa；+0x4F8 = Ramen)
  → Scenario + 0x10
  → FestaDataSet
```

### 3.3 FestaDataSet

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

### 3.4 ActInfo 元素

| 字段 | 偏移 |
|:--|:-:|
| ActType | 0x10 |
| ActGauge | 0x24 |
| **Level** | **0x38** |
| NeedActGauge | 0x4C |
| ActTrainingState | 0x60 |
| AdviceId | 0x74（每回合动态） |
| FriendOutingGainActLevel | 0x88 |

### 3.5 训练命令实体

| 字段 | 偏移 | 类型 |
|:--|:-:|:--|
| CommandType | 0x10 | i32 |
| CommandId | 0x14 | i32 |
| IsEnable | 0x18 | **u8** |
| TrainingLevel | 0x20 | i32 |
| TrainingFailureRate | 0x24 | i32（单位未定） |
| 下层 GainParameterList | 0x28 | ptr |
| 上层 GainBonusParameterList | 0x30 | ptr |

参数项：`ParameterType@0x10`、`Value@0x14`，**普通 int32**。
类型码：1速 2耐 3力 4毅 5智 10体力 30SP。

---

## 4. 编码与转义规则

| 场景 | 规则 |
|:--|:--|
| URL 参数 | `%XX` → 字节 → `+`变空格 → UTF-8 |
| base64 参数 | 不要走 `+`→空格；用 base64url 或强制 `%2B` |
| IL2CPP 字符串 | 长度@0x10（字符数）、字符 UTF-16LE@0x14；空指针→`""`；len≤0 或 >4096 丢弃 |
| JSON 输出 | `"` `\` `\n\r\t` 转义；`<0x20` → `\uXXXX`；孤立代理项 → `\uFFFD` |
| 非法 UTF-8 | lossy → U+FFFD，并加 `warnings:["invalid_utf8@N"]` |
| 二进制 | base64；不要直接塞字符串 |
| 未解析字段 | `-1` 或 `null`，**不要填 0** 冒充真值 |
| NaN / Inf | `null` |
| 字节序 | IL2CPP 小端；SQLite 固定小端；UTF-16 看 BOM（无 BOM 默认 LE）；ELF 看 `EI_DATA` |

---

## 5. 构建与装机

```bash
cargo build --release --target aarch64-linux-android --features gadget --offline
```

| 项 | 值 |
|:--|:--|
| 工具链 | NDK r26c / clang 17.0.2 |
| profile | `lto=true`、`codegen-units=1`、`opt-level=3`、`strip=true`、**`panic="unwind"`** |
| 产物 | `libhachimi_ura.so`，记字节数 + sha256 |
| CI | 只在 `v*` tag / `workflow_dispatch` 构建；workbench push 不自动出包 |

装机：

```
1. 旧 so 改名 .so.bak 留在原目录
2. 替换 lib/arm64-v8a/libhachimi_ura.so
3. 重签 APK → 覆盖安装
4. 启动游戏 → 停主界面 20~30 秒不动
5. 再进育成
```

**装机竞态是历史第一事故来源**：运行中打补丁 = 撞正在执行的函数 = 无声 SIGSEGV。

---

## 6. 验收清单

| 步 | 端点 | 期望 | 通过 |
|:-:|:--|:--|:-:|
| 1 | `/health` | 版本 = 本次 bump | ☐ |
| 2 | `/debug/encoding/selftest` | 6 组全绿 | ☐ |
| 3 | `/il2cpp/read_mem?addr=0x..&bytes=16` | 只回 16 字节 | ☐ |
| 4 | 跨页读 | 带 `gaps`，不报错 | ☐ |
| 5 | `/debug/sniff?path=<日文路径>` | 识别正确 + 不乱码 | ☐ |
| 6 | `/debug/decode?path=<sqlite>` | 认出 sqlite | ☐ |
| 7 | `/debug/acts` | 5 行，level 与游戏一致 | ☐ |
| 8 | `/debug/training_entities` | 5 行，106 号 下层25/上层6 | ☐ |
| 9 | `/debug/all` | 5 段都有返回 | ☐ |
| 10 | 连打 100 次 | 进程存活 | ☐ |
| 11 | `curl ... \| jq .` | 无解析错误 | ☐ |

---

## 7. 不要做的事

| ❌ | 原因 |
|:--|:--|
| 新开 HTTP 监听端口 | 无收益，只增线程与端口风险 |
| 用宿主 `il2cpp_get_field_value` **读值** | 必须附着线程，HTTP 线程直调 = SIGSEGV |
| 全量 dump classes | 曾把游戏打崩（31,420 类） |
| 缓存对象指针 | 对象图每帧重建 |
| 硬编码 ObscuredInt key | 每局变 |
| CI 里跑脚本改源码 | 锚点断言一改就静默失效 |
| 用首字节判 msgpack | 与 fixint 重叠，必须解码成功才算 |
| 未解析字段填 0 | 无法区分"没读到" |

---

## 8. 历史坑对照

| 坑 | 表现 | 根因 |
|:--|:--|:--|
| 日文乱码 | `あ` → `ã` | `url_decode` 逐字节 `as char` |
| `read_mem` 参数无效 | 永远 4096 | 硬编码，忽略 `bytes` |
| 整段不可读 | `0x6ddca*` 判死 | 硬编码段白名单 |
| `/debug/all` 段错误 | `sigsegv_recovered_in_debug_all` | 全局单例 jmp_buf 嵌套互踩 |
| 进育成闪退 | 无 crashlog | 运行中装机（懒补装） |
| UI 元素消失 | 按钮不出、剧情不切 | 自建 hook 拿不到 trampoline 直接 return，原方法一次不执行 |
| SQL 被拒 | `T008` | 只读守卫把含 `update` 的表名当写操作 |
| `catch_unwind` 失效 | 崩溃不被接住 | profile 是 `panic="abort"` |

---

## 附录 · 源码锚点速查

> 行号会漂移，**按字符串定位**。

| 用途 | 锚点 |
|:--|:--|
| 版本串 | `VERSION_SUFFIX` |
| 插件版本 | `const PLUGIN_VERSION: &str = env!("CARGO_PKG_VERSION");` |
| 路由分发 | `} else if path == "/api/sniff/clear" {` |
| 下载白名单 | `DL_ALLOWED` 里的 `"/api/sniff/clear",` |
| health 列表 | `/health` 的 endpoints 数组 |
| 404 列表 | `not_found` 的 `available` |
| 崩溃兜底 | `SIGSEGV_JMP_BUF` / `sys_sigsetjmp` / `debug_all` |
| 读内存 | `"/il2cpp/read_mem"`、`addr_not_mapped` |
| 字段查找 | `fn find_field_offset` |
| 字符串 | `fn read_il2cpp_string` |
| API 表 | `&*super::API` |
| 符号解析 | `resolve_il2cpp_symbol` |
| 互斥 | `READ_MUTEX` |
