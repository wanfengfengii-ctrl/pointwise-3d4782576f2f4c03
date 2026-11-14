# 快照元数据局部更新设计（DESIGN）

## 0. 结论与范围

本轮只交付本文档，不实现目标结构，也不修改 `lake.py`、`catalog.py`、`objects.py`、`checks.py`、`replay.py`、`samples.py`、`observe.py` 与样例数据。目标是把删除一个稀疏位置、重写少量文件所需的元数据访问从“复制该快照的全部文件与墓碑”降为“访问并复制该操作相关索引路径上的有界节点”，同时保留：

- `Lake` 对外接口、返回值、错误优先级、快照编号规则；
- `prepare`/`publish` 之间的 staged 发布冲突条件（`publish` 时 head 必须仍等于 plan.base）；
- 历史快照永久可读，且读取不逐代重放、不扫描全部历史再按 sid 筛选。

推荐方案是**写时复制（copy-on-write）的不可变持久索引树**：每个快照只保存文件索引根和墓碑索引根两个标识；索引节点内容寻址、不可变，提交时只复制本次变化经过的路径。它与现状“每快照全量物化”策略在 §2 中对比，被否决的第三种策略见 §2.3。

## 1. 现有调用链与基线实测

### 1.1 Lake → Catalog → Objects 调用链

| 环节 | 位置 | 行为 |
| --- | --- | --- |
| 构造 | `lake.py:5-8` | `Lake` 持有 `Catalog` 与 `Objects`；默认内存适配，磁盘适配由调用者传入路径 |
| 装载 | `lake.py:10-16` | 行载荷先落到 `Objects`（`lake.py:11-12`），写事务内检查空库（`lake.py:14`）后一次 `commit(set(files), set())`（`lake.py:16`）建立快照 1 |
| 读 | `lake.py:18-25` | `catalog.state(snapshot)`（`lake.py:19`）一次性物化该快照的 `files`/`dead` 集合；按文件名序逐文件 `objects.get`（`lake.py:21-23`），逐行排除墓碑（`lake.py:23`） |
| 删除 | `lake.py:27-37` | 先 `state(snapshot)`（`lake.py:28`）定位源快照；`lake.py:29` 按“文件是否在源、`type(pos)` 是否为 `int`、位置是否越界（越界上界取 `len(objects.get(fid))`）”的短路顺序报 `BAD_POSITION`；进入 `catalog.write()`（`lake.py:31`）后重新 `state()`（`lake.py:32`），当前文件不存在报 `STALE_LAYOUT`（`lake.py:33`），墓碑已存在则直接返回 head（`lake.py:34-36`），否则 `commit(current, dead ∪ {..})`（`lake.py:37`） |
| 重写准备 | `lake.py:39-54` | `state()`（`lake.py:40`）取 base 与当前集合；`lake.py:41` 校验输入非空、不重复且属于当前快照，否则 `BAD_INPUTS`；`lake.py:43` 读全部输入载荷；`lake.py:44-46` 校验 layout 恰好覆盖全部物理位置（含已删位置），否则 `BAD_LAYOUT`；`lake.py:48-53` 按组写出 `rewrite-JOB-i`，写出时跳过墓碑（`lake.py:51`） |
| 发布 | `lake.py:56-66` | 写事务内 `state()`（`lake.py:58`），`lake.py:59` head ≠ base 报 `PUBLISH_CONFLICT`（先于计划合法性检查）；`lake.py:61-63` 输入不在当前集合报 `BAD_PLAN`；`lake.py:64-66` 以“当前集合减输入加输出”和“墓碑中剔除输入”提交 |
| 封装 | `lake.py:68-71` | `rewrite` 在同一个 `catalog.write()` 内调用 `prepare` 与 `publish` |
| 事务 | `catalog.py:22-34` | 最外层 `BEGIN IMMEDIATE`（`catalog.py:26`）；嵌套调用通过 `in_transaction` 加入外层（`catalog.py:23`）；最外层正常 `COMMIT`（`catalog.py:30`），异常 `ROLLBACK`（`catalog.py:32-33`） |
| head | `catalog.py:36-37` | `head` 是一行单例表 |
| 状态物化 | `catalog.py:39-46` | `sid=None` 时读一次 head（`catalog.py:40`）；`snapshots` 无该 sid 报 `NO_SNAPSHOT`（`catalog.py:41`）；随后读该 sid 的**全部** `files` 行（`catalog.py:43`）与**全部** `tombstones` 行（`catalog.py:44-45`） |
| 提交 | `catalog.py:48-58` | 必须有写事务（`catalog.py:49`）；`sid = head+1`（`catalog.py:50-51`）；插入快照行（`catalog.py:53`）；**逐代插入新快照的全量文件集合**（`catalog.py:54`）和**全量墓碑集合**（`catalog.py:55-56`）；最后更新 head（`catalog.py:57`）——这是提交的线性化点 |
| 载荷 | `objects.py:13-28` | 行内容规范化 JSON（`objects.py:16`），文件名永久唯一（`objects.py:14`、磁盘 `xb` 于 `objects.py:20`），同名不同内容报 `IMMUTABLE_CONFLICT`（`objects.py:23-24`）；读取整体 JSON（`objects.py:30-32`） |

现状的结构性成本来自两处：

1. `catalog.state()`（`catalog.py:43-45`）对**任何**调用都读取该版本全部文件行与全部墓碑行。`lake.delete` 即使只删一个位置，也会先在源快照和当前快照各物化一次（`lake.py:28`、`lake.py:32`）。
2. `Catalog.commit`（`catalog.py:53-57`）**每个新快照都重写整个文件集合和整个墓碑集合**，不是增量。

因此删除单位置的写代价是 O(F′)，重写发布写代价是 O(F′+D′)。读一个指定历史快照的代价是 O(F_sid + D_sid)，虽然不随 H 增长，也不是逐代重放；问题只在写侧和存储侧。

### 1.2 基线实测（`python3 observe.py demo`，当前实现）

`cost.read`/`cost.write` 是观察器统计的 SQL 游标交给调用者的行数与 DML 实际影响行数，**不是**页 I/O、虚拟机访问或耗时，也不是设计方案的计费单位。下表只是现状事实：

| 样例/步骤 | 操作 | read(snapshots/files/tombstones/head) | write(snapshots/files/tombstones/head) | 结果 |
| --- | --- | --- | --- | --- |
| sparse 0 | delete f00:0 | 2/16/0/2 | 1/8/1/1 | head=2，dead={f00:0} |
| sparse 1 | delete f01:0 | 2/16/1/2 | 1/8/2/1 | head=3，dead 两条 |
| sparse 2 | 重复删除 f00:0 | 2/16/2/1 | 0/0/0/0 | 返回 head=3，不新增快照 |
| sparse 3 | rollback_delete f02:0 | 2/16/2/2 | 1/8/3/1（执行后被回滚） | head 仍为 3 |
| sparse 4-6 | read 1/2/3 | 各 1/8/{0,1,2}/0 | 0 | 16/15/14 行 |
| sparse 7 | pos=99 | 1/8/0/0 | 0 | `BAD_POSITION`，未进写事务 |
| local 0 | delete f00:0 | 2/16/0/2 | 1/8/1/1 | head=2 |
| local 1 | prepare | 1/8/1/1 | 0 | base=2，输出两个文件 |
| local 2 | publish | 1/8/1/2 | 1/9/0/1 | head=3，f00 墓碑被移除 |
| local 3 | prepare 空 layout | 1/9/0/1 | 0 | base=3，输出为空 |
| local 4 | publish 零输出 | 1/9/0/2 | 1/8/0/1 | head=4，零行输入被移除 |
| local 5-8 | read 1/2/3/4 | 各 1/{8,8,9,8}/{0,1,0,0}/0 | 0 | 16/15/15/15 行 |

这些数字与 `observe.py:140-160` 的断言一致。它们清楚地显示：**一次单点删除写了 8 条 files 行**，而其中只有墓碑集合真正变化；重复删除已经不再写，但两次源快照/当前快照的读取仍各物化 8 个文件。

## 2. 存储策略比较

### 2.1 策略 A：每快照全量物化（现状）

- 结构：`files(sid,fid)`、`tombstones(sid,fid,pos)`（`catalog.py:11-16`），版本即行集合；`snapshots` 只记 parent（`catalog.py:10`）。
- 版本定位：`(sid,fid)` 主键或 `sid` 过滤；读一个版本直接扫该 sid 的行，因此历史读不重放。
- 原子提交：同一事务插入新 sid 的全量行并更新 head（`catalog.py:53-57`）。
- 成本（F、D 为当前规模，F_sid、D_sid 为目标版本规模，H 为历史数）：
  - 删除：读 O(F_sid + D_sid) + O(F_head + D_head)；写 O(F′ + D′)。
  - prepare：读 O(F_head + D_head)；写 0。
  - publish：读 O(F_head + D_head)；写 O(F′ + D′ + m)。
  - 读：O(F_sid + D_sid)。
  - 存储：O(H·(F+D)) 行，逐代重复。
