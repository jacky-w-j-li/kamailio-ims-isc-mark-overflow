# kamailio / ims_isc: `isc_mark_set()` 定长栈缓冲溢出 — 修改建议

- **仓库**：kamailio/kamailio，分支 `master`
- **文件**：`src/modules/ims_isc/mark.c`，函数 `isc_mark_set()`
- **base blob**：`738a37b8d136633f21cbd854c1a5b19218f0315b`（2026-10-07 复核，仍与 `origin/master` 一致）
- **改动规模**：1 个文件，+13 / −2
- **状态**：本地已完成并验证，**尚未提交上游**（等授权）

---

## 1. 摘要

`isc_mark_set()` 把 ISC 标记写进两个 256 字节的定长栈数组，而写入量由
`mark->aor.len` 决定。`mark->aor` 来自 SIP 消息里的 P-Asserted-Identity / From /
Request-URI，**长度由对端控制**。`bin_to_base16()` 没有目标容量参数，恒写
`2 * len` 字节；随后的 `sprintf()` 也没有边界。超长 AoR 会写穿两个栈数组，
之后 `strlen(chr_mark)` 还会越界读。

补丁的思路是**拒绝表达不出来的东西**，而不是截断或扩容。

## 2. 现状代码（`mark.c:296`）

```c
int isc_mark_set(struct sip_msg *msg, isc_match *match, isc_mark *mark)
{
	str route = {0, 0};
	str as = {0, 0};
	char chr_mark[256];
	char aor_hex[256];
	int len;

	/* Drop all the old Header Lump "Route: <as>, <my>" */
	isc_mark_drop_route(msg);

	len = bin_to_base16(mark->aor.s, mark->aor.len, aor_hex);
	/* Create the Marking */
	sprintf(chr_mark, "%s@%.*s;lr;s=%d;h=%d;d=%d;a=%.*s", ISC_MARK_USERNAME,
			isc_my_uri.len, isc_my_uri.s, mark->skip, mark->handling,
			mark->direction, len, aor_hex);
	/* Add it in a lump */
	route.s = chr_mark;
	route.len = strlen(chr_mark);
	...
}
```

### 2.1 两条缺陷链

```
mark->aor（对端可控，长度不限）
   │
   ├─► bin_to_base16() 恒写 2×len ──► aor_hex[256]      溢出阈值：len ≥ 129
   │
   └─► sprintf() 无边界 ──────────► chr_mark[256]       溢出阈值：len ≥ 103（随 isc_my_uri 配置而变）
                                          │
                                          └─► strlen(chr_mark) → 越界读
```

| 位置 | 问题 | 阈值 |
|---|---|---|
| `bin_to_base16(mark->aor.s, mark->aor.len, aor_hex)` | 无容量参数，恒写 `2*len`；`aor_hex` 仅 256 字节 | `len ≥ 129`（硬阈值，与配置无关） |
| `sprintf(chr_mark, ...)` | 无边界，写入量随 `aor.len` 和 `isc_my_uri` 增长 | `len ≥ 103`（实测环境；随 `isc_my_uri` 配置而变） |
| `route.len = strlen(chr_mark)` | `sprintf` 溢出后缓冲末尾无 NUL，`strlen` 一路读到栈上其他内容 | 溢出后必然触发 |

`bin_to_base16()` 定义在同文件 `mark.c:70`：

```c
int bin_to_base16(char *from, int len, char *to)
{
	int i, j;
	for(i = 0, j = 0; i < len; i++, j += 2) {
		to[j] = hexchars[(((unsigned char)from[i]) >> 4) & 0x0F];
		to[j + 1] = hexchars[(((unsigned char)from[i])) & 0x0F];
	}
	return 2 * len;
}
```

### 2.2 可达性

`mark->aor` 的赋值路径（`ims_isc_mod.c`）：

- originating：`s = cscf_get_originating_user(msg, &s)` → `new_mark.aor = s`（`ims_isc_mod.c:390`）
- terminating：`cscf_get_terminating_user()` → `new_mark.aor = s`（`ims_isc_mod.c:432`）

