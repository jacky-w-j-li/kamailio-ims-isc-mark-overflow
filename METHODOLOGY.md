# 在 Kamailio 上找代码、提 PR 的实战方法论

> 这是一份从 **2026 年 8–10 月连续一个多月的每日 GitHub 巡查 + 5 次真实补丁提交**中提炼出来的经验总结。
> 目标读者：想给 kamailio/kamailio 这类大型 C 项目稳定贡献 PR 的人。
> 核心结论先放前面：**小 PR（≤5 行、单文件）合并窗口 0–3 天；正确解一旦收敛到唯一，剩下的唯一竞争变量就是提交速度。**

---

## 0. 三条最重要的数据（记住就够用）

| 指标 | 数值 | 含义 |
|---|---|---|
| **规模 vs 耗时** | ≤5 行 / 1 文件：**中位 17.7h / 120h**；6–30 行 **145h**；2 文件 **228h** | 越小越快，**压行数优先于压文件数** |
| **维护者评审口味** | linuxmaniac **30.3h**（IMS 归他）< henningw **139h** < miconda **208.6h** | IMS 模块找 linuxmaniac 最快 |
| **被抢 4 次** | xcap（差 32 分钟）、#4888、#4926（9 天）、#4938（3 天） | **定稿当天必须提** |

> 一个反直觉但被反复验证的事实：当 bug 的正确修法是确定性的（同一套方法论会把它收敛到**逐字节相同**的解），你和竞争者写出完全一致的补丁只是时间问题。**团队内部撞车也是白做功**（一次 PR 只能合一个），所以"快"是硬约束，不是态度问题。

---

## 1. 每日巡查流程（4 步）

```
GET /repos/kamailio/kamailio/issues?state=open&per_page=100&sort=created&direction=desc
GET /repos/kamailio/kamailio/issues?state=open&per_page=100&sort=updated&direction=desc
```

1. **拉全量**（走 REST 列表，不走 Search API —— Search 会返回"限流假空"，让人误以为没东西）。
   - 用 `'pull_request' in item` 把 issue 和 PR 分离；
   - 从 PR 的 title/body 里抽 `#(\d{3,5})` 建一个"已被占"集合。
2. **看今天新动静**：新建的 issue/PR、有更新的（评论/commit 引用）、**master 最近 commit**（谁在动哪些模块 —— 判断"我的目标还活着吗"和"会不会被 maintainer 自己扫掉"）。
3. **逐个过决策树**（见 §2）。
4. **输出**：新 issue 清单 + master 动向 + 各候选可下手性（✅/❌ + 理由）+ 手上补丁状态 + 下一步建议。

> ⚠️ **curl 报 `http=000` 不等于断网**：Windows schannel 对 MITM 代理证书做吊销检查会误报。curl 必须加 `--ssl-no-revoke`；Python `urllib`/`requests` 自动读 `https_proxy` 环境变量反而最稳。曾经因此谎报"GitHub 被墙"，是个坑。

---

## 2. 决策树：这个目标值得下手吗

按顺序过，**任何一条 ❌ 就放弃**：

| # | 检查 | 不通过就放弃的理由 |
|---|---|---|
| 1 | `assignees` 为空？ | 有人 = 已被认领，哪怕对方还没提 PR（#4920 白做功教训） |
| 2 | 关联 PR = 0？ | 有就别提重复的 |
| 3 | maintainer 是否已下场**且在做定性**？ | 他在问澄清 = 还在诊断，抢了易撞车 / 被判 as designed |
| 4 | 根因能用源码坐实吗？ | 不能坐实就别动 —— 可能是用法误解（按设计如此） |
| 5 | 改动能压到 **1 个文件、≤5 行**吗？ | ≤5 行中位 17.7h，6–30 行 145h，差 8 倍 |
| 6 | 同源副本（`ims_*`、`*_scscf`）也查了吗？ | 不查就会被覆盖面更广的竞争者拿走（#4926 教训） |

**特殊判断**：maintainer 说 *"it will be reviewed and fixed"*（认可 bug）+ assignee 空 + 无 PR → 窗口开着，可以提。