- 满足历史读与错误语义，但明确违反“单点删除不得复制整份文件集合/全体墓碑”和“重写仅访相关元数据”。

### 2.2 策略 B：写时复制不可变持久索引树（选定）

- 结构：每个快照行保存 `files_root` 与 `tomb_root`；节点不可变、内容寻址，节点表只追加。
- 版本定位：`sid` 主键点查快照行一次，直接得到该版本两个根；不沿 `parent` 链重放。
- 原子提交：新节点 + 新快照行 + head 更新在同一事务内；根标识写入快照行，head 是可见性开关。
- 成本：删除 O((h+g)·C) 条目读写，publish O(((k+m)h + (k+d)g)·C)，读 O(F_sid + D_sid + (h+g)·C)，存储增量按变化量累加而不是按 H·F。
- 满足全部边界；§3 展开。

### 2.3 策略 C：每文件版本链加全局目录（否决）

按文件维护“(sid → 行版本)”链，删除只追加一条差分，看似 O(1) 写。但要回答 `read(snapshot)` 仍需要知道**该版本有哪些文件**：要么每个快照复制一份全局文件名目录（退化为策略 A，单点删除复制 F 个键），要么维护一棵持久目录树（就是策略 B 的文件索引）。墓碑的“某文件在某位置是否已删”还需要按 `(fid,pos)` 查找；若只留差分链，必须扫描该文件全部历史差分或额外建持久索引（又回到 B）。因此 C 在“保留历史读且不重放”的约束下无法同时满足写边界，不作为候选。

### 2.4 比较表

| 维度 | A 全量物化 | B 持久索引（选定） |
| --- | --- | --- |
| 单点删除写 | O(F′+D′) 行，复制全体文件键 | O(g) 个墓碑节点、O(C·g) 条目；文件根不变 |
| 重复删除 | 读两次全量集合 | 源/当前文件点查 + 墓碑点查；0 写 |
| prepare | 读全量 files+dead | k 个文件点查 + 相关墓碑前缀扫描 |
| publish | 写 O(F′+D′+m) 行 | O((k+m)h + (k+d)g) 个节点/条目 |
| 历史读 | O(F_sid+D_sid)，不重放 | O(F_sid+D_sid+C(h+g))，不重放 |
| 额外存储 | O(H·(F+D)) | O(F + C·Σ变化路径)，节点按内容去重 |
| 大量无关文件 | 每次写都复制它们 | 路径外子树共享，不访问 |
| 大量历史 | 每代重复行 | 每代仅新增 touched 路径 + 1 快照行 |

## 3. 推荐结构 B

### 3.1 物理模式（示意，非本轮实现）

```sql
-- 快照行：版本定位的全部信息；parent 仅供审计，读取从不跟随
snapshots(sid INTEGER PRIMARY KEY, parent INTEGER,
          files_root TEXT, tomb_root TEXT);

-- 不可变索引节点：node_id 为内容哈希，同一内容天然去重
index_nodes(node_id TEXT PRIMARY KEY,
            kind INTEGER,          -- 0=文件索引, 1=墓碑索引
            level INTEGER,         -- 0=叶
            entries TEXT);         -- 规范 JSON 数组，见 §3.2

head(singleton INTEGER PRIMARY KEY CHECK(singleton=1), sid INTEGER);
```

`head` 仍是一行单例（沿用 `catalog.py:17`、`catalog.py:36-37`）；`snapshots(sid)` 仍是主键（沿用 `catalog.py:10`）。快照 0 的空行在初始化时写入（沿用 `catalog.py:18`），其两个根指向同一个空叶节点。

**替换端口而非改写业务语义**：`lake.py` 的公开方法和错误顺序不变；只把 `Lake` 内部对 `catalog.state()/commit()` 的调用改接到新的 Catalog 端口（§3.4）。`Catalog.head()` 和一次性物化的 `Catalog.state()` 可保留为兼容适配器，供回归、回放与观察器的观察钩子使用（`replay.py:24`、`observe.py:84` 的调用不进入被测成本）；生产热路径不再走全量 `state()`。

### 3.2 键编码、节点与容量 C

- 文件索引键为 `fid`（字符串，按字节序/Unicode 码点序，与 `lake.read` 的 `sorted(files)` 及 `catalog.commit` 的 `sorted(files)` 一致，`catalog.py:54`）。
- 墓碑索引键为 `(fid, pos)`，先按 `fid` 再按数值 `pos`，与 `sorted(dead)` 一致（`catalog.py:56`、`lake.py:19`）。叶条目保存 `[fid,pos]` 对，便于按 `fid` 前缀范围删除。
- 节点容量 **C**（本样例取 C=4）：每个叶最多 C 个键；每个内部节点最多 C 个分隔键和 C+1 个子指针。因此单个节点携带的条目上界为：叶 ≤ C，内部 ≤ 2C+1（键+指针都算条目）。**节点容量计入成本**：复制一个只改一个键的叶也要付最多 C 条目的读写；路径复制也要付内部节点的分隔键与指针。
- 树高：`h = ⌈log_{C+1} F⌉ + 1` 层（根为第 1 层），`g = ⌈log_{C+1} D⌉ + 1`。样例 F=8/9、D≤2、C=4 时 h=g=2。一次文件点查最坏访问 1 个根 + 1 个叶，≤9+4=13 条目；墓碑点查同理。
- 节点 ID：`H(kind || level || canonical(entries))`。内容相同即复用；不可变保证任何已发布快照引用的节点永不被原地修改。
- 构造形状（供证据计数复核）：`seed` 按 C=4 批量装填，F=8 时为叶 `[f00..f03]`、`[f04..f07]` 加根（1 个分隔键 + 2 个子指针），合计 3 个节点、11 条目；F=9 时加一个 1 键叶，合计 4 个节点、14 条目。一次文件点查在这两个形状下读 root 与目标叶，实际 7–9 条目，上界仍按 13 条目。
- 更新次序（供证据复现）：同一 `commit_roots` 内先删除、再按 `sorted(fid)` 插入；同一叶的多处编辑聚合为一次新节点写入，每个新节点只写一次。因此 publish 的节点数由本次触及的必要路径决定，而不是由 F 决定。

### 3.3 版本定位与历史读

```
locate(sid):
    row = SELECT files_root, tomb_root FROM snapshots WHERE sid = :sid
    if row is NULL: raise NO_SNAPSHOT
    return SnapshotView(sid, row.files_root, row.tomb_root)
```

- 默认读取：`Lake.read(snapshot=None)` 先点查一次 `head`（`SELECT sid FROM head WHERE singleton=1`，沿用 `catalog.py:36-37`），固定该 sid；之后只使用该 sid 的根。这保持“默认固定一次 head”的原语义。
- 显式历史读取：直接点查 `snapshots` 主键，绝不跟随 `parent`（`parent` 列仅保留审计/未来调试用途）。
- 读取过程：按 `files_root` 中序遍历得到该版本全部文件键（有序，天然满足 `lake.read` 的排序）；按 `tomb_root` 中序遍历得到该版本全部墓碑。两者都是有序序列，读取时把文件键与墓碑键按键序合并推进，因此每个墓碑只需在其所属文件输出处比较一次，不需要对每个文件单独点查。每个文件用 `Objects.get(fid)`（`objects.py:30-32`）读载荷，逐位置过滤。
- 因此定位 + 遍历的成本只依赖目标版本实际包含的 `F_sid`、`D_sid` 与路径高度，不依赖历史代数 H；不重放差分，不扫描全历史再筛 sid。

### 3.4 替换端口（Catalog 内部接口）

这些端口是设计层接口，不是本轮要写出的运行代码：

| 端口 | 语义 | 对应现状位置 |
| --- | --- | --- |
| `head()` | 单行点查 | `catalog.py:36-37` |
| `locate(sid=None)` | 点查快照行；`None` 先固定 head；缺行报 `NO_SNAPSHOT` | `catalog.py:40-41` |
| `has_file(view, fid)` | 文件树点查，只读路径节点 | `lake.py:29`、`lake.py:33`、`lake.py:41`、`lake.py:62` |
| `file_keys(view)` | 文件树中序遍历；只产出该版本文件键 | `lake.py:21`、`lake.py:44` |
| `tomb_hit(view, fid, pos)` | 墓碑树 `(fid,pos)` 点查 | `lake.py:35` |
| `tomb_range(view, fid)` | 墓碑树 `[fid,−∞)` 到 `(fid,+∞)` 范围扫描，只读该文件条目 | `lake.py:51`、`lake.py:66` |
| `tomb_keys(view)` | 墓碑树中序遍历 | `lake.py:22-23` |
| `commit_roots(base_sid, file_ops, tomb_ops)` | 在写事务内由当前 head 根做路径复制，生成新根、新快照行并更新 head | `catalog.py:48-58` |