即 P-Asserted-Identity / From / Request-URI，**一个 INVITE 就能带进来**，长度只受
SIP 消息大小限制。

## 3. 补丁

```diff
diff --git a/src/modules/ims_isc/mark.c b/src/modules/ims_isc/mark.c
index 738a37b8d1..b9ecc7d123 100644
--- a/src/modules/ims_isc/mark.c
+++ b/src/modules/ims_isc/mark.c
@@ -304,14 +304,25 @@ int isc_mark_set(struct sip_msg *msg, isc_match *match, isc_mark *mark)
 	/* Drop all the old Header Lump "Route: <as>, <my>" */
 	isc_mark_drop_route(msg);
 
+	if(mark->aor.len < 0 || mark->aor.len > (int)sizeof(aor_hex) / 2) {
+		LM_ERR("invalid length for the address of the mark (%d)\n",
+				mark->aor.len);
+		return 0;
+	}
+
 	len = bin_to_base16(mark->aor.s, mark->aor.len, aor_hex);
 	/* Create the Marking */
-	sprintf(chr_mark, "%s@%.*s;lr;s=%d;h=%d;d=%d;a=%.*s", ISC_MARK_USERNAME,
+	len = snprintf(chr_mark, sizeof(chr_mark),
+			"%s@%.*s;lr;s=%d;h=%d;d=%d;a=%.*s", ISC_MARK_USERNAME,
 			isc_my_uri.len, isc_my_uri.s, mark->skip, mark->handling,
 			mark->direction, len, aor_hex);
+	if(len < 0 || len >= (int)sizeof(chr_mark)) {
+		LM_ERR("the mark to add is too long (%d)\n", len);
+		return 0;
+	}
 	/* Add it in a lump */
 	route.s = chr_mark;
-	route.len = strlen(chr_mark);
+	route.len = len;
 	if(match)
 		as = match->server_name;
 	isc_mark_write_route(msg, &as, &route);
```

同一份补丁也存为 `ims_isc-mark-bounds.patch`，可直接 `git apply`。

## 4. 逐条设计说明

### 4.1 闸门 1：`aor.len` 检查

```c
if(mark->aor.len < 0 || mark->aor.len > (int)sizeof(aor_hex) / 2) {
```

- `aor_hex` 是 256 字节，`bin_to_base16()` 写 `2*len` ⇒ 安全条件是 `len ≤ 128`。
- 写成 `sizeof(aor_hex) / 2` 而不是硬编码 `128`：以后有人改数组大小，检查自动跟随。
- `aor_hex` **不需要 NUL 终止**——`bin_to_base16()` 只写 `2*len`，后面用 `%.*s`
  按 `len`（返回值 = `2 * aor.len`）读出来，所以 `len ≤ 128` 时内容完整且不越界。
- `len < 0` 一并挡掉：`str.len` 理论上非负，但这个检查成本为零。

### 4.2 闸门 2：`sprintf` → `snprintf` + 校验返回值

**只换成 `snprintf` 是不够的**，这是本补丁最容易漏的一点：

`snprintf()` 会静默截断。截断后 `strlen(chr_mark)` 返回一个"看起来合法"的
255，于是发出一个**被截断的、语义错误的 Route 头**——ISC 标记错、AS 路由错，
而日志里什么都没有。所以必须校验返回值：`snprintf()` 返回的是**本应写入的长度**
（不含结尾 NUL），`>= sizeof(chr_mark)` 就说明被截断了，必须拒绝。

判据用 `>=` 而不是 `>`：返回值不含 NUL，NUL 还要占 1 字节，所以 `== sizeof`
就已经越界。

### 4.3 闸门 3：`route.len = len`

这不是可有可无的清理：

1. `sprintf` 一旦溢出，`chr_mark` 末尾没有 NUL，`strlen()` 会一路读到栈上别的
   内容——这是**越界读**，读出的长度还可能大得离谱，导致 lump 写入超长。
2. 此时 `len` 已被闸门 2 保证准确，直接用它最省事，还省掉一次 O(n) 扫描。

