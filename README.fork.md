# 非官方分支说明

> **本项目是「清风输入法 (WindInput)」的非官方分支，与原项目及其作者无关。**
>
> 上游项目：<https://github.com/huanfeng/WindInput>
> 本分支作者：[@weifeng-work](https://github.com/weifeng-work)
>
> 本分支**不是**上游发布的版本，上游不对这里的问题负责；反过来，本分支的改动与缺陷
> 也不代表上游。遇到问题请先确认你装的是哪一份。

---

## 这是什么

一个自用的功能补丁分支，目的只有一个：**让五笔方案下的英文输入更好用**。

基线是上游 `main` 的 `dc80e20`（`docs/VERSION` = 0.132.0）。除下面列出的两个提交外，
其余代码与上游一致。

## 改动

### 1. `feat(english): 码表方案下上屏的英文原文也进自动造词`

**问题**：上游的英文自动造词（`[schema.english.auto_learn]`）只在 `active_is_english()`
时生效——只有「英文方案」和「临时英文」会学。五笔等码表方案里打英文，走的是
`schema.codetable.english_merge`（英文候选混入），那条路只查词库显示候选，
**没有任何学习逻辑**，打多少次都记不住。

**改法**：新增 `Coordinator::english_learn_target()`，把四处「上屏原码」出口
（空格 / 回车 / 标点顶屏 ×2）的学习目标从「活跃方案」改为按来源判定：

- 英文方案 → 落自己的桶（与上游一致）
- 码表方案且开着「英文候选」→ 落共享英文桶 `ENGLISH_SCHEMA`

**判据**（关键）：码表方案下字母串既可能是英文也可能是五笔码，不能见字母就学——

```
学 = 开着英文候选
     且 这一串没解出任何中文候选（CodeTable / Phrase）
     且 learnable()（纯 ASCII 字母数字、2~32 字符、至少一个字母）
```

- `workbuddy` 解不出中文 → 学 ✅
- `rrqn`（有效五笔码，有中文候选）→ 不学 ✅

涉及文件：

| 文件 | 改动 |
|---|---|
| `wind_input/crates/wind-engine/src/manager.rs` | 新增 `codetable_english_merge_enabled()` |
| `wind_input/crates/wind-coordinator/src/english_learn.rs` | 新增 `english_learn_target()` |
| `wind_input/crates/wind-coordinator/src/coordinator/message_handler.rs` | 4 处出口改用新判据 |

### 2. `fix(punct): 码表方案下英文候选不参与标点顶屏`

**问题**：码表方案下打 `work`，`english_merge` 会把 `workbuddy` 等前缀补全词混进候选
并排在首位。此时按句号，上游的「标点顶屏」逻辑会把高亮候选推上去，得到 `workbuddy.`
——但用户打的是 `work`，`workbuddy` 只是补全建议，不是他选的词。

**改法**：新增 `Coordinator::punct_top_skips_english()`。高亮候选来源是
`CandidateSource::English` 且活跃方案是码表时，跳过顶屏、改走「上屏原码」分支，
得到 `work.`；同时跳过顶屏的造词记账（那条候选没被选，不该记）。

| 场景 | 行为 |
|---|---|
| `work` + `.` | `work.` |
| 打全 `workbuddy` + `.` | `workbuddy.`（原码 == 候选文本，结果不变） |
| 中文码 `wq` + `.` | `你。`（高亮是中文，不受影响） |
| 拼音方案 | 完全不受影响（判据带 `is_codetable()` 门） |

### 3. `build: wind-installer 锁到 0250e9d7`

仅构建脚本改动，不影响运行逻辑。上游 `huanfeng/wind-installer` 在 `00425c92`
（2026-11-22）之后二进制目标编不过——`path_conflict` 只声明进 `lib.rs`，`main.rs`
缺 `mod path_conflict`，而 `src/ui/install_wizard.rs` 里写的是 `crate::path_conflict`，
编 bin 时 E0433。锁到最后一个可编译的提交，等上游修好再解锁。

## 已知限制

- **`learnable()` 挡掉带符号的串**：`C++`、`e-mail`、`Node.js` 不会被自动学。
  这是上游规则（英文方案码元只有字母），本分支沿用未放宽。
- **短英文词可能学不到**：若某个 2~4 字母串恰好是有效五笔码且有中文候选，
  判据会认为你在打中文。刻意的取舍，避免污染词库。
- **标点顶屏只改了一条通路**：`handle_key_event` 主标点通路已改；
  `commit_highlight_then_char`（numpad 句号、非码元字母）没有「上屏原码」分支，未接。
- **拼音方案未改**：拼音侧有独立的 `english_merge`，语义不同，本次没接。

## 英文后智能标点（无需本分支）

如果你只想要「英文词后按句号出英文句号」，**上游 0.131 起就已经支持**，
不需要这个分支：

```toml
[input.punct]
smart_after_latin = true        # 默认 false
latin_smart_list = "./_:@"      # 出厂值，含句号
```

## 构建

不在本机编译，走 GitHub Actions：

```bash
gh workflow run release.yml --repo weifeng-work/WindInput --ref main
gh run list --repo weifeng-work/WindInput --limit 3
gh run download <run-id> --repo weifeng-work/WindInput -n dist-windows -D dist
```

Windows 冷编译约 23 分钟，有缓存约 15 分钟。产物**未签名**，安装时 SmartScreen 会拦。

## 许可证

本分支与上游一致，采用 **GNU GPL-3.0**（见 [LICENSE](LICENSE)）。你可以自由使用、
修改、分发，但分发时必须同样以 GPL-3.0 提供源码并标注修改。

### 关于名称与图标

依据上游 [BRANDING.md](BRANDING.md)：GPL-3.0 第 7 条 (e) 款不授予使用项目名称与标识的
权利，`pic/logo.png` 等图形作品的著作权也不在 GPL 授权范围内。

因此**如果你想基于本分支公开发布自己的版本，请务必**：

1. 更换项目名称，不要用「清风输入法」「WindInput」或易混淆的近似名
2. 更换 logo 与应用程序图标
3. 保留本文件顶部的非官方分支声明

本项目目前**没有**发布任何 Release，只提供 Actions 构建产物供自己使用。