`lake.py` 的公开签名与返回结构完全不变；替换的是内部取值方式。`state()` 仍可保留为“用 `file_keys`/`tomb_keys` 物化集合”的适配器（其成本 >0，但只供观察器/回放等非热路径调用）。

### 3.5 热路径改造与错误顺序（必须逐条保持）

#### `Lake.read(snapshot=None)`（`lake.py:18-25`）

1. `view = catalog.locate(snapshot)`；缺行仍是 `NO_SNAPSHOT`（`catalog.py:41`）。
2. 若 `snapshot=None`，`locate` 内只点查一次 head；之后固定 view。
3. 按 `file_keys(view)` 得到有序文件键、按 `tomb_keys(view)` 得到有序墓碑键，两者按键序合并：对每个 fid 先 `payload = objects.get(fid)`（`objects.py:30-32`），再对该 fid 的墓碑位置跳过对应 `pos`。
4. 输出仍为 `{"snapshot": sid, "rows": [...]}`，排序不变。

访问：1 行快照/head + 文件树 F_sid 条目 + 墓碑树 D_sid 条目 + 两次中序遍历的路径节点 + F_sid 次载荷读取。**不是逐代重放，也不是扫描全历史**。

#### `Lake.delete(snapshot, fid, pos)`（`lake.py:27-37`）

1. `src = locate(snapshot)`；缺快照 `NO_SNAPSHOT`（先于其他检查，`catalog.py:41`）。该次定位固定读 1 行 `snapshots`。
2. 条件顺序完全照搬 `lake.py:29` 的短路：
   - `has_file(src, fid)` 为假 → `BAD_POSITION`（文件树点查，不读载荷）；
   - `type(pos) is not int` → `BAD_POSITION`（不读载荷）；
   - 负数 → `BAD_POSITION`（不读载荷）；
   - 非负整数 → `len(objects.get(fid))`（`objects.py:30-32`，载荷读取是业务必需的，契约允许）判断上界，越界 → `BAD_POSITION`。
3. 进入 `catalog.write()`（`catalog.py:22-34`）；事务内 `locate(None)` 点查 head 1 行与当前快照 1 行，取当前根。
4. `has_file(view, fid)` 为假 → `STALE_LAYOUT`（`lake.py:33`）；这与 `BAD_POSITION` 的相对优先级不变：`BAD_POSITION` 在事务外先报。
5. `tomb_hit(view, fid, pos)` 为真 → 返回当前 `sid`（`lake.py:34-36`），不插入快照、不做任何 DML。
6. 否则 `commit_roots(base_sid, file_ops=∅, tomb_ops=+(fid,pos))`：文件根**原值保留**，墓碑树路径复制、插入一个键，新快照行、更新 head。

单点删除的元数据读取 = 快照点查 2 行（源 1 行 + 当前 1 行；默认快照时另加 head 点查 1 行）+ 2 次文件点查 O(C·h) + 1 次墓碑点查 O(C·g)；写入 = O(g) 个墓碑节点（≤C 条目/节点）+ 1 快照行 + 1 head 行。**不枚举或复制其余 F−1 个文件键，也不枚举全体 D 个墓碑。**

#### `Lake.prepare(job, inputs, layout)`（`lake.py:39-54`）

1. `view = locate(None)`：head 点查 1 行 + 快照行 1 行，得到 base 与当前根。
2. `lake.py:41` 判定：输入列表非空、无重复、且每个输入 `has_file(view, fid)`——k 次文件树点查。任何不满足 → `BAD_INPUTS`。
3. 通过后读取载荷 `payload = {fid: objects.get(fid)}`（`lake.py:43`）；这是载荷访问，不优化。
4. layout 校验保持 `lake.py:44-46` 的顺序与集合语义：重复引用或未恰好覆盖全部物理位置（**包括已删位置**，上界来自载荷长度）→ `BAD_LAYOUT`。
5. 输出 `rewrite-JOB-i`（`lake.py:50`）逐组写出：`tomb_range(view, f)` 只取各输入文件的墓碑条目（总 d 条），按 `lake.py:51` 过滤；`objects.put` 写载荷（`objects.py:13-28`）。
6. 返回 `{"base": head_sid, "inputs": sorted(inputs), "outputs": [...]}`（`lake.py:54`）。

prepare 不在写事务中，元数据 DML=0；读取 = 1 head 行 + 1 快照行 + O(k·h) 文件节点/条目 + O(k·g + d) 墓碑节点/条目。只访问输入相关路径，不读无关文件、不读全体墓碑。

#### `Lake.publish(plan)`（`lake.py:56-66`）

1. `catalog.write()` 开始 `BEGIN IMMEDIATE`（`catalog.py:26`；嵌套加入外层，`catalog.py:23`）。
2. `view = locate(None)`：head 点查。若 `view.sid != plan["base"]` → `PUBLISH_CONFLICT`（`lake.py:59`，先于计划合法性）。
3. `inputs = set(plan["inputs"])`；对每个输入 `has_file(view, fid)`，不满足 → `BAD_PLAN`（`lake.py:62-63`）。注意现状不校验输出、不重新解释 layout、不防计划被篡改，这些行为保持。
4. `commit_roots(base_sid, file_ops={remove: inputs, add: plan["outputs"]}, tomb_ops={remove_prefixes: inputs})`：
   - 文件树：删除 k 个输入键，插入 m 个输出键；**零行输出也作为文件键存在**（`lake.py:50-52` 无条件 `objects.put`）。
   - 墓碑树：按 fid 前缀范围删除输入对应的全部墓碑（含输入已删位置）；与 `lake.py:66` 的 `{(f,p) if f not in inputs}` 等价。
   - 新根写入快照行；head 更新是线性化点。
5. 返回新 sid。

#### `Lake.rewrite(job, inputs, layout)`（`lake.py:68-71`）

外层 `catalog.write()` 包住 `prepare` 和 `publish`；嵌套调用不另起事务（`catalog.py:23`），锁范围与现状一致，本题不优化。由于同一事务内 head 不变，`rewrite` 不会在“自己内部”产生 `PUBLISH_CONFLICT`；冲突仍留给分离的 prepare/publish 调用（`replay.py` 的 between 场景）。

### 3.6 提交、回滚伪代码（具体到查询条件与修改记录）

```
# 前置：Catalog.write()（catalog.py:22-34）已执行 BEGIN IMMEDIATE（catalog.py:26）
# 若存在外层事务（catalog.py:23 的 nested=True），则不在此处开始/提交，由最外层负责
def commit_roots(base_sid, file_ops, tomb_ops):
    if not db.in_transaction: raise WRITE_TRANSACTION_REQUIRED   # 同 catalog.py:49

    parent = SELECT sid FROM head WHERE singleton=1              # 同 catalog.py:50
    if parent != base_sid: raise PUBLISH_CONFLICT                # Lake 已先检查（lake.py:59）
    new_sid = parent + 1                                         # 同 catalog.py:50-51

    old = SELECT files_root, tomb_root FROM snapshots WHERE sid=parent
    froot = old.files_root
    troot = old.tomb_root

    # --- 在内存中构造新路径；尚不写库 ---
    # 文件树：先删后插；每个被触及的叶聚合编辑，每个新节点只生成一次
    for fid in file_ops.remove:
        assert has_file_in(froot, fid)          # 调用者已校验，防御性点查
        froot = path_delete(froot, fid)         # 下溢时合并/借键，仅触及同父兄弟，≤O(h) 节点
    for fid in file_ops.add:
        # 键只存 fid，不存行数；零行文件同样插入
        froot = path_insert(froot, fid)         # 未涉及子树按 node_id 共享

    # 墓碑树：先按 fid 前缀范围删，再点插入
    for fid in tomb_ops.remove_prefixes:
        troot = path_delete_range(troot, (fid,MIN_POS), (fid,MAX_POS))
    for (fid,pos) in tomb_ops.add:
        troot = path_insert(troot, (fid,pos))

    # --- 同一事务内写入：只写本次新建节点 ---
    # node_id = H(kind, level, canonical(entries)); 同内容 INSERT OR IGNORE 复用即可
    INSERT OR IGNORE INTO index_nodes(node_id,kind,level,entries)
        VALUES ...                               # 仅本次创建的节点，≤O((k+m)h+(k+d)g) 行

    INSERT INTO snapshots(sid, parent, files_root, tomb_root)
        VALUES(new_sid, parent, froot, troot)   # 同 catalog.py:53
    UPDATE head SET sid = new_sid WHERE singleton=1   # 同 catalog.py:57，线性化点

# 最外层：正常路径 COMMIT（catalog.py:30）
# 异常路径：任何异常逃出最外层时 ROLLBACK（catalog.py:32-33），
#           撤销上面的节点行/快照行/head 更新；已提交快照引用的旧节点不受影响
```