**快速判据（新增）**：issue 正文里如果已经给了「出错行 + 正确写法」，默认**报告者就是提交者**，窗口以小时计（#4960：报告者贴了出错行，henningw 直接邀请他提 PR，3.5 小时后合入）。这类要么当天提，要么跳过，别进待办清单慢慢排。

---

## 3. issue 池枯竭时：5 条主动找活渠道

kamailio 全仓常只有 ~10–12 个 open issue，大多被占或是深水。**不要硬挑 issue**，按顺序找：

### 渠道 1：新 issue 池（每天先看）
当天/近 48h 新建的 issue，筛掉"已被 PR 占用"和"assignee 有人"的。窗口无规律，唯一安全做法是**当天动手**。

### 渠道 2：maintainer 的「否定评论」里挖需求（成本最低）
维护者拒绝一个 PR 时，常顺口给出他认为对的替代方案 —— 一句没人认领的准需求。
> 实例：PR #4940（dispatcher.reload_id）miconda 评论 *"I would prefer `dispatcher.update` that combines add and remove in a single operation"*。

⇒ 巡逻时**扫最近 24–48h 内 maintainer 的 PR review 评论**，特别是带 "could be useful" / "better to" / "I would prefer" 的。优点：maintainer 已表达过意图，"为什么做"这关过了。

### 渠道 3：commit 趋势分析
```
GET /repos/kamailio/kamailio/commits?per_page=100
```
按 subject 关键词归类（bounds/overflow/validation/length check/robust/clang-format/leak/crash），统计**作者 + 模块**。谁在做、做什么类型、**哪些模块已被扫过（避开撞车）**一目了然。
> ⚠️ 反向用法：maintainer 正在某模块批量加固 = 邻居模块马上也会被扫到。linuxmaniac 在 `ims_dialog` 连做 5 个安全加固 → `ims_isc` 是邻居，被扫到只是时间问题。

### 渠道 4：静态模式扫描
扫 `src/` 的危险模式（`sscanf` 无界 `%s`、`sprintf`、`strcpy`、`strcat`），按模块计数。**优先挑输入来自远端报文的**（SDP/SIP 解析路径），价值最高。
> ⚠️ **扫描器报的行号不可信**（去注释后会错位）→ 定位候选必须回 `grep -n`，别直接 `sed -n '<行号>p'`。

### 渠道 5：开高质量 issue + @ maintainer（提 PR 之外的并行路径）
主动发现的漏洞不一定要自己提 PR。报告本身能在几小时内被 maintainer 修掉：
- #4951（sandrogauci，ims_dialog 一字节越界）：10:06 开 → 12:46 linuxmaniac 用 `+1/−1` 关闭，**全程 2h40m**。
- #4886（ims_registrar_scscf off-by-one）：公开 issue 附建议补丁 → henningw **1 天**合入。

**照抄 sandrogauci 的报告模板**（henningw 最吃的"只写事实"风格）：
1. 开头直接给「引入它的 commit + 出问题的 diff 片段 + **兄弟模块的正确写法对照**」；
2. 复现只要「极简 kamailio.cfg（5–6 个模块、children=1）+ 一个 python 发包脚本」；
3. **必须给对照组**：parent 正常 / 改法正确后正常 / `MEMDBG=0` 不死；
4. **主动收缩影响面**：写 *"no non-debug crash is claimed"*，不夸大 → 可信度反而更高；
5. 末尾 `/cc @linuxmaniac @henningw` 直接 @ 到人（触发 mentioned 事件，提速明显）。

---

## 4. 找到目标后：定稿 → 提交（同一轮完成）

1. **拉 master 源码核对根因**（本地仓库常是旧版本，别在过期代码上做）；
2. 写补丁 → **clang-format 校验**（本地有 `clang-format`，`diff` 无差异才算 CLEAN）；
3. **核对 base blob**：`git rev-parse origin/master:<path>` 必须等于本地 `git diff` 的 `index <blob>..`。不等说明文件已被别人改过，补丁要重做；
4. 写草稿（标题 + 描述 + diff）→ **同一轮请示"发吧"**；
5. 授权后走 Git Data API（PAT 无 `workflow` scope 时不能 `git push` 动 `.github/workflows`，用 Git Data API 建 commit，parent 用 **fork 自己的 master sha**，不要用 upstream 的，否则建出悬空 commit）。

