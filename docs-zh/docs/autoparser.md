# Auto-Parser 架构

Auto-parser 自动分析聊天模板，以确定如何解析模型输出，包括内容、推理过程和工具调用。

## 概述

统一的 auto-parser 采用纯差分、组合式的方法（灵感来自 `git diff` 算法）来分析聊天模板：

**核心思想**：

- **最小化硬编码模式**：所有标记都通过模板对比提取（唯一的启发式规则是 JSON 检测，用于区分 `JSON_NATIVE` 和基于标签的格式）
- **组合式架构**：分别为推理、内容和工具提供独立的分析器结构，每个结构负责自身的分析和解析器构建

**分析与解析器构建分为两步**：

1. `autoparser::autoparser tmpl_analysis(tmpl)` — 运行所有差分对比，并填充分析结构
2. `autoparser::peg_generator::generate_parser(tmpl, generation_params, tmpl_analysis)` — 使用分析结果构建 PEG 解析器和可选的 GBNF 语法

## 数据结构

所有结构体均定义在 [common/chat-auto-parser.h](common/chat-auto-parser.h) 中。

### 顶层：`autoparser`（主分析器和生成器）

[common/chat-auto-parser.h:367-388](common/chat-auto-parser.h#L367-L388) — 顶层分析结果，聚合了 `jinja_caps`、`reasoning`、`content` 和 `tools` 子分析结果，以及 `preserved_tokens`（所有非空标记的并集）。

### `analyze_reasoning`

[common/chat-auto-parser.h:254-274](common/chat-auto-parser.h#L254-L274) — 推理分析结果：`mode` 枚举、`start` 标记（例如 `<think>`）和 `end` 标记（例如 `</think>`）。

### `analyze_content`

[common/chat-auto-parser.h:280-295](common/chat-auto-parser.h#L280-L295) — 内容分析结果：`mode` 枚举、`start`/`end` 标记，以及 `requires_nonnull_content` 标志。

### `analyze_tools` 及其子结构

- [common/chat-auto-parser.h:176-194](common/chat-auto-parser.h#L176-L194) — `tool_format_analysis`：`mode` 枚举、`section_start/end`、`per_call_start/end`、JSON 字段名（`function_field`、`name_field`、`args_field`、`id_field`、`gen_id_field`）以及格式标志（`fun_name_is_key`、`tools_array_wrapped`）
- [common/chat-auto-parser.h:196-200](common/chat-auto-parser.h#L196-L200) — `tool_function_analysis`：函数名前后的 `name_prefix`、`name_suffix`、`close` 标记
- [common/chat-auto-parser.h:202-210](common/chat-auto-parser.h#L202-L210) — `tool_arguments_analysis`：参数容器的 `start/end` 标记、`name_prefix/suffix`、`value_prefix/suffix`、`separator`
- [common/chat-auto-parser.h:212-217](common/chat-auto-parser.h#L212-L217) — `tool_id_analysis`：调用 ID 值前后标记的 `pos` 枚举、`prefix`/`suffix`
- [common/chat-auto-parser.h:301-361](common/chat-auto-parser.h#L301-L361) — `analyze_tools`：聚合上述四个子结构

### 枚举

**`reasoning_mode`**：模板处理推理/思考块的方式。

| 值              | 说明                                                                      |
|-----------------|---------------------------------------------------------------------------|
| `NONE`          | 未检测到推理标记                                                          |
| `TAG_BASED`     | 基于标签：`<think>...</think>`（对于分隔符式格式，start 可为空）          |
| `TOOLS_ONLY`    | 推理仅出现在工具调用响应中，而非普通内容中                                |

**生成提示与推理预填充**：在调用专用处理程序或 auto-parser 之前，由 `common_chat_templates_apply_jinja` 通过两次渲染模板计算得出 —— 一次使用 `add_generation_prompt=false`，一次使用 `add_generation_prompt=true` —— 并将差异后缀存储为 `generation_params::generation_prompt`。该字符串会传递至 `common_chat_params::generation_prompt` 和 `common_chat_parser_params::generation_prompt`。

生成提示会在 PEG 解析之前通过 `wrap_for_generation_prompt()` 拼接到模型输出前。推理开始标记之前的部分会作为字面量前缀拼接，以确保模板添加的任何样板内容都被消耗。完整字符串也会通过 `llama_sampler_accept` 传递给语法采样器（存储在 `common_params_sampling::grammar_prefill` 中），使语法越过提示中已存在的 token。它用于确定推理预算采样器的初始状态 —— 如果预填充 token 以推理开始序列开头（但不包含结束序列），则为 COUNTING，否则为 IDLE。

**`grammar_prefill`**（`common_params_sampling`）：生成提示字符串在语法采样器初始化时被分词并接受。仅当 `grammar_external` 为 false 时应用（即语法未由用户显式设置）。

`generate_parser()` 中处理推理预填充的三种结果：

1. **生成提示中同时包含开始和结束标记**（例如 `<think></think>\n`）：解析器认为推理已打开并立即关闭；仅包含空白字符的推理内容会被丢弃。
2. **生成提示中仅包含开始标记**（例如 `<think>\n`）：解析器认为推理已处于打开状态。
3. **开始标记存在但不在末尾**（例如 Apriel 的 `<|begin_assistant|>` 后跟样板内容）：该标记是模板产物；开始字面量被清除，使推理使用分隔符式（仅结束标记）。对于忽略 `add_generation_prompt` 的模板（差异为空），将使用渲染后的 `data.prompt` 作为回退 —— 但仅适用于非 TOOLS_ONLY 模式，因为在 TOOLS_ONLY 模式下开始标签由模型生成，可能出现在先前的对话轮次中。

**`content_mode`**：模板包装助手内容的方式。

| 值                       | 说明                                                    |
|--------------------------|---------------------------------------------------------|
| `PLAIN`                  | 无内容标记                                              |
| `ALWAYS_WRAPPED`         | 内容始终被包装：`<response>...</response>`              |
| `WRAPPED_WITH_REASONING` | 仅在存在推理时才包装内容                                |

**`tool_format`**：工具调用结构的分类。

| 值              | 说明                                                      |
|-----------------|-----------------------------------------------------------|
| `NONE`          | 未检测到工具支持                                          |
| `JSON_NATIVE`   | 纯 JSON：`{"name": "X", "arguments": {...}}`              |
| `TAG_WITH_JSON` | 基于标签且参数为 JSON：`<function=X>{...}</function>`     |
| `TAG_WITH_TAGGED`| 基于标签且参数也为标签：`<param=key>value</param>`       |

**`call_id_position`**：基于标签的格式中调用 ID 的位置。

| 值                      | 说明                              |
|-------------------------|-----------------------------------|
| `NONE`                  | 未检测到调用 ID 支持              |
| `PRE_FUNC_NAME`         | 函数名之前                        |
| `BETWEEN_FUNC_AND_ARGS` | 函数名与参数之间                  |
| `POST_ARGS`             | 参数之后                          |

## 工具调用格式

### JSON_NATIVE

**结构**：整个工具调用（函数名、参数、值）均为 JSON 格式。可选的节包装标签。

**检测方式**：函数名出现在 JSON 结构中（引号前为 `{` 或 `:`）。

**示例**：

标准 OpenAI 风格：

```json
<tool_call>
{"name": "get_weather", "arguments": {"location": "Paris", "unit": "celsius"}}
</tool_call>
```

Mistral Nemo 带数组包装：

```json
[TOOL_CALLS]
[{"name": "calculate", "arguments": {"expr": "2+2"}}]
```

函数名作为 JSON 键（Apertus 风格）：

```json
{"get_weather": {"location": "Paris"}}
```

---

### TAG_WITH_JSON

**结构**：函数名位于 JSON 之外，在标签属性或 XML 风格标签中。参数是 JSON 对象。

**检测方式**：函数名不在 JSON 中，但参数名出现在 JSON 上下文中。

**示例**：

Functionary v3.1：

```xml
<function=get_weather>{"location": "Paris", "unit": "celsius"}</function>
```

MiniMax：

```xml
<minimax:tool_call>
<tool_name>calculate</tool_name>
<arguments>{"expr": "2+2"}</arguments>
</minimax:tool_call>
```

---

### TAG_WITH_TAGGED

**结构**：函数名和参数名都在 XML 风格标签中。字符串值不加引号；非字符串值使用 JSON 格式。

**检测方式**：函数名和参数名都不出现在 JSON 上下文中。

**示例**：

Qwen/Hermes XML 格式：

```xml
<function=get_weather>
<param=location>Paris</param>
<param=unit>celsius</param>
</function>
```

混合类型：

```xml
<function=calculate>
<param=expr>2+2</param>
<param=precision>2</param>
<param=options>{"round": true}</param>
</function>
```

字符串值（`Paris`、`celsius`、`2+2`）不加引号；`options`（对象类型）使用 JSON 格式。

---

## 分析流程

```text
autoparser::autoparser(tmpl)
    |
    |-- Phase 1: analyze_reasoning(tmpl, jinja_caps.supports_tool_calls)
    |     |-- R1: compare_reasoning_presence()   — 带/不带 reasoning_content 字段
    |     |-- R2: compare_thinking_enabled()     — enable_thinking=false 与 true
    |     '-- R3: compare_reasoning_scope()      — reasoning+content 与 reasoning+tools
    |           (仅在 supports_tool_calls 时执行)
    |
    |-- Phase 2: analyze_content(tmpl, reasoning)
    |     '-- C1: 对比仅内容输出与工具输出，以及仅内容输出与推理输出
    |
    |-- Phase 3: analyze_tools(tmpl, jinja_caps, reasoning)
    |     (当 !jinja_caps.supports_tool_calls 时完全跳过)
    |     |
    |     |-- T1: analyze_tool_calls()           — 无工具与有工具输出；分类格式
    |     |         |-- JSON 路径 → analyze_tool_call_format_json_native()
    |     |         '-- 标签路径 → analyze_tool_call_format_non_json()
    |     |
    |     (当 format != NONE 且 format != JSON_NATIVE 时：)
    |     |
    |     |-- T2: check_per_call_markers()       — 1 次调用与 2 次调用；必要时将 section 移到 per-call
    |     |         (仅在 supports_parallel_tool_calls 时执行)
    |     |
    |     |-- T3: extract_function_markers()     — func_alpha 与 func_beta；提取 name prefix/suffix/close
    |     |
    |     |-- T4: analyze_arguments()            — (仅 TAG_WITH_TAGGED)
    |     |         |-- A1: extract_argument_name_markers()   — arg_name_A 与 arg_name_B
    |     |         '-- A2: extract_argument_value_markers()  — 值 "XXXX" 与 "YYYY"
    |     |
    |     |-- T5: extract_argument_separator()   — 1 个参数与 2 个参数；查找参数间分隔符
    |     |
    |     |-- T6: extract_args_markers()         — 0 个参数与 1 个参数；查找参数容器标记
    |     |
    |     '-- T7: extract_call_id_markers()      — 调用 ID "call00001" 与 "call99999"
    |
    '-- collect_preserved_tokens()               — 所有非空标记的并集
    |
    '-- apply workarounds()                      — 针对边缘情况模板的事后补丁
    |
    v
autoparser (analysis result)
    |
    v
autoparser::peg_generator::generate_parser(tmpl, inputs, analysis)
    |-- analysis.build_parser(inputs)            — 构建 PEG 解析器 arena
    |     |-- reasoning.build_parser(ctx)        — 推理解析器（模式相关）
    |     |-- content.build_parser(ctx)          — 内容解析器（模式相关）
    |     '-- tools.build_parser(ctx)            — 工具解析器（按 tool_format 分发）
    |           |-- build_tool_parser_json_native()
    |           |-- build_tool_parser_tag_json()
    |           '-- build_tool_parser_tag_tagged()
    |
    |-- 构建 GBNF 语法（如果存在工具且 trigger_marker 非空）
    '-- 从 section_start 或 per_call_start 设置 grammar_triggers
    |
    v
common_chat_params (prompt, parser, grammar, triggers, preserved_tokens)
```

## 入口点

Auto-parser 在 [common/chat.cpp:1280-1310](common/chat.cpp#L1280-L1310) 的 `common_chat_templates_apply_jinja` 中被调用。首先处理少量专用模板（Ministral/Magistral Large 3、带 `<|channel|>` 的 GPT-OSS、带 `>>>all` 的 Functionary v3.2），然后 auto-parser 通过 `autoparser::autoparser` + `peg_generator::generate_parser` 处理其余所有情况。

## 算法细节

### 核心机制：差分对比

所有分析阶段都使用同一个因式化的对比函数，声明于 [common/chat-auto-parser-helpers.h:68](common/chat-auto-parser-helpers.h#L68)：

```cpp
compare_variants(tmpl, params_A, params_modifier)
```

该函数通过对 `params_A` 的副本应用修改器 lambda 创建变体 B，将两者通过模板渲染，并计算一个 `diff_split`（[common/chat-auto-parser.h:28-37](common/chat-auto-parser.h#L28-L37)）：

- `prefix` — A 与 B 的公共前缀
- `suffix` — A 与 B 的公共后缀
- `left` — 变体 A 独有
- `right` — 变体 B 独有

差分通过 `calculate_diff_split()` 计算，该函数先找出最长公共前缀和最长公共后缀，然后迭代地将不完整的 `<...>` 或 `[...]` 标记从前缀/后缀移入 left/right，直到稳定（标签边界修复）。

文本通过 `segmentize_markers()` 切分为标记和非标记片段，按 `<...>` 和 `[...]` 边界分割。

### Phase 1：推理分析

**R1 — `compare_reasoning_presence()`**：对比带 `reasoning_content` 字段的助手消息与不带该字段的情况。

- 在 `diff.right`（带推理的输出）中搜索推理内容 needle
- 使用 PEG 解析器查找周围标记：
  - 如果在 `diff.right` 中同时找到前/后标记 → `TAG_BASED`
  - 如果两者都找到但后标记仅在完整输出 B 中 → `TAG_BASED`（模板强制添加标记；通过预填充处理）
  - 如果仅找到后标记 → `TAG_BASED`（分隔符式，空 start）
- 设置 `reasoning.start` 和 `reasoning.end`

**R2 — `compare_thinking_enabled()`**：对比生成提示下 `enable_thinking=false` 与 `true` 的情况。

- 检测模板添加的推理标记：`enable_thinking=true` 追加了非空标记 → 设置 `reasoning.start`，mode = `TAG_BASED`
- 处理反向情况（`enable_thinking=false` 反而追加了标记）：提取开始（来自前一段）和结束标记；mode = `TAG_BASED`
- 模板添加的推理预填充稍后在 `common_chat_templates_apply_jinja` 中提取，并在解析前拼接到模型输出

**R3 — `compare_reasoning_scope()`**：对比带推理+文本内容与带推理+工具调用的助手消息。

- 仅在 `jinja_caps.supports_tool_calls` 时运行
- 检测 `TOOLS_ONLY`：推理内容存在于 B（带工具）中但不存在于 A（带文本内容）中
- 使用 PEG 解析器从工具调用输出中提取推理标记

### Phase 2：内容分析

**C1**：`analyze_content` 构造函数中的两次对比：

- 对比 1：仅内容输出与工具调用输出 → `diff_tools`
- 对比 2：仅内容输出与推理+空内容输出 → `diff_reasoning`

分类逻辑：

- `PLAIN`：`diff_tools.left` 等于响应字符串（内容就是整个差异，无包装）
- `ALWAYS_WRAPPED`：在 `pure_content` 中找到围绕内容文本的标记 → 提取 `start`/`end`

### Phase 3：工具调用分析

**T1 — `analyze_tool_calls()`**：对比无工具输出与有工具输出。

- 将工具调用节提取为 `diff.right`
- 调用 `analyze_tool_call_format()`，它先从被搜索文本中剥离推理标记，然后：
  - 对函数名和参数名 needle 都调用 `in_json_haystack()`
  - `in_json_haystack()` 使用 PEG 解析器检查 needle 是否出现在 JSON 上下文中（前面是 `{` 或 `:` 且有引号包围）
  - 如果函数名在 JSON 中 → `JSON_NATIVE` → `analyze_tool_call_format_json_native()`
  - 如果函数名不在 JSON 中但参数名在 JSON 中 → `TAG_WITH_JSON`
  - 如果两者都不在 JSON 中 → `TAG_WITH_TAGGED`
  - `analyze_tool_call_format_json_native()`：解析 JSON 对象，将字段值与 needle 匹配以填充 `name_field`、`args_field`、`id_field`、`gen_id_field`；检测 `tools_array_wrapped`；提取 `section_start`/`section_end`
  - `analyze_tool_call_format_non_json()`：在被搜索文本上使用 PEG 解析器查找最多两个开始标记（section + per-call），然后查找最多两个结束标记

**T2 — `check_per_call_markers()`**：对比 1 次调用与 2 次调用。

- 对第二次调用部分与公共后缀进行二次差分
- 如果第二次调用内容以 `section_start` 开头 → section 标记实际上是 per-call → 将 `section_start/end` 移至 `per_call_start/end` 并清空 section 标记

**T3 — `extract_function_markers()`**：对比函数名 `FUN_FIRST` 与 `FUN_SECOND`（两个不同命名函数）。

- 在 `diff.left` 中定位函数名出现位置
- 从公共前缀中提取 `function.name_prefix`，从名称后到下一个标记提取 `function.name_suffix`
- 将 `name_suffix` 延伸至 `diff.suffix`（TAG_WITH_TAGGED 到第一个标记；TAG_WITH_JSON 到第一个 `{` 或 `[`）
- 从最后一个参数值之后到 per-call/section 结束标记提取 `function.close`

**T4 — `analyze_arguments()`**（仅 TAG_WITH_TAGGED）：

- **A1 `extract_argument_name_markers()`**：对比 `arg_name_A` 与 `arg_name_B`（两个不同参数名）。
  - 找到共享的包围结构 → `arguments.name_prefix`、`arguments.name_suffix`
- **A2 `extract_argument_value_markers()`**：对比参数值 `"XXXX"` 与 `"YYYY"`（同一参数，不同值）。
  - 找到值周围的标记 → `arguments.value_prefix`、`arguments.value_suffix`

**T5 — `extract_argument_separator()`**：对比 1 个参数与 2 个参数（同一函数）。

- 使用 `until_common_prefix(diff.right, ARG_FIRST, ARG_SECOND)` 查找两个参数块之间的分隔内容

**T6 — `extract_args_markers()`**：对比 0 个参数与 1 个参数。

- 使用 `until_common_prefix()` 和 `after_common_suffix()`，以空和单参数 JSON 字符串为锚点，查找容器标记（`arguments.start`、`arguments.end`）

**T7 — `extract_call_id_markers()`**：对比调用 ID `"call00001"` 与 `"call99999"`。

- 根据函数名出现在 `diff.prefix` 或 `diff.suffix` 来分类位置：
  - 函数名仅在 prefix 中 → `BETWEEN_FUNC_AND_ARGS` 或 `POST_ARGS`（进一步通过 `{` 出现位置区分）
  - 函数名仅在 suffix 中 → `PRE_FUNC_NAME`
- 提取调用 ID 值周围的 `call_id.prefix` 和 `call_id.suffix` 标记
- 如果 `per_call_end` 错误地包含了调用 ID 后缀，则将其清空

### Workarounds

`common/chat-diff-analyzer.cpp` 中的 workarounds 数组在分析后应用事后补丁。每个 workaround 是一个 lambda，检查模板源码并覆盖分析结果。当前的 workaround：

1. **旧版 Qwen/DeepSeek thinking 模板** — 源码包含 `content.split('</think>')` 但不包含 `<SPECIAL_12>`：如果未检测到推理，则设置 `reasoning.mode = TAG_BASED`，并指定 `<think>`/`</think>` 标记
2. **Granite 3.3** — 源码包含特定的 "Write your thoughts" 文本：强制 `<think>`/`</think>` 的 `TAG_BASED` 推理，以及 `<response>`/`</response>` 的 `WRAPPED_WITH_REASONING` 内容
3. **Cohere Command R+** — 源码包含 `<|CHATBOT_TOKEN|>`：如果尚未设置 content start，则设置 `ALWAYS_WRAPPED` 内容模式
4. **Functionary 3.1** — 源码包含 `set has_code_interpreter`：强制 `PLAIN` 内容、特定的 `per_call_start/end`，并将 preserved tokens 清空为仅保留 Functionary 专用标记
5. **DeepSeek-R1-Distill-Qwen** — 源码包含 `tool▁calls▁begin` 标记：用正确的 Unicode 方块字符覆盖工具节/per-call 标记

### 解析器构建

每个分析器结构体（`analyze_reasoning`、`analyze_content`、`analyze_tools`）都实现了 `build_parser(parser_build_context&)`。它们共享一个 `parser_build_context`，其中包含 PEG 构建器、推理输入、预构建的推理解析器，以及指向内容分析器的指针。

#### 推理解析器（`analyze_reasoning::build_parser`）

| 模式                                          | 解析器                                                                    |
|-----------------------------------------------|---------------------------------------------------------------------------|
| 不提取推理                                    | `eps()`                                                                   |
| `TAG_BASED` 或 `TOOLS_ONLY`（非空 start）     | `optional(start + reasoning(until(end)) + end + space())`                 |
| `TAG_BASED` 或 `TOOLS_ONLY`（空 start）       | `optional(reasoning(until(end)) + end + space())` — 分隔符式              |

注意：start 标记为空可能是因为分析器检测到了分隔符式推理，也可能是因为 `generate_parser()` 清除了模板产物 start 标记（参见上文“生成提示与推理预填充”）。仅包含空白字符的推理内容（例如来自 `<think></think>` 预填充）会被 mapper 丢弃。

#### 内容解析器（`analyze_content::build_parser`）

| 条件                                       | 解析器                                                                          |
|--------------------------------------------|---------------------------------------------------------------------------------|
| 存在 `json_schema`                         | `reasoning + space() + content(schema(json(), "response-format", ...)) + end()` |
| 存在工具                                   | 分派到 `analyze_tools::build_parser()`                                          |
| `ALWAYS_WRAPPED` 且带推理                  | `reasoning + start + content(until(end)) + end + end()`                         |
| `ALWAYS_WRAPPED` 且不带推理                | `content(until(start)) + start + content(until(end)) + end + end()`             |
| 默认（PLAIN）                              | `reasoning + content(rest()) + end()`                                           |

#### 工具解析器（`analyze_tools::build_parser`）

按 `format.mode` 分发：

**`build_tool_parser_json_native()`**：调用 `p.standard_json_tools()`，其内部分派到：

- `build_json_tools_function_is_key()` — 函数名是 JSON 键：`{"get_weather": {...}}`
- `build_json_tools_nested_keys()` — 嵌套：`{"function": {"name": "X", "arguments": {...}}}`
- `build_json_tools_flat_keys()` — 平铺：`{"name": "X", "arguments": {...}}`

处理内容包装器、数组包装（`tools_array_wrapped`）、并行调用和 `parameter_order`。

**`build_tool_parser_tag_json()`**：对每个工具函数：

```text
tool_open(name_prefix + tool_name(literal(name)) + name_suffix) +
    call_id_section +
    tool_args(schema(json(), tool_schema))
  [+ function.close if non-empty]
```

包装在 per-call 标记中（可选并行调用重复），然后可选地包装在 section 标记中。

**`build_tool_parser_tag_tagged()`**：对每个工具函数，为每个参数构建一个解析器：

- 字符串类型：`tool_arg_string_value(schema(until(value_suffix), ...))`
- JSON 类型：`tool_arg_json_value(schema(json(), ...))`
- 必需参数为普通解析器；可选参数包装在 `optional()` 中
- 连续参数之间用 `space()` 连接

对于关闭：如果存在 `function.close` 则使用它；否则使用 `peek(per_call_end)` 避免在部分流式传输期间过早关闭；最后回退到 `tool_close(space())` 以触发 mapper 回调。

三个工具解析器都返回：

```text
reasoning + optional(content(until(trigger_marker))) + tool_calls + end()
```

每个返回的解析器都会由 `wrap_for_generation_prompt()` 包装，它会为生成提示的样板前缀（推理开始标记之前的部分）拼接一个字面量。

## Mapper

`common_chat_peg_mapper` 将 PEG 解析结果（AST 节点）映射为 `common_chat_msg` 结构。关键设计：

- **缓冲参数**：在 `tool_name` 确定之前，参数文本进入 `args_buffer`；一旦名称设置完成，缓冲区被刷新到 `current_tool->arguments`
- **`args_target()`**：返回当前活动目标的引用（缓冲区或工具参数），消除分支
- **`closing_quote_pending`**：跟踪字符串参数值最终化时是否需要追加闭合 `"`（用于标签格式中模式声明的字符串类型）
- **仅空白字符的推理**：完全由空白字符组成的推理内容（例如来自 `<think></think>` 预填充）会被清空，使消息不显示推理
- **大括号自动闭合**：在工具关闭时，未闭合的 `{` 大括号会自动闭合

## 文件

| 文件                                      | 说明                                                                            |
|-------------------------------------------|---------------------------------------------------------------------------------|
| `common/chat-auto-parser.h`               | 所有分析结构体、枚举、`autoparser`、`peg_generator`、`generation_params`        |
| `common/chat-auto-parser-generator.cpp`   | 解析器生成器：`generate_parser()` 和 `build_parser()` 方法                      |
| `common/chat-diff-analyzer.cpp`           | 差分分析实现和 workaround                                                       |
| `common/chat-auto-parser-helpers.h/cpp`   | `calculate_diff_split()`、`segmentize_markers()`、`compare_variants()`、        |
|                                           | `wrap_for_generation_prompt()`、字符串辅助函数                                  |
| `common/chat-peg-parser.h/cpp`            | `common_chat_peg_builder`、`common_chat_peg_mapper` 及辅助函数                  |
| `common/chat.cpp`                         | 入口点：`common_chat_templates_apply_jinja()`                                   |
| `tools/parser/debug-template-parser.cpp`  | 模板分析调试工具                                                                |
| `tools/parser/template-analysis.cpp`      | 模板分析工具                                                                    |

## 测试与调试

### 调试工具

**模板调试器**：`tools/parser/debug-template-parser.cpp`

- 用法：`./bin/llama-debug-template-parser path/to/template.jinja`
- 显示检测到的格式、标记、生成的解析器和 GBNF 语法

**模板分析**：`tools/parser/template-analysis.cpp`

- 用法：`./bin/llama-template-analysis path/to/template.jinja`

**调试日志**：通过 `LLAMA_ARG_LOG_VERBOSITY=2` 启用

- 显示详细的分析步骤、模式提取结果和生成的解析器结构

**PEG 测试构建器**：用于创建测试用例的流式 API —— 参见 [tests/test-chat.cpp:947-1043](tests/test-chat.cpp#L947-L1043)。用法示例：

```cpp
auto tst = peg_tester("models/templates/Template.jinja");
tst.test("input text")
   .reasoning_format(COMMON_REASONING_FORMAT_AUTO)
   .tools({tool_json})
   .parallel_tool_calls(true)
   .enable_thinking(true)
   .expect(expected_message)
   .run();
```

### 已测试模板

以下模板在 `tests/test-chat.cpp` 中有活跃测试：

| 模板 | 格式 | 说明 |
| -------- | ------ | ----- |
| Ministral-3-14B-Reasoning | Reasoning | `[THINK]...[/THINK]` 标签（专用处理程序） |
| NVIDIA-Nemotron-3-Nano-30B | TAG_WITH_TAGGED | 推理 + 工具 |
| CohereForAI Command-R7B | JSON_NATIVE | `<\|START_THINKING\|>`/`<\|START_RESPONSE\|>` 标记 |
| Google Gemma 2 2B | Content only | 无工具支持 |
| Qwen-QwQ-32B | Reasoning | 强制开放式思考 |
| NousResearch Hermes 2 Pro | JSON_NATIVE | `<tool_call>` 包装器 |
| IBM Granite 3.3 | JSON_NATIVE | `<think></think>` + `<response></response>` |
| IBM Granite 4.0 | JSON_NATIVE | `<tool_call>` 包装器（4.1 使用同一模板） |
| ByteDance Seed-OSS | TAG_WITH_TAGGED | 自定义 `<seed:think>` 和 `<seed:tool_call>` 标签 |
| Qwen3-Coder | TAG_WITH_TAGGED | XML 风格工具格式 |
| DeepSeek V3.1 | JSON_NATIVE | 强制思考模式 |
| GLM-4.6 | TAG_WITH_TAGGED | `<tool_call>name\n<arg_key>...<arg_value>...` 格式 |
| GLM-4.7-Flash | TAG_WITH_TAGGED | 更新的 GLM 格式 |
| Kimi-K2-Thinking | JSON_NATIVE | 推理 + JSON 工具 |
| Apertus-8B-Instruct | JSON_NATIVE | 函数名作为 JSON 键 |
| MiniMax-M2 | TAG_WITH_JSON | 带 JSON 参数的 XML invoke |
| NVIDIA-Nemotron-Nano-v2 | JSON_NATIVE | `<TOOLCALL>` 包装器（嵌套） |
| CohereForAI Command-R Plus | JSON_NATIVE | Markdown 代码块格式 |
| Mistral-Nemo-Instruct-2407 | JSON_NATIVE | `[TOOL_CALLS]` 包装器，带 ID 字段 |
| Functionary v3.1 | TAG_WITH_JSON | `<function=X>` 格式 |
| Functionary v3.2 | Specialized | `>>>` 接收方分隔符（专用处理程序） |
| Fireworks Firefunction v2 | TAG_WITH_JSON | Fireworks 工具格式 |
| DeepSeek R1 Distill (Llama/Qwen) | Reasoning | 强制开放式思考 |
| llama-cpp-deepseek-r1 | Reasoning | 强制开放式思考 |
| Kimi-K2 / Kimi-K2-Instruct | JSON_NATIVE | 带特殊标记的 JSON 工具 |
| Llama 3.1/3.2/3.3 | JSON_NATIVE | 标准 Llama 工具格式 |
| OpenAI GPT-OSS | Specialized | 基于通道（专用处理程序） |
| Apriel 1.5 | JSON_NATIVE | 带 JSON 数组的 `<tool_calls>` 包装器 |
| Apriel 1.6 Thinker | Reasoning | 隐式推理开始 |
| Mistral Small 3.2 | JSON_NATIVE | `[TOOL_CALLS]func[ARGS]{...}` 带调用 ID |
| Devstral | JSON_NATIVE | `[TOOL_CALLS]func[ARGS]{...}` 不带调用 ID |
| StepFun 3.5 Flash | TAG_WITH_TAGGED | `<function=X><parameter=Y>` 格式 |

## 添加新模板支持

要支持新的模板格式：

1. **如果遵循标准模式** — Auto-parser 应自动检测。运行 `llama-debug-template-parser` 验证标记是否正确提取。
2. **如果差分分析提取了错误的标记** — 向 `common/chat-diff-analyzer.cpp` 中的 `workarounds` 向量添加一个 workaround lambda。检查模板源码中的唯一标识子串。
3. **如果需要根本不同的处理方式** — 在 `chat.cpp` 的 auto-parser 块之前添加专用处理函数（GPT-OSS、Functionary v3.2 和 Ministral 就是这样处理的）。

## 边缘情况与注意事项

1. **生成提示与推理预填充**：生成提示由 `common_chat_templates_apply_jinja` 中对比 `add_generation_prompt=false` 与 `true` 提取，因此它精确包含模板追加的内容 —— 避免将先前对话轮次误判。
2. **Per-Call 与 Per-Section 标记**：有些模板单独包装每个工具调用（`per_call_start/end`）；有些包装整个节（`section_start/end`）。T2（`check_per_call_markers()`）通过检查两次调用输出中的第二次调用是否以 section 标记开头来消除歧义。
3. **标签边界修复**：`calculate_diff_split()` 迭代调整前缀/后缀边界，避免拆分 `<tag>` 或 `[marker]` token，确保干净提取。
4. **调用 ID 副作用**：当检测到调用 ID 时，`per_call_end` 可能被错误设置为包含调用 ID 后缀。T7 会在这种情况下清空 `per_call_end`。
5. **工具分析门控**：仅当 `jinja_caps.supports_tool_calls` 为 true 时，`analyze_tools` 才会构造（并运行所有工具分析阶段）。在工具分析内部，仅当 `jinja_caps.supports_parallel_tool_calls` 为 true 时才会运行 `check_per_call_markers()`（T2）。
6. **`analyze_arguments()` 门控**：在工具分析内部，A1 和 A2（参数名/值标记提取）仅对 `TAG_WITH_TAGGED` 格式运行。`extract_argument_separator()` 和 `extract_args_markers()` 对所有非 `JSON_NATIVE` 格式运行。
7. **未检测到的工具格式**：如果 `analyze_tools` 判定支持工具调用但无法确定格式，`build_parser()` 会记录错误并返回 `eps()`（优雅降级），而不会中止。