- **查询条件**：`snapshots` 按 `sid` 主键点查；`index_nodes` 按 `node_id` 主键点查/插入；`head` 按 `singleton=1` 单行。
- **修改记录**：只插入本次路径新建的节点行、1 行新快照、更新 1 行 head；绝不插入整份文件键清单或整份墓碑清单。
- **根绑定**：`snapshots(sid)` 行是版本与两个根的绑定；新根字符串与新旧节点在同事务内写入，head 更新后才对默认读者可见。
- **提交/回滚**：`COMMIT` 由 `Catalog.write()` 最外层执行（`catalog.py:30`）；异常时最外层 `ROLLBACK`（`catalog.py:32-33`），新节点行、新快照行与 head 更新一起撤销。已提交快照引用的旧节点不被修改，所以历史读取不受影响。
- `base_sid` 绑定的是调用者读到的 head；`publish` 在事务内重新点查 head 并比较，保持 staged 冲突条件。若两个写者交错，SQLite `BEGIN IMMEDIATE` 使第二个写者在事务开始处等待或失败，不会半发布。

## 4. 正确性与边界条件

### 4.1 错误优先级

保持现状调用顺序，逐项对应：

- `delete`：`NO_SNAPSHOT`（`catalog.py:40-41`）→ `BAD_POSITION`（`lake.py:29`，短路顺序成员/类型/负数/上界）→ `STALE_LAYOUT`（`lake.py:33`）→ 重复墓碑返回旧 head（`lake.py:34-36`）→ 提交。新端口不改变任何一个判断的先后，也不在 `BAD_POSITION` 之前扫描无关文件。
- `prepare`：`NO_SNAPSHOT`（若 head 行异常）→ `BAD_INPUTS`（`lake.py:41`）→ `BAD_LAYOUT`（`lake.py:46`）。输入合法后才读载荷；layout 合法后才写输出。
- `publish`：`PUBLISH_CONFLICT`（`lake.py:59`）→ `BAD_PLAN`（`lake.py:63`）→ 提交。冲突检查在计划合法性之前，不能因为先查输入而改变错误。
- `read`：显式不存在的 sid → `NO_SNAPSHOT`（`catalog.py:41`），与现状一致。

### 4.2 重复删除与同文件多次稀疏删除

`tomb_hit(view,fid,pos)` 在墓碑树做一次点查；命中即返回当前 head（`lake.py:34-36`）。新方案不创建新快照、不写节点、不更新 head；读取成本是两次文件路径点查加一次墓碑点查，而不是两次全量集合物化。证据 `sparse` 第 2 步的 predicted 与 actual 都有 `write` 全 0、head 保持 3。

同一文件的不同位置依次删除（`(fid,0)`、`(fid,1)`、……）时，每次删除都从**当前**墓碑根做一次点查与一次路径复制：第 i 次删除只复制该 fid 所在的叶和 root（叶内条目数 i−1→i），写入 O(g) 节点、O(C·g) 条目，与 F 无关。墓碑键 `(fid,pos)` 按 pos 数值序排列，同一文件的条目落在同一叶或相邻叶，因此不会扩散到无关文件的子树；当叶分裂时按 §3.2 的容量规则复制 O(g) 个节点，并只触及同父兄弟。重复删除同一 `(fid,pos)` 则在第 4.2 第一段的路径上直接返回。

### 4.3 移除带墓碑的输入

`publish` 对每个输入 fid 在墓碑树做**前缀范围删除**，命中条目数 d，删除后新墓碑根不再包含这些条目；这与现状 `{(f,p) for f,p in dead if f not in inputs}`（`lake.py:66`）等价，但不需要物化其余墓碑。`local` 第 2 步发布后 `dead=[]`，f00 墓碑随输入一起消失；证据 predicted 明确说明按 fid 前缀只读 1 条并复制 1 个空叶根。

### 4.4 空文件与零输出发布

- 文件键的存在性与行数无关；`file_keys` 会把零行文件也作为键返回，`objects.get` 返回 `[]`，`read` 不产生行。`local` 快照 3 的 `files` 含 `rewrite-local-1`（零行），读输出仍 15 行；若用 head 的 logical 状态冒充历史快照，会漏掉这个键——本方案按快照 3 自己的根遍历。
- `prepare` 对零行输入：`expected` 为空集（`range(len([]))`），空 `layout` 合法（`lake.py:44-46`）。
- `publish` 的 `outputs=[]`：文件树只删除输入键，不插入输出键；仍创建新快照（`local` 第 4 步 head=4，files 8），旧快照的根不变。写入是路径复制 O(h) 节点，不是 8 个文件键的全量复制。
- 空 `layout` 不产生 `objects.put`，所以没有新载荷对象。

### 4.5 staged 发布冲突

`prepare` 只点查 head 得到 base，不登记保留、不锁；`publish` 在 `BEGIN IMMEDIATE` 内重新点查 head 并比较 `sid == plan["base"]`。中间若有 `delete` 提交（产生新 head），`publish` 报 `PUBLISH_CONFLICT`，且不写任何节点/快照/head；这与 `replay.py` 的 `between` 及 `checks.py:22-23` 一致。`rewrite` 因外层事务包住 prepare/publish，head 在内部稳定，保持现状锁范围。

### 4.6 外层回滚与不可达对象

`observe.py` 的 `rollback_delete` 在外层 `catalog.write()` 内执行一次删除后抛异常（`observe.py:109-112`）。新方案的回滚路径：

- 正常删除要写的所有索引节点、快照行、head 更新都在该最外层事务内；异常逃出后 `ROLLBACK`（`catalog.py:32-33`）把这一切撤销。
- 已提交快照引用的旧根与旧节点完全未被修改，因此逻辑 head 与历史都不变；证据 `sparse` 第 3 步 predicted 的 `logical` 与第 2 步相同。
- 若删除前已写出新载荷对象（本例 `objects.get` 只读，无新载荷；`rewrite` 中 prepare 可能写输出载荷），回滚会留下不可达对象；契约允许（CONTRACT.md 第 20 行），它们不影响任何快照的根。

### 4.7 历史一致性论证（为什么无需重放）

不变量：

1. **节点不可变**：节点 ID 为内容哈希，发布后不原地修改。任何路径复制只新建节点，旧根仍指向旧节点。
2. **版本根绑定**：每个已发布快照行记录当时计算出的 `files_root`/`tomb_root`；读该 sid 只使用这两个根。
3. **差分只影响新根**：`delete` 保留文件根、只换墓碑根；`publish` 从旧根复制路径生成新根，删除/新增只发生在新建路径上。
4. **原子可见**：节点、快照行、head 在同一事务；读者从 head 或显式 `sid` 看到的根与载荷状态一致，不存在半发布。
5. **不重放**：读路径不跟随 `parent`，也不读取其他 sid 的行作差分；`parent` 只用于审计。
6. **历史永久**：没有删除/回收旧快照行或旧节点；快照 1 的根始终可达（在样例中，`local` 的 4 个快照各自列出正确的文件/墓碑集合）。

因此 `read(sid)` 的目标集合恰好是该快照自己的文件键集与墓碑键集；它与当前 head 的逻辑状态无关，也不会把当前状态误用于历史读取。

### 4.8 历史输出在推荐结构中的可达性核对（正文核对，不是目标结构代码）

| 实际 read | actual 行数 | 设计中的可达路径 | 关键结论 |
| --- | --- | --- | --- |
| sparse 快照 1 | 16 | 根 A（8 键、0 墓碑）→8 个载荷各 2 行 | 无墓碑过滤 |
| sparse 快照 2 | 15 | 根 A（共享 8 键）+ 墓碑根（1 条 f00:0） | 只在目标墓碑根排除 1 行 |
| sparse 快照 3 | 14 | 根 A（共享 8 键）+ 墓碑根（2 条 f00:0,f01:0） | 排除 2 行；快照 2 的根仍指向原节点 |
| local 快照 1 | 16 | 根 A（8 键、0 墓碑） | 后续发布未修改该根 |
| local 快照 2 | 15 | 根 A + 1 条 f00:0 墓碑 | 删除产生的快照 2 独立 |
| local 快照 3 | 15 | 根 B（9 键，含零行 `rewrite-local-1`，0 墓碑） | 必须遍历 9 个键；零行文件贡献 0 行，`rewrite-local-0` 贡献 1 行 |
| local 快照 4 | 15 | 根 C（8 键，零行输入已移除，0 墓碑） | 删键只影响新根，快照 1—3 的根不变 |