> 🔴 **流程硬约束**：因为抢活窗口极短（最短 32 分钟），**出草稿的那一轮就必须把"发吧"问出去，绝不让定稿跨天**。

---

## 5. 修法模板：照抄这些形状，能省掉一轮 review

出现频次（78 个已合并 PR 统计）：NULL-check 33、LM_ERR/WARN 32、length-check 29、goto-error 19、free-on-error 15、`str*`→`memcpy` 12、clamp 8、`sprintf`→`snprintf` 4。

### 先判断：读取类 vs 写入类
- **读取/解析类**（搜索范围错、参数长度校验缺失）→ **改一行即可**，无需错误处理。实证：AlexisHadj `siputils` `#4931`（`+1/−1`，已合并）。
- **写入类**（sprintf/strcpy/memcpy 进缓冲）→ **必须检查返回值并拒绝**，不能只截断。
- ⇒ "改太多要简洁"不是普适的：**写入类不能砍检查，读取类可以。**

### 高频形状
1. **解析/拷贝前的最小长度检查**：先 `LM_ERR` 再 `goto error`（统一错误出口释放资源）。
2. **偏移+长度 vs 边界**：`offset + N + len > total` 一站式判据比分开判断好读；热路径加 `unlikely()`。
3. **安全追加写**：抽 helper，返回 `-1` 而不是截断，`src->len < cap - dst->len` 形式避免整数溢出，负值显式挡（`str.len` 可能为负）。
4. **`strncpy` → `memcpy` + 显式 NUL**（先检查容量）。
5. **rpc 输入**：早检查 + `rpc->fault` 而不是 return 值。
6. **格式串**：变量当 format 一律改成显式 `"%s"`。

### 🚨 sprintf / snprintf 专项（踩坑密度最高）
- ❌ 旧说法"`sprintf` 用 `%.*s` 精度限定通常安全" —— **完全错误**。真相：`%.*s` 的精度只限制**从源读取**的字符数，**完全不限制写入目标缓冲的字节数**。`sprintf(dst, "%.*s ", len, src)` 当 `len > sizeof(dst)-2` 时照样栈溢出（活例：issue #4920，`char callid[50]` + `sprintf(callid, "%.*s ", e->callid.len, e->callid.s)` → 256 字节 Call-ID 写 258 字节进 50 字节缓冲）。
  **正确判据**：比较「目标缓冲 `sizeof(dst)`」vs「源长度上限」，而不是看格式串有没有精度。
- ⚠️ 加固必须**检查返回值**：maintainer（miconda）要的是"拒绝"而非"截断"。`len = snprintf(buf, size, ...); if (len < 0 || (unsigned)len >= size) { LM_ERR(...); return -1; }`。
- ⛔ 改 snprintf 时**必须同时删掉后面的 `dst[len] = '\0';`** —— snprintf 返回"本应写入"长度，截断时 `len > size` → 这行变成一次新的越界写，**等于没修**。
- `len` 后面不用就干脆不赋值（否则 `-Wunused-but-set-variable`）；若删了赋值，声明一起删。

### ⚠️ 固定缓冲拷贝：`>= sizeof(buf)` 是头号陷阱
拷贝进固定缓冲时，NUL 终止符总在"数据最后 1 字节之后再写 1 个字节"，所以判据天然要 **`>=`**，**写成 `>` 会留下恰好 1 字节的越界写**：
```c
/* ❌ 错：len == sizeof(buff) 时 memcpy 刚好填满，buff[len] = 0 越界 1 字节 */
if (vb->host.len > sizeof(buff)) { ... goto error; }
memcpy(&buff, vb->host.s, vb->host.len);
buff[vb->host.len] = 0;

/* ✅ 对 */
if (vb->host.len >= (int)sizeof(buff)) { LM_ERR(...); goto error; }
```
**光看代码看不出来**，必须用 canary 实测（见 §7）。

