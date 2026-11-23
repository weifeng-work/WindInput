# 本地分叉说明（weifeng-work/WindInput）

> 目的：给清风输入法补两个个人需求。上游不接受这类改动也没关系，这里是**自用分支**，
> 基线是上游 `main`（0.132.0）。

## 1. 改了什么

两个提交。

### 提交一：`feat(english): 码表方案下上屏的英文原文也进自动造词`

| 文件 | 改动 |
|---|---|
| `wind_input/crates/wind-engine/src/manager.rs` | 新增 `EngineManager::codetable_english_merge_enabled()` |
| `wind_input/crates/wind-coordinator/src/english_learn.rs` | 新增 `Coordinator::english_learn_target()` |
| `wind_input/crates/wind-coordinator/src/coordinator/message_handler.rs` | 4 处原码上屏出口改用新判据 |

### 提交二：`fix(punct): 码表方案下英文候选不参与标点顶屏`

| 文件 | 改动 |
|---|---|
| `wind_input/crates/wind-coordinator/src/handle_candidate.rs` | 新增 `Coordinator::punct_top_skips_english()` |
| `wind_input/crates/wind-coordinator/src/coordinator/message_handler.rs` | 标点顶屏判据加 `&& !skip_english_top`；跳过顶屏时一并跳过顶屏造词 |

**为什么需要第二条**：码表方案下打 `work`，`english_merge` 会把 `workbuddy` 等前缀补全
词混进候选并排在首位。此时按句号，上游的「标点顶屏」逻辑会把高亮候选推上去，得到
`workbuddy.` —— 但用户打的是 `work`，`workbuddy` 只是补全建议，不是他选的词。

改后：高亮候选来源是 `CandidateSource::English` 时，不顶屏，改走「上屏原码」分支，
得到 `work.`。中文码不受影响（打 `wq` 出「你」，按句号仍是「你。」）。

连带效果：

| 场景 | 行为 |
|---|---|
| `work` + `.` | `work.` |
| 打全 `workbuddy` + `.` | `workbuddy.`（原码 == 候选文本，结果不变） |
| 中文码 `wq` + `.` | `你。`（高亮是中文，不受影响） |
| 想上 `workbuddy` | 空格/数字选词，再打句号 |
| 拼音方案 | 完全不受影响（判据带 `is_codetable()` 门） |

> ⚠️ 只改了 `handle_key_event` 那条主标点通路。`commit_highlight_then_char`
> （numpad 句号、非码元字母）仍走原逻辑，暂未接——那条路没有「上屏原码」分支，
> 接进去要另加落点，本次不动。

### 为什么改（提交一）

上游的英文自动造词（`[schema.english.auto_learn]`）只在 `active_is_english()` 时生效，
也就是**只有英文方案和临时英文会学**。五笔等码表方案里打英文，走的是
`schema.codetable.english_merge`（英文候选混入），那条路**只查词库显示候选，没有任何
学习逻辑**——打多少次都记不住。

### 判据设计（关键）

码表方案下的字母串**既可能是英文也可能是五笔码**，不能见字母就学，否则会把 `rrqn`
这类有效码全收进英文词库。所以判据是：

```
学 = 开着英文候选
     且 这一串没有解出任何中文候选（CodeTable / Phrase）
     且 learnable()（纯 ASCII 字母数字、2~32 字符、至少一个字母）
```

- `workbuddy` 解不出中文 → 学 ✅
- `rrqn`（有效五笔码，有中文候选）→ 不学 ✅
- 用户没开「英文候选」→ 不学（码表方案里本就没有英文这条线）

学习落点：共享英文桶 `ENGLISH_SCHEMA`（即 `english` 方案的用户/临时词库），
所以学到的词在**英文方案、临时英文、五笔方案**三处都查得到。

### 触发出口

四处「上屏原码」都接了：空格、回车、标点顶屏（两处）。选词出口不用改——
选中英文候选时若它不在词库，那条路本来就有学习。

## 2. 英文后智能标点：不用改代码

这个上游 **0.131 就实现了**，本分支基线 0.132 自带。默认**关着**，打开即可：

```toml
[input.punct]
smart_after_latin = true
latin_smart_list = "./_:@"
```

效果：中文标点模式下，前一字符是拉丁字母时，这些标点直出半角。
`Node.` 不再变 `Node。`，`workbuddy.` 不再变 `workbuddy。`。

> 列表里没有 `,`。要逗号也走英文，自己加：`latin_smart_list = ".,/_:@"`

判据来源是「输入序列记账」（`state.spacing_prev`），不是读文档现况，所以切换应用后
第一次按键可能不准——这是上游设计，不是本分支的改动。

## 3. 怎么构建（不在本机编译）

本机不装 MSVC 工具链，全部走 GitHub Actions：

```bash
# 1. 改完推到 fork
git push fork main

# 2. 触发打包（workflow_dispatch 只产 artifact，不发 Release）
gh workflow run release.yml --repo weifeng-work/WindInput --ref main

# 3. 等 Windows 作业完成（冷编译约 23 分钟）
gh run list --repo weifeng-work/WindInput --limit 3
gh run view --job <job-id> --repo weifeng-work/WindInput

# 4. 下载
gh run download <run-id> --repo weifeng-work/WindInput -n dist-windows -D dist
```

产物：
- `dist/WindInput-Setup-<ver>.exe` —— 安装包
- `dist/WindInput-Portable-<ver>.zip` —— 绿色版

> ⚠️ 产物**没有数字签名**（`NotSigned`），安装时 SmartScreen 会拦。
> 点「更多信息」→「仍要运行」。

> ⚠️ fork 的 Actions 默认不索引工作流。首次需先往 `main` 推一次提交，
> 工作流列表才会出现（本次实测：推完 `total_count` 从 0 变 8）。

## 4. 已知限制

- **`learnable()` 挡掉带符号的串**：`C++`、`e-mail`、`Node.js` 不会被自动学。
  这是上游规则（英文方案码元只有字母），本分支沿用未放宽。
- **短英文词可能学不到**：若某个 2~4 字母串恰好是有效五笔码且有中文候选，
  判据会认为你在打中文。这是刻意的取舍，避免污染词库。
- **只覆盖「上屏原码」**：选中英文候选的路径走上游逻辑，未改动。
- **不支持拼音方案**：拼音侧有独立的 `english_merge`，语义不同，本次没接。

## 5. 和上游的关系

分支基于上游 `main`，未提交 PR。若上游愿意收，`english_learn_target()` 那一段
可以直接搬；`manager.rs` 的判据方法是新增的公开 API，不破坏现有调用。
