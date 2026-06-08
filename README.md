# Codey：工具输出相关性剪枝版 Codex

Codey 是从 OpenAI Codex CLI 派生的非官方 fork，核心目标是降低长任务里“工具输出已经读过，但后续每轮请求仍继续携带”的上下文成本。它面向漏洞分析、仓库审计、大型代码排查、日志分析这类会反复执行 `rg`、`sed`、`nl`、`git show`、`curl`、测试命令的任务。

Codey 不是 OpenAI 官方项目，也没有得到 OpenAI 背书或赞助。OpenAI、Codex 及相关名称归各自权利方所有。本仓库保留上游 Apache-2.0 许可证和 attribution，见 [NOTICE](./NOTICE) 与 [MODIFICATIONS.md](./MODIFICATIONS.md)。

使用本 fork 连接 OpenAI 服务时，仍需要你自己的 OpenAI 账号或 API 凭据，并遵守 OpenAI 对应条款和使用政策。

## 为什么做这个

我在 Codex 中间加 hook 后发现，长任务真正烧 token 的地方不是最终回答，而是历史上下文里的工具输出回灌。

在一批漏洞分析任务中，51 个任务目录里有 43 个 `result.json` 带 usage 数据，累计消耗如下：

```text
total_tokens:              27,314,621
input_tokens:              26,951,677
cached_input_tokens:       22,635,904
output_tokens:                362,944
reasoning_output_tokens:      203,923
```

也就是说，约 98.7% 都是 input token。缓存命中很高，但缓存不等于免费解决上下文膨胀：请求仍然越来越大，调度也越来越重。

典型样本里，一个任务从第 1 次请求约 15,354 chars 增长到最后约 177,025 chars。固定开销本身也不小：

```text
instructions:   ~14,732 chars/request
tools schema:   ~20,111 chars/request
output schema:   ~2,500 chars/request
```

但跨任务看，最大的问题是工具输出。大范围 `rg`、`curl`、`git show`、`sed` 的输出进入历史后，后续每次模型请求都会继续携带。最大单任务样本达到：

```text
total_tokens:         1,828,513
input_tokens:         1,814,252
cached_input_tokens:  1,673,600
```

这和我原来的直觉不一样：原生 Codex 并不会自动把“已读过但不再需要的大工具输出”从后续请求里自然丢掉。

## Codey 的思路

Codey 选择的是“读完后由主模型标记可丢弃内容”的路线，而不是默认在工具返回前塞一个小模型做摘要。

流程是：

1. 工具第一次返回时，主模型仍然能看到完整输出。
2. 开启后会额外暴露内部工具 `trim_prompt_context`。
3. 主模型在确认某些历史工具输出后续不再需要时，调用 `trim_prompt_context` 标记它们。
4. 后续请求里，这些历史工具输出会被替换成短占位记录。
5. 占位记录保留工具名、`call_id`、可见 `Chunk ID`、命令、工作目录、可选 `output_ref` 和剪枝原因。

这样做的重点是：不提前截断证据，不改变命令执行结果，也不把质量完全交给小模型摘要。模型先读完整证据，再决定哪些历史输出可以从后续上下文里拿掉。

## 与基线对比

下面的数据来自本地漏洞可达性分析任务和 Codex 历史记录。不同 CVE 的任务难度和探索路径不同，数字不应理解为固定收益，但能说明这个方向相对原生基线的优势和边界。

### 条件式剪枝 vs 原生基线

| 样本             | 原生或历史基线 | Codey / 当前策略 | token 变化 | 工具调用变化                                    | 质量结论                               |
| ---------------- | -------------: | ---------------: | ---------: | ----------------------------------------------- | -------------------------------------- |
| `CVE-2026-25879` |        218,266 |          156,746 |     -28.2% | `16 exec + 2 write_stdin` -> `18 exec`          | 正文可达链正确；结构化行号仍有偏移     |
| `CVE-2026-9227`  |        385,203 |          243,647 |     -36.8% | `22 exec + 3 write_stdin` -> `24 exec + 1 trim` | 主要危险入口保留；少列一个辅助入口     |
| `CVE-2026-45131` |        259,457 |           83,356 |     -67.9% | `13 exec + 3 write_stdin` -> `6 exec`           | 正确定位危险 workflow；少列一个分支    |
| `CVE-2026-45288` |        373,288 |          185,856 |     -50.2% | `45 exec` -> `14 exec + 1 trim`                 | 主要 sink 和入口保留；少列一层包装入口 |

这组复测的共同点是：没有出现空结果退化，主要危险点和核心可达链仍在；代价是结果会更精简，部分等价入口、辅助入口或 wrapper 可能不再完整枚举。

### 高覆盖任务对比

`CVE-2026-49490` 的对照更能说明“覆盖率”和“成本”之间的关系：

| 版本             | total tokens | 工具数 | prune | execution_functions | outmost_functions |
| ---------------- | -----------: | -----: | ----: | ------------------: | ----------------: |
| 旧高覆盖基线     |    1,858,159 |     75 |     0 |                   3 |                 1 |
| 精简 prompt 版本 |      690,190 |     23 |     1 |                   1 |                 2 |
| 扩大覆盖复测版本 |    1,281,199 |     72 |     0 |                   4 |                 4 |

精简 prompt 版本 token 最低，但覆盖也明显收缩。扩大覆盖复测版本工具数接近旧高覆盖基线，结构化结果更多，同时仍比旧高覆盖基线少 576,960 tokens，约 31.0%。这说明收益不是简单来自“少做事”，而是减少了历史 payload 的重复携带和不必要调度。