## 5. 两个已知取舍（评审时请重点看这两条）

### 5.1 是 fail-open，不是 fail-closed

`isc_forward()`（`isc.c:70`）这样调用：

```c
isc_mark_set(msg, m, mark);   /* 返回值被忽略 */
```

所以补丁里 `return 0` 的实际效果是：**不写 Route 头，但消息仍旧被转发到 AS**
（`msg->dst_uri` 已经设成 `m->server_name`）。属于安全降级——丢掉 mark，后续
不会再触发下一个 iFC——但不是"拒绝转发"。

要变成 fail-closed，得把返回值检查传进 `isc_forward()` 再传到 `ims_isc_mod.c`，
改动变成 2-3 个文件。按上游经验，1 个文件的 PR 中位 120h 合并，2 个文件 228h+。
**建议本次只做 1 个文件**，fail-closed 留作 follow-up。PR 描述里用
"rejects what cannot be represented" 表述，配合 `LM_ERR` 留痕。

### 5.2 检查放在 `isc_mark_drop_route()` 之后

失败时会先删掉旧的 Route lump。移到 drop 之前更干净，行数不变，但会改动已经
实测过的定稿。默认保持原样；如果评审认为挪到前面更好，改动成本很低。

### 5.3 为什么是"拒绝"而不是"扩容"

`aor_hex` 按最坏情况扩容要跟着 AoR 上限走（SIP 消息可达 64KB），那就不该放在
栈上，改动性质会从"加固"变成"重构"。拒绝一个表达不出来的超长 AoR 是正确行为——
它本来就不可能是一个合法的用户标识。

## 6. 验证记录

| 项目 | 结果 |
|---|---|
| clang-format 18.1.8 | CLEAN |
| 基线编译（VM，gcc 15.2，导出 master 上本模块全部 14 个源文件） | `LD ims_isc.so`，EXIT=0 |
| 打补丁后编译 | `LD ims_isc.so`，**warning = 0** |
| 运行时边界实测（把 `isc_mark_set()` 逻辑抽成独立程序） | `aor.len=102` 正常（mark 长 255）<br>`aor.len=103` 需 257 字节 → 拒绝（补丁前：栈破坏）<br>`aor.len=128` `aor_hex` 正好填满、mark 过长 → 拒绝<br>`aor.len=200` 补丁前 SIGABRT |
| base 一致性 | `origin/master` 上 `mark.c` blob = `738a37b8d136`，等于补丁 base；近期触及 ims_isc 的只有 `12c61cc397`（改的是 `ims_isc_mod.c`，**未碰 mark.c**） |

对 `aor.len <= 102` 的行为没有任何改变。

## 7. 上游提交计划

因所用 PAT 无 `workflow` scope，`git push` 会被拒，走 GitHub Git Data API：

1. 取 fork `jacky-w-j-li/kamailio` 的 master sha 与 tree
2. `POST /git/blobs`（新的 `mark.c` 内容）
3. `POST /git/trees`（`base_tree` = fork master tree，只替换 `src/modules/ims_isc/mark.c`）
4. `POST /git/commits`（parents 必须是 **fork 自己的 master sha**，用 upstream sha 会建出悬空 commit）
5. `POST /git/refs` → `refs/heads/ims_isc-mark-bounds`
6. `POST /repos/kamailio/kamailio/pulls`（head = `jacky-w-j-li:ims_isc-mark-bounds`，base = `master`）

提交前已确认 fork 状态：`compare` = `behind / ahead_by=0 / behind_by=435`
（纯落后无分叉 ⇒ PR diff 只有自己的 +13/−2），fork 上 `mark.c` 的 blob 同样等于 base。

建议的 commit subject / PR 标题：

```
ims_isc: sanity checks for building the isc mark
```

句式对齐 linuxmaniac 在邻居模块的 commit
`ims_dialog: sanity checks for parse_dlg_rr_param`——他是 IMS 方向的把关人，
用他的措辞能降低归类成本。

不提交复现程序（kamailio 没有放复现程序的目录惯例），实测数据写进 PR 描述，
保持 **1 个文件**。
