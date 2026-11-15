# AGENTS.md

2026 考研复习空间（管综 199、英语二 204）。

**记录存在云端数据库里，不在这里。** 本工作区放的是回看用的本地副本、新写的总结、草稿扫描件；
所有结构化数据（题干、作答、判定、错因、重做状态）都在 `geyuqi.me` 的库里：**查直接写 SQL，写走 `mem` 命令**。

## 1. 目录

```text
mem_study/
├── 题库/                 本地副本：题干、作答与批改、答案速查、各卷总结（61 份 Markdown）
├── 总结/                 新写的总结与常用口径
│   ├── 管综-批改总结.md
│   ├── 错因标签口径.md
│   └── README.md
├── 草稿/                 扫描件、照片、草稿 PDF，不进版本管理
└── AGENTS.md
```

`题库/` 的用处是**离线阅读**——`grep -rn "倍数搞错" 题库/` 能直接看到原句上下文；要定位题号、
做统计、跨卷聚合，用 SQL（§3.3）。两种检索各有适用，权威值仍在库里，二者不一致时以库为准。

## 2. 数据在哪

| 项 | 值 |
|---|---|
| 真源 | 线上 SQLite，跑在 `geyuqi.me`（`mem-api.service`） |
| 规模 | 8 卷 / 343 题 / 42 场 / 343 条作答 |
| 站点 | <https://geyuqi.me/study/>，与接口同一进程实时渲染，写完立刻可见 |
| 入口 | 查询 `mem sql`（只读 SQL）· 页面口径 `/api/study` · 写入 `mem` 命令（`/api/*`） |
| 鉴权 | token 在 `~/.mem-api-token`，或 `MEM_WRITE_TOKEN` |

源码在 `../mem_web/`：SQL 用法与表结构见 `../mem_web/doc/sql.md`，写接口的字段白名单见 `../mem_web/doc/cli.md` §5。

## 3. 查库与写入

结构化数据全在线上 SQLite（4 张表，`geyuqi.me` 上的 `mem.db`）。分工是硬的：
**查直接写 SQL，写只能走 `mem` 命令**——只读执行器能查一切，但一行也改不了。

### 3.1 查：直接写 SQL

```bash
mem sql "select ..."             # 默认 SSH 到服务器执行，读真源
mem sql --http "select ..."      # 走 HTTPS /api/sql，不用 SSH（需 token）
mem sql --json "..."             # {ok, columns, rows, count, truncated, ms}，脚本用这个
mem sql --csv "..." > out.csv    # 直接给 Excel / 数透
mem schema attempts              # 字段名先查，不要猜
```

闸门：单条语句、首关键字只能是 `select` / `with` / `explain` / `values` / `pragma`、结果上限 1000 行、
20 s 超时。`insert` / `update` / `delete` 会被拒（`bad_request`）——**不是权限没配好，是设计如此**。

**唯一目标**：`mem sql` 默认打线上 `https://geyuqi.me`（唯一可写的生产库），查的就是真源，不切换目标。

**四条铁律**：

1. 脚本化调用一律加 `--json`，解析 `rows`，别正则人话表格。
2. 写之前先用 SQL 查 id（`attempts.id`、`sessions.id`），**不要自己拼 id**。
3. 只读与写入都只打默认线上目标。
4. 4xx 不重试，按 `code` 处理；写完用 `mem s get <id> --json` 回读确认。

**页面口径的统计不用自己算**：站点的聚合（学习天数、连续天数、各卷掌握度、知识点正确率、
周聚合）已经在渲染页面时算好，`/api/study` 原样返回这份结果——它和页面读的是同一时刻
同一份库，所以数字与站点显示严格一致：

```bash
curl -s https://geyuqi.me/api/study?section=stats      # 学习天数 / 连续 / 总正确率
curl -s https://geyuqi.me/api/study?section=volumes    # 各卷掌握度（进度、正确率、用时）
curl -s https://geyuqi.me/api/study?section=knowledge  # 知识点树与考点正确率
```