这些行数与 `observe.py:150-158` 的断言一致；证据块中保留了每一步的真实 `read.result`，predicted 的 `logical` 只是对当前逻辑状态的同形预测，历史可达性由本节和 `access` 文本核对，**没有把 predicted 当作实际历史读取结果**。

## 5. 成本上界

### 5.1 参数

- F：当前/目标版本文件键数；D：当前/目标墓碑条目数；H：历史快照数；
- k：本次 prepare/publish 输入文件数；m：输出文件数；d：输入对应的已有墓碑条目数；
- C：节点容量（叶键数，内部子指针数 C+1）；h=O(log_{C+1} F)，g=O(log_{C+1} D)；
- 成本单位：**条目**（一个键或一个子指针）+ **元数据行**（快照/head）。载荷行单独标注，属于业务必需读取（`objects.py:30-32`）。

### 5.2 操作上界（推荐方案 B；与 A 对照）

| 操作 | B 元数据读取 | B 元数据写入 | 关键原因 |
| --- | --- | --- | --- |
| 单点 delete | 快照点查 2 行（源 + 当前；默认快照时加 1 head 行）+ 2 次文件点查 O(C·h) + 1 次墓碑点查 O(C·g) + 载荷文件 | O(C·g) 条目（新墓碑路径）+ 1 快照行 + 1 head 行 | 文件根不变；只复制墓碑路径 |
| 重复 delete | 同上，无写 | 0 | `tomb_hit` 命中即返回 |
| prepare | 1 head 行 + 1 快照行 + k 次文件点查 O(k·C·h) + 墓碑前缀 O(k·C·g + d) | 0 | 不读无关文件/全体墓碑；载荷按输入读取 |
| publish | 1 head 行 + k 次文件点查 O(k·C·h) + d 墓碑条目 | O(C·((k+m)·h + (k+d)·g)) 条目 + 1 快照行 + 1 head 行 | 只复制输入/输出路径；输出含零行文件 |
| read(sid) | 1 快照行 + 文件树 F_sid + 墓碑树 D_sid + O(C(h+g)) 路径 | 0 | 目标版本自身遍历，不重放 |
| seed | 可 bulk-load 全部输入 O(F) 条目 | O(F/C) 节点 + 1 快照行 | 契约允许 seed 建全索引 |
| 存储 | — | O(F + C·Σ_i(k_i+m_i+d_i)(h_i+g_i)) 条目 + H 快照行，节点内容可去重 | 不再 O(H·(F+D)) |

策略 A 对照：delete 写 O(F′+D′)，publish 写 O(F′+D′+m)，prepare 读 O(F+D)，read 读 O(F_sid+D_sid)，存储 O(H·(F+D))。A 与 B 的历史读都只依赖目标版本；差别在写侧、存储侧和对无关文件的接触。

### 5.3 随各量增长的形态

- **文件数 F**：B 的写只随 `h≈log F` 增长；无关文件不进入访问/复制。A 每代线性写 F。
- **墓碑数 D**：B 单点删除 O(log D) 路径；同文件多次稀疏删除时 d 增长，publish 的范围删除按 d 条目计；A 每代重写 D。
- **历史数 H**：B 每代新增一条快照行 + 本次路径节点，读与 H 无关；A 存储 O(H·(F+D))，读仍与 H 无关。
- **本次变化量 k、m、d**：B 写随 `(k+m)h + (k+d)g` 增长，是操作实际规模，而非数据库总规模。
- **节点容量 C**：每个新节点最多 2C+1 条目（内部）或 C 条目（叶）；路径复制按节点计费，因此“只改一个键”至少付 O(C·h) 条目。这是把节点内条目计入成本后的真实上界，不能用一个“索引节点”掩盖整份清单。

## 6. 实测与预测逐项核对

- §1.2 是**基线实测**：来自 `python3 observe.py demo`，当前 `Catalog.state/commit` 实现；证据块 `actual` 与之一致。
- 证据块 `predicted` 是**设计预测**：每步给出同形 `logical` 与 `access` 文字，逐步说明方案会访问/复制哪些索引节点，`read` 步说明如何由 `sid` 快照行定位目标根、如何遍历，回滚步说明未发布状态如何被事务撤销。
- 预测的 `logical` 与实测逐项相等（`design_check.py:20-21` 会比对），但这只说明方案没有改变外部行为；访问成本与事务伪代码仍须人工审查（`design_check.py` 末行输出也明确这样声明）。
- 特别地，`actual` 的 `cost` 计数只统计旧实现四张表的行数；新方案的节点/条目成本无法由该观察器直接计量，因此本文没有把旧计数冒称为新结构的实测。

## 7. 可重跑验收

```text
python3 checks.py                # BASELINE_OK
python3 replay.py all            # normal/before/between/after 当前行 4/3/3/4，历史 4；between=CONFLICT，after=STALE
python3 observe.py verify        # METADATA_OBSERVATION_OK
python3 observe.py demo          # 每行 {"case","actual"}
python3 design_check.py          # 重跑回归+观察器，核对本文证据；DESIGN_EVIDENCE_OK
```

`.pointwise-runtime.json` 的 `verify` 将改为 `["python3","design_check.py"]`，`demo` 保持 `["python3","observe.py","demo"]`。生产源码、`checks.py`、`replay.py`、`samples.py`、`observe.py`、`examples/*` 均不修改，原断言不动。

## 8. 不变量与风险清单

1. 外部接口/返回结构/错误串联顺序/快照编号 `parent+1`/`PUBLISH_CONFLICT` 条件全部保持。
2. 单点删除只点查源与当前文件路径、点查墓碑，只写墓碑路径；不复制文件清单。
3. prepare/publish 只访问输入/输出文件路径与这些输入的墓碑范围；零行文件仍是键。
4. 历史读由 `sid` 快照行的两个根直接定位，按目标版本遍历，不跟 parent、不重放、不全历史筛选。
5. 所有元数据写入与 head 在同一事务；回滚一起消失，不可达载荷允许。
6. 节点容量 C 与节点条目计入成本；树高、路径复制成本已给出上界。
7. 风险/未做：不实现新引擎、不迁移旧库、不回收旧节点/载荷、不做分支/多维护者协调、不优化 `rewrite` 的锁范围、不允许旧位置删除穿过重写（保持 `STALE_LAYOUT`）。