### 同一函数多个分支的"上界"可能不同
`ims_qos_npn` 的 Via host 拷贝有两个分支，写入上界差 2 字节：IPv4 写 `buff[len]`、IPv6（先去 `[` `]`）写 `buff[len-2]`。**统一 guard 只能按更严的 IPv4 边界取，会误伤合法的 47 字节带括号 IPv6**。⇒ 先逐分支算"最多写到哪个下标"，再决定统一还是分开。**分支内分别检查是多几行但行为保持的修复，比统一 guard 更容易过 review。**

> ⛔ **实测必须"逐字照抄"对方代码**，否则是循环论证：复现程序里代表"对方版本"的变体，必须是从原文逐字复制的。如果按自己理解重写一遍再拿实测去判定对方对错，证明的只是"你实现的版本"。截图里的 `>` 与 `>=` 只差一字符，肉眼极不可靠 —— 涉及判据对错的判断，请对方把代码用**文字**再贴一次。

---

## 6. 静态扫描的两道「否决闸」（否则 90% 是白做功）

正则扫到"危险函数 + 固定数组"只是**初筛**，必须再过两道闸：

**闸 1：是否已有边界检查**（扫描器看不到）。逐个打开函数体肉眼确认。

**闸 2：输入长度是否真的可达缓冲**。三种封顶来源，扫描器一律看不到：
1. **DB schema 的 VARCHAR 长度** —— 查 `utils/kamctl/mysql/<mod>-create.sql`（注意：xcap/rls 等表在 `presence-create.sql` 里）。⚠️ 这个判据是**双向**的：rls `cid[512]` vs `resource_uri VARCHAR(255)` → 封顶 < 缓冲 → 不可达（否决）；xcap `buf[128]` vs `etag VARCHAR(128)` → 封顶 **≥** 缓冲 → **正好可达**（坐实，etag ≥ 112 字节就溢出）。**要比较"封顶值 vs 缓冲大小"，不是看到 VARCHAR 就判死。**
2. **前置校验函数封顶** —— 如无 `is_e164()` 要求 `len < MAX_NUM_LEN` 则不可达。
3. **触发分支是否远端可控** —— 若只在本地生成的 tuple 才走无检查分支，攻击者控制不了，不可达。

**启示**：同一文件"A 函数有检查、B 函数没有"看起来像明显遗漏，但往往 B 靠上游封顶已安全 —— 先过闸再说。

---

## 7. 验证三件套（做边界类漏洞的标准动作）

**① 先拉最新 master 再测**（别在过期代码上验证）：
```bash
git fetch origin master
git diff <本地HEAD>..origin/master -- src/modules/<mod>/   # 空 = 模块未被动
git rev-parse origin/master:src/modules/<mod>/<file>.c      # 与本地 diff 的 index blob 比对
```

**② 单模块真编译**（不用全量 `make all`）：
```bash
make prefix=/usr/local/kamailio-6.1 include_modules='<mod>' cfg
cd src/modules/<mod> && make            # 出 .so 即编译+链接通过
rm -f notify.o save.o && make notify.o save.o 2>&1 | grep -ci warning   # 告警计数
```

**③ fork 子进程跑边界阶梯**（独立镜像程序，如 `pcscf_alias_repro.c`）：
- 每个长度 fork 一个子进程，SIGABRT 只杀子进程，不打断整轮扫描；
- 父进程 **fork 前 `fflush(stdout)`**，子进程用 **`_exit()`**（不能 flush，否则整张表重复输出）；
- 返回值**不要用 exit code 传**（>255 会截断），改用 pipe 写 int；
- 用 `volatile unsigned char guard[32]` 放在目标缓冲后面，检测"静默破坏"档（未触发 canary 时也能证明）。

> 无编译器时的救命招：用 Python `ctypes` 直接调真实 C 运行时验证格式化串行为（Windows `msvcrt`，Linux `libc.so.6`）。⚠️ 两个固定坑：① `create_string_buffer(n)[i]` 返回 `bytes` 不是 int，要用 `ctypes.cast(buf, ctypes.POINTER(c_ubyte))[i]`；② Windows 只有 `msvcrt._snprintf`（截断返回 **-1 且不补 NUL**），模拟 C99 snprintf = `_snprintf(buf,n,...)` + 手动 `P[n-1]=0`。