`section` 可选 `meta` / `stats` / `today` / `days` / `weeks` / `countdown` / `volumes` /
`knowledge` / `warnings`；不带则返回全集（约 96 KB）。写进总结的统计数字**优先取这里**——
免去了自己对齐 `correct` 三态、`distinct` 去重、`duration_min` 为空不估算这些口径。
SQL 留给固定切面覆盖不了的临时分析（§3.3）。

> `/api/study` 已在线上可用（2026-11-12 验证 `section=stats` 返回 200），直接 curl 即可。

**录场次：`fullText` 默认不传，也不要传（踩过坑）**。不传时服务端自动生成「标题 + 录入说明 + 逐题两行答案」，
这种块必然是纯汇总，原文页跳过它、直接渲染逐题作答卡片。判据在 `web/render/render.mjs::isAnswerSummaryOnly`：
全文块去掉标题、`>` 引用、分隔线、`- 我的答案/正确答案/得分` 行后，**剩余正文 < 40 字**才算汇总。
只要塞进任何正文段落（哪怕一段「共性归纳」），整篇就被当正文渲染——15 题变成一篇长文、没有卡片。
逐题分析一律写进 `attempts[].process`（库列 `process_md`）；整场归纳写 `总结/`，不往 `fullText` 塞。
改单题正文用 `mem a set <id> --file`，只传分析段——题头与 `### Answer` 由服务端生成，把标题一起回写会重复。

### 3.2 四张表

```text
papers     (slug, title, subject, question_ids, question_count, score_rule_json, notes, updated_at)
questions  (id, paper_slug, number, seq, type, topic_id, topic, origin, content_md, answer,
            scorable, notes, updated_at)
sessions   (id, paper_slug, date, started_at, duration_min, timing_mode, duration_source,
            question_ids, question_count, note, updated_at)
attempts   (id, session_id, question_id, user_answer, correct, substantive_correct, score,
            score_max, is_redo, error_tag, redo_status, resolved_at, process_md, notes, updated_at)
```

- `correct` 三态：`1` 对 / `0` 错 / `NULL` **未判定**。算正确率必须带 `where correct is not null`，否则未判定的会被算成错。
- `topic` 在 `questions` 上，日期在 `sessions.date`（`YYYY-MM-DD`）。
- `content_md`（题干）与 `process_md`（批改全文，单条可达上万字符）都是长文本，**别 `select *`**。
- `redo_status` 是自由文本（`待重做`、`重做 1 次 ❌`），不是枚举；销账时间看 `resolved_at`。
- 一题可有多条作答（重做），数「做过的题」要 `count(distinct question_id)`，不是 `count(*)`。

### 3.3 常用查询

```sql
-- 某卷题号（不取题干，省得刷屏）
select number, topic, type, answer from questions where paper_slug = '2024-exam' order by seq

-- 题干全文检索（跨卷）
select id, topic, answer, substr(replace(content_md, char(10), ' '), 1, 40) 题干
from questions where content_md like '%不定方程%'

-- 某段日期的场次
select id, date, paper_slug, question_count, duration_min, timing_mode
from sessions where date between '2026-11-08' and '2026-11-11' order by date desc, id

-- 某场逐题判定（替代 mem s get <id> --process 的表格部分）
select q.number 题号, a.user_answer 作答, q.answer 标准, a.correct 判定,
       a.error_tag 错因, a.redo_status 重做
from attempts a join questions q on q.id = a.question_id
where a.session_id = '2024-exam-a3' order by q.seq

-- 待重做台账
select q.id 题, q.topic 考点, a.error_tag 错因, s.date 日期, a.redo_status 状态
from attempts a join questions q on q.id = a.question_id join sessions s on s.id = a.session_id
where a.redo_status is not null and a.redo_status not like '%✅%' order by s.date

-- 各卷进度：总题 / 已做 / 做对（一律 distinct，重做不重复计）
select q.paper_slug 卷, count(distinct q.id) 总题,
       count(distinct a.question_id) 已做,
       count(distinct case when a.correct = 1 then a.question_id end) 做对
from questions q left join attempts a on a.question_id = q.id
group by q.paper_slug order by 总题 desc

-- 某题历次作答
select s.date 日期, a.user_answer 作答, a.correct 判定, a.error_tag 错因, a.redo_status 重做
from attempts a join sessions s on s.id = a.session_id
where a.question_id = 'math-applied-distance-14' order by s.date
```

