# Codey

Codey is an unofficial fork derived from OpenAI Codex CLI. It focuses on
tool-output relevance pruning for long local coding sessions, repository audits,
and other workflows where large tool outputs are useful once but expensive to
carry in every later model request.

**Unofficial fork notice:** Codey is not affiliated with, endorsed by, or
sponsored by OpenAI. OpenAI, Codex, and related names remain the property of
their respective owners. This repository preserves upstream Apache-2.0 license
and notice files; see [NOTICE](./NOTICE) and [MODIFICATIONS.md](./MODIFICATIONS.md).

Using OpenAI services through this fork requires your own OpenAI account or API
credentials and remains subject to OpenAI's applicable terms and usage policies.

## Quickstart

Download Codey builds from the
[bilisheep/codey releases page](https://github.com/bilisheep/codey/releases).
Release archives are named `codey-<tag>-<platform>` and include the CLI binary
plus a short release README. The native binary is still built from the upstream
`codex` binary target for compatibility with the existing Rust workspace.

Build from source:

```shell
git clone https://github.com/bilisheep/codey.git
cd codey/codex-rs
cargo build --release --bin codex
```

Then run the built executable at `target/release/codex`.

### 本分支特性：工具输出相关性剪枝

这个分支是 `bilisheep/tool-output-relevance-pruning` 的实验版本，核心目标是降低长任务中
“已经读过的大型工具输出”在后续模型请求里反复回灌造成的 token 成本。它默认关闭，开启后不改变
命令执行结果，也不会在工具第一次返回时裁剪输出；主模型仍会先完整看到本轮工具输出，然后在判断某些
输出后续不再需要时调用 `trim_prompt_context`，把这些历史工具输出替换成短占位记录。

剪枝后的占位记录会保留基本追溯信息，例如工具名、`call_id`、可见 `Chunk ID`、命令、工作目录、
可选 `output_ref` 和剪枝策略说明。这样后续上下文不再携带完整原文，但仍能知道被剪掉的内容来自哪条
命令；如果之后确实还需要细节，模型应重新执行更窄的命令获取证据。

#### 如何开启

在 `~/.codex/config.toml` 中加入：

```toml
[tool_output_relevance_pruning]
enabled = true
```

长时间批处理、仓库审计或漏洞可达性分析建议使用下面这组保守配置：

```toml
[tool_output_relevance_pruning]
enabled = true
apply_to = ["exec_command"]
target_tokens = 140
```

如果你通过 `codex exec` 或容器 runner 使用单独的 `CODEX_HOME`，需要把同样的配置写入该环境下的
`$CODEX_HOME/config.toml`。例如 Docker runner 挂载 `/root/.codex` 时，实际生效文件通常是容器内
`/root/.codex/config.toml`，对应宿主机的挂载目录。

#### 配置参数

```toml
[tool_output_relevance_pruning]
enabled = true
apply_to = ["exec_command"]
target_tokens = 140
```

| 参数            | 默认值             | 当前含义                                                                                 |
| --------------- | ------------------ | ---------------------------------------------------------------------------------------- |
| `enabled`       | `false`            | 是否开启工具输出相关性剪枝。关闭时不会暴露 `trim_prompt_context`，行为等同原版 Codex。   |
| `apply_to`      | `["exec_command"]` | 允许被剪枝的工具名。当前分支主要面向 shell / unified exec 输出，建议保持默认。           |
| `target_tokens` | `140`              | 剪枝后占位记录的目标大小。当前实现会生成很短的追溯占位，不建议为了追求更详细占位而调大。 |

#### `trim_prompt_context` 如何工作

开启后，Codex 会向主模型额外暴露一个内部工具：`trim_prompt_context`。模型在读完一批非
`trim_prompt_context` 工具输出后，只有当至少一个已经消费过的大输出后续不再需要时才应调用它。
工具参数包括：

| 参数            | 用法                                                                                                |
| --------------- | --------------------------------------------------------------------------------------------------- |
| `drop_call_ids` | 明确剪掉这些工具调用 ID 或可见 `Chunk ID` 对应的历史输出。                                          |
| `drop_commands` | 按命令片段匹配并剪掉历史输出，片段过短会被忽略。                                                    |
| `keep_call_ids` | 明确保留这些工具调用 ID 或可见 `Chunk ID`。如果只提供 keep selector，其他可剪枝输出会成为剪枝候选。 |
| `keep_commands` | 按命令片段保留历史输出。如果只提供 keep selector，其他可剪枝输出会成为剪枝候选。                    |
| `reason`        | 简短说明剪枝原因，不包含隐藏推理。                                                                  |

剪枝只会发生在历史上下文层：原工具输出已经在当前轮被模型看过；后续请求中，这段历史输出会被短占位替代。
如果替换文本并不比原文更短，Codex 会跳过该条剪枝，避免无效改写。

#### 适合开启的场景

- 批处理漏洞可达性分析，尤其是需要反复执行 `rg`、`sed`、`nl`、`git show`、测试命令的任务。
- 大仓库排查、日志分析、构建失败定位等长流程任务。
- 工具输出里有大量一次性搜索结果，模型只需要保留少量文件、函数、行号、diff 或失败摘要。

#### 不适合或收益有限的场景

- 很短的单轮任务，或者工具输出本来就很小。
- 你希望完整工具输出一直留在后续上下文中供模型反复引用。
- 你的主要成本来自当前轮首次接收工具输出。这个分支节省的是后续请求的历史上下文 token，不是当前轮工具回传 token。
- 模型没有调用 `trim_prompt_context`，或者它判断所有工具输出仍是证据时，不会产生剪枝收益。

#### 如何确认生效

开启后可以通过两类迹象确认：

- 模型工具列表中会出现 `trim_prompt_context`。
- rollout trace 中会出现 `tool_output_relevance_pruning.snapshot` 事件，事件里包含
  `candidate_outputs`、`rewritten_outputs`、`original_token_estimate`、
  `replacement_token_estimate` 和 `saved_token_estimate` 等估算字段。

本分支的预发布制品当前覆盖 Linux x64、Linux arm64、Windows x64、Windows arm64 和 macOS arm64。
不发布 32 位 x86，也不发布 macOS x64 制品；Intel Mac 需要本机自行编译。

### Using Codex with your ChatGPT plan

Run `codex` and select **Sign in with ChatGPT**. We recommend signing into your ChatGPT account to use Codex as part of your Plus, Pro, Business, Edu, or Enterprise plan. [Learn more about what's included in your ChatGPT plan](https://help.openai.com/en/articles/11369540-codex-in-chatgpt).

You can also use Codex with an API key, but this requires [additional setup](https://developers.openai.com/codex/auth#sign-in-with-an-api-key).

## Docs

- [**Codex Documentation**](https://developers.openai.com/codex)
- [**Contributing**](./docs/contributing.md)
- [**Installing & building**](./docs/install.md)
- [**Open source fund**](./docs/open-source-fund.md)

This repository is licensed under the [Apache-2.0 License](LICENSE).