---

## 8. PR 文案与提交规范

- **标题**：`<module>: Add robust <X> validation` 这类固定句式，让维护者一眼归类（branding 比内容更容易被接受）。
- **描述**：保留官方 PR 模板，不要另起炉灶；**绝不写不存在的链接**（某 PR 因 LLM 编造链接被点名）；引用 issue/commit 用真实编号。
- **AI 参与**：按模板勾选 `LLM/AI Coding Assistants were involved`，但正文要人工写、只写事实（改了什么、为什么）。
- **解法挑最简单的那个**：#4891 改动正确，但 henningw 自提说"*I choose another, simpler approach*"——但**保留 `Co-authored by:` 原作者署名**。简单 + 留名，是双赢。
- **常量提 `#define`**：别在代码里写裸数字。

### 合并时间估算（给自己排期）
- ≤5 行 / 1 文件：中位 17.7h / 120h；6–30 行 145h；2 文件 228h。
- 维护者：linuxmaniac（IMS）30.3h 最快；henningw 139h；miconda 208.6h（写入类他要"检查返回值+拒绝"）。

### 被抢先时的处置
- ❌ 不提重复 PR（必被关，不专业）。
- ✅ 在对方 PR 下贴 **independent verification**：放你有而他 PR 里没有的硬证据（DB schema 封顶值 → 可达性算术、阈值推导、VM 实测数据表）。既符合"不抢功"的社区礼仪，又能让维护者记住你、加速合并。

---

## 9. 同源副本 / 先例未传播：两个最容易漏、也最容易被抢的渠道

**同源副本必查**：一个模块有的逻辑，兄弟模块常有一份复制品。已知配对：`ims_qos` ↔ `ims_qos_npn`、`registrar` ↔ `ims_registrar_scscf`、core `msg_translator.c` ↔ `textops/textops.c`、`dialog` ↔ `ims_dialog`。
> 惨痛教训 #4926：我们 09-09 出好补丁却一直等"发吧"，09-18 被 rizwan3659 的 #4937 拿走。**他赢在覆盖面**：核心思路一致，但他顺手把 `ims_registrar_scscf` 的同源副本也改了，并多挖两个 bug。

**先例传播粒度是「字段级」不是「函数级」**（#4920 后续）：`c6d2630b3`（lrkproxy 修 Call-ID 溢出）引入了 helper `lrkp_build_spaced_field()`，**但只替换了 `callid` 一个字段** —— 同一对函数里 8 个 `char X[20]` + `sprintf("%.*s ")` **原样保留**（×2 = 16 处）。⇒ 拿到"官方修过 X"的先例后，**还要检查同一个函数里其他同形字段有没有一起修**。这类漏网修法最容易写：直接复用作者刚写的 helper，review 成本≈0。

**第三类常被漏掉的目标：自家 helper 写到无容量参数的目标缓冲**。
典型：`int bin_to_base16(char *from, int len, char *to)` —— 只写 `2*len`，函数签名里根本没有目标容量，调用点的 `char out[256]` 溢出在 grep 危险函数列表时永远扫不到。
> 找法：模块内自定义转换函数，看有没有 capacity 形参；再追调用点，比较"输出长度表达式 vs 目标 sizeof"。
> 实例：`ims_isc/mark.c isc_mark_set()` 的 `aor_hex[256]` + `chr_mark[256]`，输入 = SIP 的 P-Asserted-Identity / From / Request-URI → 远端可控 → 真栈溢出（本仓库 `ims_isc-mark-bounds.patch` 即此）。
> ⚠️ 修复顺序：**先卡住 helper 的输入长度，再卡 snprintf** —— helper 的写发生在后者之前。

---

## 10. 已被别人扫过的模块（别撞车）

**AlexisHadj 已占**（22 天 9 个 PR 100% 合并）：seas、sipcapture、userblocklist、sca、ldap、sipt、cdp、siputils、evapi。
**其他已被扫**：ims_registrar_scscf、core `msg_translator.c`、ims_diameter_server/cJSON、core srjson。