**进度用题目数，正确率用题次**，两个数不一样，写进总结时要注明口径（见第 6 节）。
拿不准口径就别自己算，直接取 `/api/study?section=volumes`（§3.1），数字与站点一致。
更多现成查询（正确率、趋势、连续学习日、时长分布）见 `../mem_web/doc/sql.md` §4。

### 3.4 写：只能走 `mem`

```bash
mem s new session.json                          # 记一场：单事务，写前自动快照
mem s set <id> durationMin=35 timingMode=session
mem a set <attemptId> correct=true errorTag=审题
mem a add <sessionId> attempt.json              # 往已有场次追加一题
mem a done <attemptId>                          # 销账：判定须为「对」
mem q set 2024-exam-q001 --file patch.json      # 长文本走 --file
```

记一场的最小 JSON（字段与 `POST /api/sessions` 一致）：

```json
{
  "paperSlug": "2024-exam",
  "date": "2026-11-11",
  "timingMode": "session",
  "durationMin": 35,
  "attempts": [
    { "questionId": "2024-exam-q001", "userAnswer": "E", "process": "代入验证，一次算对。" },
    { "questionId": "2024-exam-q002", "userAnswer": "C", "errorTag": "审题",
      "process": "把「充分」看成了「必要」，方向反了。" }
  ]
}
```

- `correct` 可省略：有标准答案的客观题自动判，判不了留 `null`（未判），**不要猜**。
- **逐题批改放 `process`**（库列 `process_md`）；`fullText` **默认不传**（理由见 §3.1），只有确需加「标题 + 一两行汇总」才传，且不得含正文段落。这两个字段都长，别塞命令行，用 `--file` 或整份 JSON。
- `errorTag` / `redoStatus` 是自由文本，库里现存中文值，照旧用，别指望枚举校验。

**为什么写不能走 SQL**：一次写要做三件事——字段校验、写前快照、导出 JSON 并刷新页表（40~230 ms）。
直接 `update` 会绕开全部三件，页面立刻和库脱节，且没有快照可退。只读执行器从根上就拒这类语句，
所以这不是「权限没配好」，而是入口只有一个：`mem` / `POST /api/*`。

## 4. 一场练习的流程

**新卷首次录入**（卷还不存在时）：`mem p new --file` 建卷 → `mem q new --file` 逐题 → `mem q set <id> answer=X` 补答案 → `mem s new` 记场次。

建卷时 `totalQuestions` / `scorableQuestions` / `answerSources` 与 `notes` 要落在 **`mem p set`** 上——`mem p new` 的白名单只有 `slug` / `title` / `subject` / `scoreRule`，这四个字段传了会被静默丢弃（2026-11-14 建 `2021-exam` 实测：传了四个字段，`notes` 存成 `{"text":null,"baseline":{}}`，站点于是把整卷显示成「进度未知 · 未闭环」；页面能打开、题也能做，所以很容易漏）。补法是：

```bash
mem p set 2021-exam questionCount=57 totalQuestions=57 scorableQuestions=55 note="……"
```