<!-- metadata-evidence -->
```json
{
 "sparse": {
  "actual": {
   "initial": {
    "head": 1,
    "files": [
     "f00",
     "f01",
     "f02",
     "f03",
     "f04",
     "f05",
     "f06",
     "f07"
    ],
    "dead": []
   },
   "steps": [
    {
     "op": "delete",
     "snapshot": 2,
     "cost": {
      "read": {
       "snapshots": 2,
       "files": 16,
       "tombstones": 0,
       "head": 2
      },
      "write": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 1,
       "head": 1
      }
     },
     "logical": {
      "head": 2,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ]
      ]
     }
    },
    {
     "op": "delete",
     "snapshot": 3,
     "cost": {
      "read": {
       "snapshots": 2,
       "files": 16,
       "tombstones": 1,
       "head": 2
      },
      "write": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 2,
       "head": 1
      }
     },
     "logical": {
      "head": 3,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ],
       [
        "f01",
        0
       ]
      ]
     }
    },
    {
     "op": "delete",
     "snapshot": 3,
     "cost": {
      "read": {
       "snapshots": 2,
       "files": 16,
       "tombstones": 2,
       "head": 1
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 3,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ],
       [
        "f01",
        0
       ]
      ]
     }
    },
    {
     "op": "rollback_delete",
     "error": "OBSERVER_ROLLBACK",
     "cost": {
      "read": {
       "snapshots": 2,
       "files": 16,
       "tombstones": 2,
       "head": 2
      },
      "write": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 3,
       "head": 1
      }
     },
     "logical": {
      "head": 3,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ],
       [
        "f01",
        0
       ]
      ]
     }
    },
    {
     "op": "read",
     "result": {
      "snapshot": 1,
      "rows": [
       {
        "file": "f00",
        "pos": 0,
        "value": {
         "reading": 0
        }
       },
       {
        "file": "f00",
        "pos": 1,
        "value": {
         "reading": 1
        }
       },
       {
        "file": "f01",
        "pos": 0,
        "value": {
         "reading": 2
        }
       },
       {
        "file": "f01",
        "pos": 1,
        "value": {
         "reading": 3
        }
       },
       {
        "file": "f02",
        "pos": 0,
        "value": {
         "reading": 4
        }
       },
       {
        "file": "f02",
        "pos": 1,
        "value": {
         "reading": 5
        }
       },
       {
        "file": "f03",
        "pos": 0,
        "value": {
         "reading": 6
        }
       },
       {
        "file": "f03",
        "pos": 1,
        "value": {
         "reading": 7
        }
       },
       {
        "file": "f04",
        "pos": 0,
        "value": {
         "reading": 8
        }
       },
       {
        "file": "f04",
        "pos": 1,
        "value": {
         "reading": 9
        }
       },
       {
        "file": "f05",
        "pos": 0,
        "value": {
         "reading": 10
        }
       },
       {
        "file": "f05",
        "pos": 1,
        "value": {
         "reading": 11
        }
       },
       {
        "file": "f06",
        "pos": 0,
        "value": {
         "reading": 12
        }
       },
       {
        "file": "f06",
        "pos": 1,
        "value": {
         "reading": 13
        }
       },
       {
        "file": "f07",
        "pos": 0,
        "value": {
         "reading": 14
        }
       },
       {
        "file": "f07",
        "pos": 1,
        "value": {
         "reading": 15
        }
       }
      ]
     },
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 0,
       "head": 0
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 3,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ],
       [
        "f01",
        0
       ]
      ]
     }
    },
    {
     "op": "read",
     "result": {
      "snapshot": 2,
      "rows": [
       {
        "file": "f00",
        "pos": 1,
        "value": {
         "reading": 1
        }
       },
       {
        "file": "f01",
        "pos": 0,
        "value": {
         "reading": 2
        }
       },
       {
        "file": "f01",
        "pos": 1,
        "value": {
         "reading": 3
        }
       },
       {
        "file": "f02",
        "pos": 0,
        "value": {
         "reading": 4
        }
       },
       {
        "file": "f02",
        "pos": 1,
        "value": {
         "reading": 5
        }
       },
       {
        "file": "f03",
        "pos": 0,
        "value": {
         "reading": 6
        }
       },
       {
        "file": "f03",
        "pos": 1,
        "value": {
         "reading": 7
        }
       },
       {
        "file": "f04",
        "pos": 0,
        "value": {
         "reading": 8
        }
       },
       {
        "file": "f04",
        "pos": 1,
        "value": {
         "reading": 9
        }
       },
       {
        "file": "f05",
        "pos": 0,
        "value": {
         "reading": 10
        }
       },
       {
        "file": "f05",
        "pos": 1,
        "value": {
         "reading": 11
        }
       },
       {
        "file": "f06",
        "pos": 0,
        "value": {
         "reading": 12
        }
       },
       {
        "file": "f06",
        "pos": 1,
        "value": {
         "reading": 13
        }
       },
       {
        "file": "f07",
        "pos": 0,
        "value": {
         "reading": 14
        }
       },
       {
        "file": "f07",
        "pos": 1,
        "value": {
         "reading": 15
        }
       }
      ]
     },
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 1,
       "head": 0
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 3,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ],
       [
        "f01",
        0
       ]
      ]
     }
    },
    {
     "op": "read",
     "result": {
      "snapshot": 3,
      "rows": [
       {
        "file": "f00",
        "pos": 1,
        "value": {
         "reading": 1
        }
       },
       {
        "file": "f01",
        "pos": 1,
        "value": {
         "reading": 3
        }
       },
       {
        "file": "f02",
        "pos": 0,
        "value": {
         "reading": 4
        }
       },
       {
        "file": "f02",
        "pos": 1,
        "value": {
         "reading": 5
        }
       },
       {
        "file": "f03",
        "pos": 0,
        "value": {
         "reading": 6
        }
       },
       {
        "file": "f03",
        "pos": 1,
        "value": {
         "reading": 7
        }
       },
       {
        "file": "f04",
        "pos": 0,
        "value": {
         "reading": 8
        }
       },
       {
        "file": "f04",
        "pos": 1,
        "value": {
         "reading": 9
        }
       },
       {
        "file": "f05",
        "pos": 0,
        "value": {
         "reading": 10
        }
       },
       {
        "file": "f05",
        "pos": 1,
        "value": {
         "reading": 11
        }
       },
       {
        "file": "f06",
        "pos": 0,
        "value": {
         "reading": 12
        }
       },
       {
        "file": "f06",
        "pos": 1,
        "value": {
         "reading": 13
        }
       },
       {
        "file": "f07",
        "pos": 0,
        "value": {
         "reading": 14
        }
       },
       {
        "file": "f07",
        "pos": 1,
        "value": {
         "reading": 15
        }
       }
      ]
     },
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 2,
       "head": 0
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 3,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ],
       [
        "f01",
        0
       ]
      ]
     }
    },
    {
     "op": "delete",
     "error": "BAD_POSITION",
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 0,
       "head": 0
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 3,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ],
       [
        "f01",
        0
       ]
      ]
     }
    }
   ]
  },
  "predicted": [
   {
    "logical": {
     "head": 2,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ]
     ]
    },
    "access": "点查 snapshots(sid=1) 一行取得 files_root/tomb_root；文件树按 f00 下潜 = root（1 分隔键 + 2 子指针 = 3 条目）+ 叶（f00..f03 共 4 条目）= 2 节点、7 条目，确认 f00 在源版本；Objects.get(f00) 读 2 行载荷校验 pos=0。BEGIN IMMEDIATE 后点查 head 一行与当前快照行；当前文件根与快照1相同，再点查 f00（2 节点、7 条目）；墓碑树点查 (f00,0)：空叶根，1 节点、0 条目，未命中。写：文件根原值复用、0 节点复制；墓碑树新建 1 个叶根（1 条目），加 snapshots 1 行、head 1 行。基线实测同一步写 files=8、tombstones=1、snapshots=1、head=1；新方案不复制 8 个文件键。"
   },
   {
    "logical": {
     "head": 3,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ],
      [
       "f01",
       0
      ]
     ]
    },
    "access": "点查 snapshots(sid=1) 1 行定位源根；文件树点查 f01（2 节点、7 条目）；Objects.get(f01) 读 2 行。写事务内点查 head（sid=2）与当前快照行；文件根仍共享，点查 f01（2 节点、7 条目）；墓碑树点查 (f01,0)（1 个叶根、1 条目）未命中。写：文件根 0 节点复制；墓碑叶根复制 1 个节点（旧 1 条目→新 2 条目），snapshots 1 行、head 1 行。基线实测写 files=8、tombstones=2，即每代复制 8 个文件键；新方案只复制墓碑路径。"
   },
   {
    "logical": {
     "head": 3,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ],
      [
       "f01",
       0
      ]
     ]
    },
    "access": "点查 snapshots(sid=1) 1 行、文件树 f00 点查（2 节点、7 条目）、Objects.get(f00) 2 行；写事务内点查 head(sid=3) 与当前快照行，当前文件根点查 f00（2 节点、7 条目）；墓碑树点查 (f00,0)：叶根有 2 条目并命中，lake.py 的重复删除分支直接返回 sid=3。没有 INSERT/UPDATE，也不新建快照；文件树与墓碑树均 0 复制（叶根内容未变，node_id 不变）。基线实测写全为 0，与此一致。"
   },
   {
    "logical": {
     "head": 3,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ],
      [
       "f01",
       0
      ]
     ]
    },
    "access": "源版本仍是快照1：点查 1 行 + 文件树 f02（2 节点、7 条目）+ 载荷 2 行；写事务内 head=3、当前文件根点查 f02（2 节点、7 条目）、墓碑树点查 (f02,0)（1 个叶根、2 条目）未命中。遂执行与正常删除相同的写：新墓碑叶根（3 条目）、snapshots(sid=4) 1 行、head 更新 1 行；观察器抛 RollbackProbe 后最外层 write 执行 ROLLBACK，这些行与 head 更新一起消失，已提交快照1—3引用的旧节点不变，逻辑 head 仍是 3。基线实测记录了已执行但回滚的写 files=8、tombstones=3、snapshots=1、head=1；新方案同样只记录路径写（1 个墓碑节点 + 1 快照行 + 1 head 行），文件树 0 复制。"
   },
   {
    "logical": {
     "head": 3,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ],
      [
       "f01",
       0
      ]
     ]
    },
    "access": "点查 snapshots(sid=1) 一行即绑定该版本的 files_root/tomb_root，不沿 parent 回放，也不扫描其他快照行。按 root 中序遍历文件树：root + 2 叶 = 3 节点、11 条目（8 键 + 1 分隔键 + 2 子指针）；墓碑树为空叶根 = 1 节点、0 条目。按文件名序读取 8 个 Objects 载荷共 16 行，无墓碑过滤。成本只随该版本 F=8、D=0 增长，与 H=3 无关；基线实测读 snapshots=1、files=8、tombstones=0。"
   },
   {
    "logical": {
     "head": 3,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ],
      [
       "f01",
       0
      ]
     ]
    },
    "access": "点查 snapshots(sid=2) 一行得到独立根；文件根与快照1共享同一持久节点（3 节点、11 条目）；墓碑根是快照2新增的叶根（1 节点、1 条目：f00:0）。遍历 8 个文件、16 行载荷，按该版本墓碑排除 f00:0，输出 15 行。没有读取快照1或快照3的行来推导差分；基线实测读 files=8、tombstones=1。"
   },
   {
    "logical": {
     "head": 3,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ],
      [
       "f01",
       0
      ]
     ]
    },
    "access": "点查 snapshots(sid=3) 一行；文件根仍与快照1共享（3 节点、11 条目）；墓碑根是快照3的叶根（1 节点、2 条目：f00:0、f01:0）。读取 8 个载荷 16 行后排除 2 行，输出 14 行。定位只依赖 sid 主键点查与该版本根，成本 O(F+D)，不随历史代数增长；基线实测读 files=8、tombstones=2。"
   },
   {
    "logical": {
     "head": 3,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ],
      [
       "f01",
       0
      ]
     ]
    },
    "access": "点查 snapshots(sid=1) 一行；文件树点查 f00（2 节点、7 条目）确认文件在源版本；pos=99 是整数且 ≥0，于是按 lake.py 的短路顺序读取 Objects.get(f00) 的 2 行判断上界，99≥2 立即抛 BAD_POSITION。此时尚未进入写事务，没有 STALE_LAYOUT 检查、没有墓碑查找、没有 DML；新方案不枚举 8 个文件或全部墓碑。基线实测因 state() 物化而读 files=8、写 0。"
   }
  ]
 },
 "local": {
  "actual": {
   "initial": {
    "head": 1,
    "files": [
     "f00",
     "f01",
     "f02",
     "f03",
     "f04",
     "f05",
     "f06",
     "f07"
    ],
    "dead": []
   },
   "steps": [
    {
     "op": "delete",
     "snapshot": 2,
     "cost": {
      "read": {
       "snapshots": 2,
       "files": 16,
       "tombstones": 0,
       "head": 2
      },
      "write": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 1,
       "head": 1
      }
     },
     "logical": {
      "head": 2,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ]
      ]
     }
    },
    {
     "op": "prepare",
     "plan": {
      "base": 2,
      "inputs": [
       "f00"
      ],
      "outputs": [
       "rewrite-local-0",
       "rewrite-local-1"
      ]
     },
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 1,
       "head": 1
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 2,
      "files": [
       "f00",
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07"
      ],
      "dead": [
       [
        "f00",
        0
       ]
      ]
     }
    },
    {
     "op": "publish",
     "snapshot": 3,
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 1,
       "head": 2
      },
      "write": {
       "snapshots": 1,
       "files": 9,
       "tombstones": 0,
       "head": 1
      }
     },
     "logical": {
      "head": 3,
      "files": [
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07",
       "rewrite-local-0",
       "rewrite-local-1"
      ],
      "dead": []
     }
    },
    {
     "op": "prepare",
     "plan": {
      "base": 3,
      "inputs": [
       "rewrite-local-1"
      ],
      "outputs": []
     },
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 9,
       "tombstones": 0,
       "head": 1
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 3,
      "files": [
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07",
       "rewrite-local-0",
       "rewrite-local-1"
      ],
      "dead": []
     }
    },
    {
     "op": "publish",
     "snapshot": 4,
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 9,
       "tombstones": 0,
       "head": 2
      },
      "write": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 0,
       "head": 1
      }
     },
     "logical": {
      "head": 4,
      "files": [
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07",
       "rewrite-local-0"
      ],
      "dead": []
     }
    },
    {
     "op": "read",
     "result": {
      "snapshot": 1,
      "rows": [
       {
        "file": "f00",
        "pos": 0,
        "value": {
         "reading": 0
        }
       },
       {
        "file": "f00",
        "pos": 1,
        "value": {
         "reading": 1
        }
       },
       {
        "file": "f01",
        "pos": 0,
        "value": {
         "reading": 2
        }
       },
       {
        "file": "f01",
        "pos": 1,
        "value": {
         "reading": 3
        }
       },
       {
        "file": "f02",
        "pos": 0,
        "value": {
         "reading": 4
        }
       },
       {
        "file": "f02",
        "pos": 1,
        "value": {
         "reading": 5
        }
       },
       {
        "file": "f03",
        "pos": 0,
        "value": {
         "reading": 6
        }
       },
       {
        "file": "f03",
        "pos": 1,
        "value": {
         "reading": 7
        }
       },
       {
        "file": "f04",
        "pos": 0,
        "value": {
         "reading": 8
        }
       },
       {
        "file": "f04",
        "pos": 1,
        "value": {
         "reading": 9
        }
       },
       {
        "file": "f05",
        "pos": 0,
        "value": {
         "reading": 10
        }
       },
       {
        "file": "f05",
        "pos": 1,
        "value": {
         "reading": 11
        }
       },
       {
        "file": "f06",
        "pos": 0,
        "value": {
         "reading": 12
        }
       },
       {
        "file": "f06",
        "pos": 1,
        "value": {
         "reading": 13
        }
       },
       {
        "file": "f07",
        "pos": 0,
        "value": {
         "reading": 14
        }
       },
       {
        "file": "f07",
        "pos": 1,
        "value": {
         "reading": 15
        }
       }
      ]
     },
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 0,
       "head": 0
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 4,
      "files": [
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07",
       "rewrite-local-0"
      ],
      "dead": []
     }
    },
    {
     "op": "read",
     "result": {
      "snapshot": 2,
      "rows": [
       {
        "file": "f00",
        "pos": 1,
        "value": {
         "reading": 1
        }
       },
       {
        "file": "f01",
        "pos": 0,
        "value": {
         "reading": 2
        }
       },
       {
        "file": "f01",
        "pos": 1,
        "value": {
         "reading": 3
        }
       },
       {
        "file": "f02",
        "pos": 0,
        "value": {
         "reading": 4
        }
       },
       {
        "file": "f02",
        "pos": 1,
        "value": {
         "reading": 5
        }
       },
       {
        "file": "f03",
        "pos": 0,
        "value": {
         "reading": 6
        }
       },
       {
        "file": "f03",
        "pos": 1,
        "value": {
         "reading": 7
        }
       },
       {
        "file": "f04",
        "pos": 0,
        "value": {
         "reading": 8
        }
       },
       {
        "file": "f04",
        "pos": 1,
        "value": {
         "reading": 9
        }
       },
       {
        "file": "f05",
        "pos": 0,
        "value": {
         "reading": 10
        }
       },
       {
        "file": "f05",
        "pos": 1,
        "value": {
         "reading": 11
        }
       },
       {
        "file": "f06",
        "pos": 0,
        "value": {
         "reading": 12
        }
       },
       {
        "file": "f06",
        "pos": 1,
        "value": {
         "reading": 13
        }
       },
       {
        "file": "f07",
        "pos": 0,
        "value": {
         "reading": 14
        }
       },
       {
        "file": "f07",
        "pos": 1,
        "value": {
         "reading": 15
        }
       }
      ]
     },
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 1,
       "head": 0
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 4,
      "files": [
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07",
       "rewrite-local-0"
      ],
      "dead": []
     }
    },
    {
     "op": "read",
     "result": {
      "snapshot": 3,
      "rows": [
       {
        "file": "f01",
        "pos": 0,
        "value": {
         "reading": 2
        }
       },
       {
        "file": "f01",
        "pos": 1,
        "value": {
         "reading": 3
        }
       },
       {
        "file": "f02",
        "pos": 0,
        "value": {
         "reading": 4
        }
       },
       {
        "file": "f02",
        "pos": 1,
        "value": {
         "reading": 5
        }
       },
       {
        "file": "f03",
        "pos": 0,
        "value": {
         "reading": 6
        }
       },
       {
        "file": "f03",
        "pos": 1,
        "value": {
         "reading": 7
        }
       },
       {
        "file": "f04",
        "pos": 0,
        "value": {
         "reading": 8
        }
       },
       {
        "file": "f04",
        "pos": 1,
        "value": {
         "reading": 9
        }
       },
       {
        "file": "f05",
        "pos": 0,
        "value": {
         "reading": 10
        }
       },
       {
        "file": "f05",
        "pos": 1,
        "value": {
         "reading": 11
        }
       },
       {
        "file": "f06",
        "pos": 0,
        "value": {
         "reading": 12
        }
       },
       {
        "file": "f06",
        "pos": 1,
        "value": {
         "reading": 13
        }
       },
       {
        "file": "f07",
        "pos": 0,
        "value": {
         "reading": 14
        }
       },
       {
        "file": "f07",
        "pos": 1,
        "value": {
         "reading": 15
        }
       },
       {
        "file": "rewrite-local-0",
        "pos": 0,
        "value": {
         "reading": 1
        }
       }
      ]
     },
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 9,
       "tombstones": 0,
       "head": 0
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 4,
      "files": [
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07",
       "rewrite-local-0"
      ],
      "dead": []
     }
    },
    {
     "op": "read",
     "result": {
      "snapshot": 4,
      "rows": [
       {
        "file": "f01",
        "pos": 0,
        "value": {
         "reading": 2
        }
       },
       {
        "file": "f01",
        "pos": 1,
        "value": {
         "reading": 3
        }
       },
       {
        "file": "f02",
        "pos": 0,
        "value": {
         "reading": 4
        }
       },
       {
        "file": "f02",
        "pos": 1,
        "value": {
         "reading": 5
        }
       },
       {
        "file": "f03",
        "pos": 0,
        "value": {
         "reading": 6
        }
       },
       {
        "file": "f03",
        "pos": 1,
        "value": {
         "reading": 7
        }
       },
       {
        "file": "f04",
        "pos": 0,
        "value": {
         "reading": 8
        }
       },
       {
        "file": "f04",
        "pos": 1,
        "value": {
         "reading": 9
        }
       },
       {
        "file": "f05",
        "pos": 0,
        "value": {
         "reading": 10
        }
       },
       {
        "file": "f05",
        "pos": 1,
        "value": {
         "reading": 11
        }
       },
       {
        "file": "f06",
        "pos": 0,
        "value": {
         "reading": 12
        }
       },
       {
        "file": "f06",
        "pos": 1,
        "value": {
         "reading": 13
        }
       },
       {
        "file": "f07",
        "pos": 0,
        "value": {
         "reading": 14
        }
       },
       {
        "file": "f07",
        "pos": 1,
        "value": {
         "reading": 15
        }
       },
       {
        "file": "rewrite-local-0",
        "pos": 0,
        "value": {
         "reading": 1
        }
       }
      ]
     },
     "cost": {
      "read": {
       "snapshots": 1,
       "files": 8,
       "tombstones": 0,
       "head": 0
      },
      "write": {
       "snapshots": 0,
       "files": 0,
       "tombstones": 0,
       "head": 0
      }
     },
     "logical": {
      "head": 4,
      "files": [
       "f01",
       "f02",
       "f03",
       "f04",
       "f05",
       "f06",
       "f07",
       "rewrite-local-0"
      ],
      "dead": []
     }
    }
   ]
  },
  "predicted": [
   {
    "logical": {
     "head": 2,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ]
     ]
    },
    "access": "与 sparse 第 0 步相同：snapshots(sid=1) 点查 1 行；文件树 f00 点查（2 节点、7 条目）；载荷 2 行；写事务内 head 与当前快照行、当前文件根 f00 点查（2 节点、7 条目）、墓碑空叶根（1 节点、0 条目）；写 1 个墓碑叶根（1 条目）+ snapshots 1 行 + head 1 行，文件树 0 节点复制。基线实测写 files=8、tombstones=1。"
   },
   {
    "logical": {
     "head": 2,
     "files": [
      "f00",
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07"
     ],
     "dead": [
      [
       "f00",
       0
      ]
     ]
    },
    "access": "prepare 不计写事务：点查 head 一行与当前快照行；对唯一输入 f00 做文件树点查（2 节点、7 条目）完成 BAD_INPUTS 判定；Objects.get(f00) 读 2 行；墓碑树按 fid=f00 等值点查，命中 1 条目（1 个叶根），不扫描全体墓碑。layout 校验只用 2 个物理位置。输出载荷 rewrite-local-0（1 行）与 rewrite-local-1（0 行）经 objects.put 写入，属于载荷而非元数据；元数据 DML=0。基线实测读 files=8、tombstones=1 是 state() 物化全部集合所致。"
   },
   {
    "logical": {
     "head": 3,
     "files": [
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07",
      "rewrite-local-0",
      "rewrite-local-1"
     ],
     "dead": []
    },
    "access": "BEGIN IMMEDIATE 中点查 head（sid=2）与 base 相等；文件树点查输入 f00（2 节点、7 条目）；墓碑树按 fid=f00 等值点查，取 1 条目删除。文件树 COW 按“先删 f00、再按序插入两个输出”执行，叶层聚合后一次写新节点：叶 [f01,f02,f03] 3 条目、叶 [f04,f05,f06] 3 条目、叶 [f07,rewrite-local-0,rewrite-local-1] 3 条目、root（2 分隔键 + 3 子指针）5 条目，共 4 个节点、14 条目。本样例 F=8、C=4 只有 2 个旧叶，删除与插入分别落在两叶上，所以两个旧叶都在必要路径内、其附带键计入 14 条目；这不是枚举清单，F 增大时路径只触及 O(log_C F) 个叶。墓碑树复制 1 个空叶根（删 1 条目、0 条目）。写 snapshots(sid=3) 1 行 + head 1 行。基线实测写 files=9、tombstones=0。"
   },
   {
    "logical": {
     "head": 3,
     "files": [
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07",
      "rewrite-local-0",
      "rewrite-local-1"
     ],
     "dead": []
    },
    "access": "点查 head 一行与当前快照行；输入 rewrite-local-1 的文件树点查（root 5 条目 + 叶 3 条目 = 2 节点、8 条目）存在；Objects.get 返回 0 行，所以 expected 为空集，layout=[] 的 requested 也为空集，合法；layout 无组，故不写任何输出载荷。墓碑树按 fid 等值点查，命中 0 条目（空叶根 1 节点）。元数据 DML=0。基线实测同样写 0，但读 files=9；新方案只付该文件键的路径。"
   },
   {
    "logical": {
     "head": 4,
     "files": [
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07",
      "rewrite-local-0"
     ],
     "dead": []
    },
    "access": "head 点查 sid=3 等于 base；输入 rewrite-local-1 的点查（2 节点、8 条目）存在；墓碑树按 fid 等值点查 0 条目。文件树 COW 只删 1 个键（k=1、m=0）：目标叶 [f07,rl0,rl1] 由 3 键降为 2 键，恰在半满下限 2，无须合并；写该叶（[f07,rl0] 2 条目）+ root（2 分隔键 + 3 子指针 = 5 条目）= 2 节点、7 条目，其余 2 个叶共享；无输出键。写 snapshots(sid=4) 1 行、head 1 行。空 layout/零输出发布不会物化 8 个文件清单；基线实测写 files=8。"
   },
   {
    "logical": {
     "head": 4,
     "files": [
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07",
      "rewrite-local-0"
     ],
     "dead": []
    },
    "access": "点查 snapshots(sid=1) 1 行定位根；文件树 root + 2 叶 = 3 节点、11 条目；墓碑为空叶根 1 节点、0 条目；读 8 个载荷 16 行，输出 16 行。快照1的根未被后续发布修改，故无需重放；基线实测读 files=8、tombstones=0。"
   },
   {
    "logical": {
     "head": 4,
     "files": [
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07",
      "rewrite-local-0"
     ],
     "dead": []
    },
    "access": "点查 snapshots(sid=2) 1 行；文件根与快照1共享（3 节点、11 条目）；墓碑叶根 1 节点、1 条目；读 16 行载荷、排除 f00:0，输出 15 行。只访问目标版本实际包含的文件和墓碑；基线实测读 files=8、tombstones=1。"
   },
   {
    "logical": {
     "head": 4,
     "files": [
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07",
      "rewrite-local-0"
     ],
     "dead": []
    },
    "access": "点查 snapshots(sid=3) 1 行；文件树 root + 3 叶 = 4 节点、14 条目（9 键 + 2 分隔键 + 3 子指针），含零行文件 rewrite-local-1；墓碑为空叶根 1 节点、0 条目。遍历 9 个文件键，对零行文件 Objects.get 返回 []、不产生行；共 14 个原行 + 1 个 rewrite-local-0 行 = 15 行。若用当前逻辑状态冒充快照3，会漏掉 rewrite-local-1 这个键；本方案按快照3的根遍历。基线实测读 files=9、tombstones=0。"
   },
   {
    "logical": {
     "head": 4,
     "files": [
      "f01",
      "f02",
      "f03",
      "f04",
      "f05",
      "f06",
      "f07",
      "rewrite-local-0"
     ],
     "dead": []
    },
    "access": "点查 snapshots(sid=4) 1 行；文件树 root + 3 叶 = 4 节点、13 条目（8 键 + 2 分隔键 + 3 子指针），零行输入 rewrite-local-1 已被移除；墓碑为空叶根 1 节点、0 条目。读 15 行载荷，输出 15 行。旧快照1—3的根仍指向各自节点，删键只影响新根路径，历史不变；基线实测读 files=8、tombstones=0。"
   }
  ]
 }
}
```

</content>