### 为什么不是默认前置小模型压缩

我也做过“工具输出先交给小模型压缩，再回灌给主模型”的实验。结果说明：压缩本身有效，但如果 evidence packet 不够好，主模型会转而反复 `sed` / `nl` / `rg` 重新确认源码，最终反而更贵。

| 版本                                  | total tokens | shell 命令数 | 结果形态                  |
| ------------------------------------- | -----------: | -----------: | ------------------------- |
| 原生旧跑                              |      602,957 |           32 | `execution 2 / outmost 5` |
| 直接压缩版                            |      817,559 |           54 | `execution 3 / outmost 4` |
| 修正 `cmd + source_coverage` 的压缩版 |      517,119 |           26 | `execution 2 / outmost 5` |

这个对照给出的结论是：压缩链路不是不能做，但它需要更复杂的 evidence packet 设计。Codey 当前主线选择更小侵入的条件式历史剪枝：不强制每次工具后都开新轮 trim，也不默认依赖小模型摘要质量。

## 开启方式

在 `~/.codex/config.toml` 中加入：

```toml
[tool_output_relevance_pruning]
enabled = true
```

漏洞分析、仓库审计、批量排查这类长任务建议使用：

```toml
[tool_output_relevance_pruning]
enabled = true
apply_to = ["exec_command"]
threshold_tokens = 200
target_tokens = 140
timeout_ms = 8000
```

如果通过 `codex exec`、Docker runner 或其他独立环境运行，需要把配置写入对应环境的 `$CODEX_HOME/config.toml`。例如 runner 容器使用 `/root/.codex` 时，实际生效文件通常是容器内 `/root/.codex/config.toml`。

| 参数               |             默认值 | 说明                                                                           |
| ------------------ | -----------------: | ------------------------------------------------------------------------------ |
| `enabled`          |            `false` | 是否开启工具输出相关性剪枝。关闭时不暴露 `trim_prompt_context`，行为等同原版。 |
| `apply_to`         | `["exec_command"]` | 允许剪枝的工具名。当前主要面向 shell / unified exec 长输出。                   |
| `threshold_tokens` |              `200` | 大输出候选门槛。实际是否剪枝仍由模型给出的 keep/drop selector 决定。           |
| `target_tokens`    |              `140` | 剪枝后占位记录的目标大小。占位会保留定位信息，不追求复述原文。                 |
| `timeout_ms`       |             `8000` | 兼容保留字段；当前条件式主路径通常无需调整。                                   |
| `model`            |     `gpt-5.4-mini` | 兼容保留字段；当前主路径不依赖小模型压缩。                                     |

## `trim_prompt_context` 参数

开启后，模型可以调用内部工具 `trim_prompt_context`。参数含义如下：

| 参数            | 用法                                                                                            |
| --------------- | ----------------------------------------------------------------------------------------------- |
| `drop_call_ids` | 明确剪掉这些工具调用 ID 或可见 `Chunk ID` 对应的历史输出。                                      |
| `drop_commands` | 按命令片段匹配并剪掉历史输出；片段过短会被忽略。                                                |
| `keep_call_ids` | 明确保留这些工具调用 ID 或可见 `Chunk ID`。如果只提供 keep selector，其他可剪枝输出会成为候选。 |
| `keep_commands` | 按命令片段保留历史输出。如果只提供 keep selector，其他可剪枝输出会成为候选。                    |
| `reason`        | 简短说明剪枝原因，不包含隐藏推理。                                                              |

剪枝只作用在历史上下文层：工具输出已经在当前轮被模型看过；后续请求中，这段历史输出才会被短占位替代。如果替换文本不比原文短，Codey 会跳过该条剪枝。

## 适合的场景

- 漏洞可达性分析、供应链审计、批量仓库调查。
- 需要频繁执行 `rg`、`sed`、`nl`、`git show`、`curl`、测试命令的长流程。
- 工具输出里有大量一次性搜索结果，后续只需要保留文件、函数、行号、diff 或失败摘要。
- 单个任务会进入十几轮、几十轮模型请求，历史上下文明显膨胀。

## 不适合的场景

- 很短的单轮任务，或者工具输出本来就很小。
- 你希望完整工具输出一直留在后续上下文中反复引用。
- 你的主要成本来自当前轮首次接收工具输出。Codey 节省的是后续请求的历史上下文 token，不会减少当前轮工具回传 token。
- 需要尽可能完整枚举所有 secondary wrappers、辅助入口、等价入口的任务。当前策略更偏向保留 primary evidence。

## 如何确认生效

开启后可以观察两类迹象：

- 模型工具列表中出现 `trim_prompt_context`。
- rollout trace 中出现 `tool_output_relevance_pruning.snapshot` 事件，里面包含 `candidate_outputs`、`rewritten_outputs`、`original_token_estimate`、`replacement_token_estimate`、`saved_token_estimate` 等估算字段。

## 构建

```shell
git clone https://github.com/bilisheep/codey.git
cd codey/codex-rs
cargo build --release --bin codex
```

构建后的可执行文件在：

```text
codex-rs/target/release/codex
```

当前发布制品仍从上游 `codex` Rust binary target 构建。Release 包里可能同时提供 `codey` 主入口和兼容用 `codex` 二进制副本。

## 许可

本仓库基于 Apache License 2.0 分发。上游许可证见 [LICENSE](./LICENSE)，attribution 见 [NOTICE](./NOTICE)，Codey 相对上游的主要修改见 [MODIFICATIONS.md](./MODIFICATIONS.md)。