`questionCount` 也一起给：此后每次 `q new` 都会把它重算成「实际已录题数」，卷没录完（如只录了 27/57 题）时掌握度会按 27 题算。**`p set` 也一样**——不传 `questionCount` 就按实际题数重算回去（2026-11-14 实测：先设 57，改 `note`/`answerSources` 时又被压回 27），所以它是每个写操作之后的最后一步，改完别的字段再补一次。建完验一句：

```bash
curl -s "https://geyuqi.me/api/study?section=volumes" \
  | python3 -c "import sys,json;[print(v['slug'],v.get('totalQuestions'),v.get('progress')) for v in json.load(sys.stdin) if v['slug']=='<slug>']"
```

打印 `null` 就是没写进去（正常形如 `reasoning-4 15 1`）。注意这一节只列**已有作答**的卷，新卷一场没做时查不到属正常，该改用 `mem sql "select notes from papers where slug='<slug>'"` 看 `baseline.totalQuestions` 是否落库。

`mem q new` 不传 `scorable` 会存 **0**（不是 1），整卷录完要逐题 `mem q set <id> scorable=true` 补；答案页没拍时 `answer` 留空、`scorable=1` 保留，补答案后判定会自动出。

**卷还必须挂原文页，才进统计。** `build-site` 组装卷清单时只收 `baseline.paperPrefix` 非空的卷（`loadVolumes().filter(v => v.papers?.length)`），没挂 `paperFile` 的卷在掌握度里会退化成 `slug=volume-N`、`name=<slug>`、`status=unknown`，题数和进度全空——题照样能做，只是统计里认不出来。补法（长文本走 `--file`）：

```bash
mem p set <slug> --file paper_patch.json   # {"paperFile":"<卷目录>/paper.md","paperPrefix":"<页头 markdown>"}
```

`paperPrefix` 只放页头到第一个 section 头为止（标题 / 科目 / 科目代码 / 一、问题求解 + 选项说明），其余 section 由题目的 `topic` 分组渲染。挂完还要把已有场次的 `file` 挪到同一目录，否则场次按 `volume` 匹配不上、`sessions` 记 0：

```bash
mem s set <sessionId> file=<卷目录>/answerN.md
```

注意 `s set file` 会让服务端重算整场全文并写回**本场第一条作答的 `process_md`**（`content.json` 从首条取 `===== 场次全文：` 段，属设计）。所以从库里导本地副本时要按该标记截断，否则整篇汇总会混进第一题的批改里。

1. 做卷（纸上）→ 草稿拍照或扫描，存 `草稿/<日期>-<卷名>/`，不进版本管理。
2. 逐页转录 → 按第 5 节流程批改，形成逐题判定与错因。
3. `mem s new session.json` 一次写入：场次头（日期、时长、计时口径）+ 逐题（作答、判定、`process` 批改全文）。一条不合法整场不写，宁可拆两次。
   **录完立刻验**：`curl -s "https://geyuqi.me/study/source/<卷目录>/<answerN>/" | grep -c 'attempt-card'` 应等于本场题数。少于题数＝`fullText` 被当正文渲染了，按 §3.1 改成不传再 `mem s set <id> --file`。**别用 `grep -c 'class="attempt-card"'`**——错题卡片的 class 是 `attempt-card is-bad`，会少算错题数（13 题里 3 道错，只数出 10），以为渲染漏了。
4. 错题标 `errorTag`，需要重练的标 `redoStatus=待重做`。
5. 隔天不看批改、白纸重做：新场次里 `isRedo=true`，做完 `mem a set <id> correct=true`，再 `mem a done <id>` 销账。
6. 一场或一卷结束，在 `总结/` 里追加当段总结。

## 5. 批改流程