⚠️ 曾经误把这份清单当成"维护者已扫过"，其实全是同一个人批量扫。**那是"别人扫过"，不是"官方扫过"，未占模块仍有大片**：pua / rls / presence\* / nat_traversal / enum / dispatcher / auth_web3 / msrp / websocket / tls / http_client / ndb_\* / app_\* / ims_\*。

### 判断模块"冷门不冷门"的硬指标（别凭感觉答）
1. 编译分组：`src/Makefile.groups` 里属于哪个 `mod_list_*`；
2. 被谁依赖：`grep -rn "<mod>" src/modules --include=*.c | grep -v "^src/modules/<mod>"`；
3. 发行版打包：`pkg/kamailio/deb/*/control`、`obs/kamailio.spec`、Alpine APKBUILD 是否含它；
4. 年代/出身：文件头 Copyright 年份 + 公司（老模块 = miconda 自家，他熟）；
5. 维护活跃度：`WebFetch https://github.com/kamailio/kamailio/commits/master/src/modules/<mod>`（比 API 稳，不需要 token）；
6. 🔑 **最强信号**：maintainer 近 12 个月内是否 commit 过同一文件/同一函数。命中即"他有上下文"，补丁零解释成本。

---

## 11. 踩坑清单（每一条都真实发生过）

1. **抢活时效**：小而稳的目标必须「当天定稿 + 当天提交」。被抢 4 次（见 §0），且团队内部撞车也是白做功。
2. **`assignees` 数组必查**：不能只看"有无关联 PR"。assignee 有人 = 视为已被认领，不要抢。
3. **maintainer 已下场 = 先观察**：他在诊断、问澄清时抢 PR 易撞车或被判 as designed。
4. **「开 issue 当天自提 PR」是社区常态**：新 issue 常在 24h 内被作者自己 PR 抢走。
5. **是真 bug 还是用法误解**：先用源码确认行为，符合"按设计如此"就不值得 PR。
6. **深水 vs 软柿子**：动 tm 事务状态机价值高但 review 严；优先"小而稳"。
7. **不要误判"官方已扫过"**：别人批量扫 ≠ 官方扫，空白仍大。
8. **不要凭 curl 的 000 断言网络不通**（见 §1 注）。
9. **「先例未传播」要先验证副本是否真的存在**：不是每个先例都有漏网副本，都修了就立刻放弃，别硬凑。
10. **扫描必须在干净工作区上做**：本地常残留已作废补丁，直接对 `src/` 跑扫描会把"已修好的"当未修。判据：`git hash-object <本地文件>` == `origin/master:<path>` 时才可信。
11. **先例 commit 的传播粒度是字段级**（见 §9）。
12. **报告者已定位到代码行的 issue → 默认报告者即提交者**（见 §2）。

---

## 12. 安全与协作礼仪（红线）

- **任何写 GitHub 远端的副作用**（发 issue/PR 评论、提 PR、push、改描述）**必须先得明确一句话授权**。纯分析、本地草稿、拉取查询、本地编译验证可自由做。
- **不抢功、不抢提**：倾向以 "+1 independent verification" 支持性评论配合上游动作。
- **漏洞披露谨慎**：本仓库最初建为 **private**，正是为避免把上游尚未修复的漏洞细节公开。需要给同事看时，**加协作者（Settings → Collaborators）比直接转 public 更安全**；若已合并或不在意披露，再转 public 不迟。（注：本文档随仓库一同公开时，请确认上游是否已修复。）
- **PAT 安全**：明文 token 不要落进 `.git/config` 或命令行历史；用完到 GitHub Settings → Developer settings 撤销轮换。本仓库曾用一次性带 token 的 URL 推送，随后已改回无 token 形式并 `grep -c ghp_ .git/config` 确认 0 残留。

---

*—— 整理自 2026-08 至 2026-10 的 kamailio 巡查记录与 5 次真实补丁提交（#4921/#4869/#4855/#4836 已合并；ims_isc mark.c 溢出补丁见本仓库 `ims_isc-mark-bounds.patch`）。*