1. **逐页转录**：草稿或照片先按页转录题号、条件 (1)(2)、选项字母与判定勾叉，回题库核对题号后再判题，不凭记忆对号。
2. **独立复算**：每题先自己用分数或符号算出结果，再与作答比对；**不以「选项里有这个数」作为正确依据**——错值恰好躺在选项里，是最常见的假阳性。
3. **异常值回查**：结果与选项、条件 (1)(2) 全都不符时，默认自己的模型错了，先回查模型与方向，再判「不充分」。**E 不是垃圾桶。**
4. **回代校验**：求出人数、钱数、件数或根数后，代回一个原始条件验一次（约 20 秒），专治移项与符号 slip。**批改本身同样适用**：写进批改的「正确算式」必须自己解到最终答案并回代验证，不能只指出错误位置就下结论——只说「应得 $4y=28+k$」而不把它解完，等于把错误再传一遍。
5. **错因标签**：模型选错 / 约束遗漏 / 倍数搞错 / 移项·符号 / 算术笔误 / 书写跳步。跳步导致不可复查也算错因，不因「结果对了」放过；一题多个毛病用 `errorTags` 数组。
6. **待重做**：错题除写错因，另标 `redoStatus=待重做`；隔天不看批改、白纸重做，做完按第 4 节销账。

批改正文的固定结构：先列「题号 | 你的答案 | 正确答案 | 判定」→ 逐题分析 → 错因归纳与下一步建议。

## 6. 口径

- **速度**：`durationMin` 可选，没记就留空，不估算、不计入累计轨迹。场次级：单场 = `durationMin ÷ 题数`；按题计时：`Σ 单题用时 ÷ 有计时的题数`。两种口径**不混算**（整场用时含订正、翻页，逐题用时只含解题），`timingMode` 记 `session` / `per-question` / `mixed` / `none`。基准：首次专项 ≤4 min/题，二刷 ≤3，考场 2–2.5。
- **进度**：已完成 / 总题数按库里实际题号清点（§3.3 的进度查询，`distinct` 去重），不凭记忆估算；算错就在下一次记录里注明修正。
- **正确率**：区分卷面与实质（算对但填错、思路对但抄错，单列 `substantiveCorrect`），口径在总结中说明。
- **站点统计**：学习日与热力图按 `sessions.date` 算，不读 git 提交；页面文案里残留的「提交」字样是已知遗留，别当 bug 去"修"。

## 7. 改错了怎么退

| 情况 | 处理 |
|---|---|
| 刚记的场次不想留 | `mem s rm <sessionId>`（连作答一起删） |
| 单条作答写错 | `mem a set <id> ...` 改回；删最后一条会被拒（`last_attempt`），改删整场 |
| 判定误判 | `mem a set <id> correct=...` 覆盖，并在备注里留一句为什么改 |
| 整场搞乱 | `mem backups` 看写前快照，找用户确认后再恢复 |

`not_yet_correct`、`last_attempt`、`question_in_use` 等错误的含义见 `../mem_web/doc/api-design.md`。

## 8. 不要做的事

- 不要用 SQL 改数据（`insert` / `update` / `delete`，也不要 `sqlite3` 直写库文件）：绕过事务、写前快照与页表刷新，页面会和库脱节。**只读查询随便写 SQL，写入一律走 `mem`。**
- 不要在 `题库/` 里随手改判定或答案：那里是回看副本，改了不会同步到库，只会造出第二份不一致的数据。要更正就 `mem a set` / `mem q set`。
- 不要把整篇批改正文塞进 `fullText`，也不要为了「让归纳上页面」这么做：原文页会退化成一篇长文、逐题卡片全丢。归纳写进 `总结/`。
- 不要凭记忆报题目、答案或分数；查不到就写"未记录"。
- 不要自己恢复线上数据，不要动线上服务。
- 不要拿旧备份覆盖线上（`../mem_web/backups/*.db`、`mem backups` 写前快照）：它们是恢复用的保险，不是数据源。真要回滚先问一句。

## 9. 文风

- 中文为主，中英文之间加空格；公式用 LaTeX。
- 直接指出错在哪一步，**不编造答案或分数**，不空泛鼓励。
- 写进总结的每个数字都要能追溯到库或原文。
