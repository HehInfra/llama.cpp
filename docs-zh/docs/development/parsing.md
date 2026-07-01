# 解析模型输出

`common` 库包含一个适合解析模型输出的 PEG 解析器实现。

带有 `common_peg_*` 前缀的类型旨在通用，除了解析模型输出外，还可用于解析用户提供的正则表达式模式等场景。

带有 `common_chat_peg_*` 前缀的类型是专门用于模型输出的辅助工具。

该解析器的特性包括：

- 流式输入的部分解析
- 内置 JSON 解析器
- 通过“标记”节点生成带有语义的抽象语法树（AST）

## 示例

下面是一个刻意简化的示例，演示如何使用 PEG 解析器来解析一个以 JSON 形式输出参数的模型。

```cpp
auto parser = build_chat_peg_parser([&](common_chat_peg_builder & p) {
    // Build a choice of all available tools
    auto tool_choice = p.choice();
    for (const auto & tool : tools) {
        const auto & function = tool.at("function");
        std::string name = function.at("name");
        const auto & schema = function.at("parameters");

        auto tool_name = p.json_member("name", "\"" + p.literal(name) + "\"");
        auto tool_args = p.json_member("arguments", p.schema(p.json(), "tool-" + name + "-schema", schema));

        tool_choice |= p.rule("tool-" + name, "{" << tool_name << "," << tool_args << "}");
    }

    // Define the tool call structure: <tool_call>[{tool}]</tool_call>
    auto tool_call = p.trigger_rule("tool-call",
        p.sequence({
            p.literal("<tool_call>["),
            tool_choice,
            p.literal("]</tool_call>")
        })
    );

    // Parser accepts content, optionally followed by a tool call
    return p.sequence({
        p.content(p.until("<tool_call>")),
        p.optional(tool_call),
        p.end()
    });
});
```

更完整的示例请参见 [tests/test-chat-peg-parser.cpp](/tests/test-chat-peg-parser.cpp) 中的 `test_example_native()`。

## 解析器/组合子

### 基础匹配器

- **`eps()`** - 匹配空内容，总是成功（epsilon/空匹配）
- **`start()`** - 匹配输入起始位置（锚点 `^`）
- **`end()`** - 匹配输入结束位置（锚点 `$`）
- **`literal(string)`** - 匹配精确的字面量字符串
- **`any()`** - 匹配任意单个字符（`.`）

### 组合子

- **`sequence(...)`** - 按顺序匹配多个解析器；所有解析器都必须成功
- **`choice(...)`** - 从多个候选解析器中匹配第一个成功的（有序选择）
- **`one_or_more(p)`** - 匹配一次或多次重复（`+`）
- **`zero_or_more(p)`** - 匹配零次或多次重复（`*`）
- **`optional(p)`** - 匹配零次或一次（`?`）
- **`repeat(p, min, max)`** - 匹配 min 到 max 次重复（使用 `-1` 表示无上限）
- **`repeat(p, n)`** - 精确匹配 n 次重复

### 前瞻

- **`peek(p)`** - 正向前瞻：解析器成功时成功，但不消耗输入（`&`）
- **`negate(p)`** - 负向前瞻：解析器失败时成功，但不消耗输入（`!`）

### 字符类与工具

- **`chars(classes, min, max)`** - 匹配字符类中字符的重复
- **`space()`** - 匹配零个或多个空白字符（空格、制表符、换行符）
- **`until(delimiter)`** - 匹配字符直到遇到分隔符（分隔符不被消耗）
- **`until_one_of(delimiters)`** - 匹配字符直到遇到列表中任意一个分隔符
- **`rest()`** - 匹配剩余所有内容（`.*`）

### JSON 解析器

- **`json()`** - 完整 JSON 解析器（对象、数组、字符串、数字、布尔值、null）
- **`json_object()`** - JSON 对象解析器
- **`json_array()`** - JSON 数组解析器
- **`json_string()`** - JSON 字符串解析器
- **`json_number()`** - JSON 数字解析器
- **`json_bool()`** - JSON 布尔值解析器
- **`json_null()`** - JSON null 解析器
- **`json_string_content()`** - 不含外层引号的 JSON 字符串内容解析器
- **`json_member(key, p)`** - 具有指定键和值解析器的 JSON 对象成员

### 语法构建

- **`ref(name)`** - 创建一个命名规则的轻量级引用（用于递归语法）
- **`rule(name, p, trigger)`** - 创建一个命名规则并返回其引用
- **`trigger_rule(name, p)`** - 创建一个触发规则（惰性语法生成的入口点）
- **`schema(p, name, schema, raw)`** - 用 JSON Schema 元数据包装解析器以生成语法

### AST 控制

- **`atomic(p)`** - 阻止为部分解析创建 AST 节点
- **`tag(tag, p)`** - 创建带有语义标签的 AST 节点（多个节点可共享标签）

## GBNF 语法生成

PEG 解析器还可以作为生成 GBNF 语法的便捷 DSL，但有一些例外。

```cpp
data.grammar = build_grammar([&](const common_grammar_builder & builder) {
    foreach_function(params.tools, [&](const json & fn) {
        builder.resolve_refs(fn.at("parameters"));
    });
    parser.build_grammar(builder, data.grammar_lazy);
});
```

一个显著的例外是 `negate(p)` 前瞻解析器，它无法被定义为 CFG 语法，因此不会生成规则。它的使用应受到限制，最好隐藏在 `schema()` 解析器之后。在很多情况下，`until(delimiter)` 或 `until_one_of(delimiters)` 是更好的选择。

另一个限制是 PEG 解析器要求语法无歧义。相比之下，`llama-grammar` 实现可以支持有歧义的语法，尽管这类语法难以解析。

### 惰性语法

在惰性语法生成期间，语法中只会生成从 `trigger_rule(p)` 可达的规则。所有触发规则都会作为根规则的候选项加入。仍然需要定义触发模式，因为解析器与语法采样之间没有交互。

### JSON Schema

`schema(p, name, schema, raw)` 解析器将使用 `json-schema-to-grammar` 实现来生成语法，而不是使用底层解析器。

`raw` 选项会生成适合原始字符串的语法，而不是 JSON 字符串。换句话说，它不会用引号包裹，也不需要对引号进行转义。只有在 `type == "string"` 时才应使用。

缺点是这可能导致有歧义的语法。例如，如果用户提供了模式 `^.*$`，可能会生成如下语法：

```
root ::= "<arg>" .* "</arg>"
```

这会生成一个 PEG 解析器无法解析的有歧义语法。为了缓解这个问题，如果在模式中发现了 `.*`，则会改为使用底层解析器生成语法。

## 对话解析中常见的 AST 结构

大多数模型输出可以分为以下几类：

- 仅内容
- 工具调用，参数以单个 JSON 对象形式输出
- 工具调用，参数以独立实体形式输出，可能是 XML（Qwen3-Coder、MiniMax M2）或伪函数调用（LFM2）

为提供广泛的覆盖能力，[`common/chat-peg-parser.h`](/common/chat-peg-parser.h) 包含了一系列构建器和映射器，可帮助为上述类型创建解析器以及访问器/提取器。它们要求解析器为节点打上标签，以符合某种 AST “结构”。这种规范化使得信息提取和解析泛化变得容易。

### 简单模式

`common_chat_peg_builder` 构建了一个 `simple` 解析器，用于支持仅输出内容的模型，并可选择性地包含推理内容。

- **`reasoning(p)`** - 用于提取 `reasoning_content` 的标记节点
- **`content(p)`** - 用于提取 `content` 的标记节点

```cpp
build_chat_peg_parser([&](common_chat_peg_parser & p) {
    return p.sequence({
        p.optional("<think>" + p.reasoning(p.until("</think>")) + "</think>"),
        p.content(p.until("<tool_call>")),
        p.end()
    });
});
```

使用 `common_chat_peg_mapper` 来提取内容。注意，当 `chat_format == COMMON_CHAT_FORMAT_PEG_SIMPLE` 时，`common_chat_peg_parser` 已经自动完成了这一步。

```cpp
auto result = parser.parse(ctx);

common_chat_msg msg;
auto mapper = common_chat_peg_mapper(msg);
mapper.from_ast(ctx.ast, result);
```

### 原生模式

`common_chat_peg_builder` 构建了一个 `native` 解析器，适用于以直接 JSON 对象形式输出工具参数的模型。

- **`reasoning(p)`** - 用于 `reasoning_content` 的标记节点
- **`content(p)`** - 用于 `content` 的标记节点
- **`tool(p)`** - 标记单个工具调用的整体
- **`tool_open(p)`** - 标记工具调用的开始
- **`tool_close(p)`** - 标记工具调用的结束
- **`tool_id(p)`** - 标记工具调用 ID（可选）
- **`tool_name(p)`** - 标记工具名称
- **`tool_args(p)`** - 标记工具参数

```cpp
build_chat_peg_parser([&](common_chat_peg_builder & p) {
    auto get_weather_tool = p.tool(p.sequence({
        p.tool_open(p.literal("{")),
        p.json_member("name", "\"" + p.tool_name(p.literal("get_weather")) + "\""),
        p.literal(","),
        p.json_member("arguments", p.tool_args(p.json())),
        p.tool_close(p.literal("}"))
    }));

    return p.sequence({
        p.content(p.until("<tool_call>")),
        p.literal("<tool_call>"),
        get_weather_tool,
        p.literal("</tool_call>"),
        p.end()
    });
});
```

### 构造模式

`common_chat_peg_builder` 构建了一个 `constructed` 解析器，适用于以独立实体形式输出工具参数的模型，例如 XML 标签。

- **`reasoning(p)`** - 用于 `reasoning_content` 的标记节点
- **`content(p)`** - 用于 `content` 的标记节点
- **`tool(p)`** - 标记单个工具调用的整体
- **`tool_open(p)`** - 标记工具调用的开始
- **`tool_close(p)`** - 标记工具调用的结束
- **`tool_name(p)`** - 标记工具名称
- **`tool_arg(p)`** - 标记一个完整的工具参数（名称 + 值）
- **`tool_arg_open(p)`** - 标记工具参数的开始
- **`tool_arg_close(p)`** - 标记工具参数的结束
- **`tool_arg_name(p)`** - 标记参数名称
- **`tool_arg_string_value(p)`** - 标记参数的字符串值
- **`tool_arg_json_value(p)`** - 标记参数的 JSON 值

```cpp
build_chat_peg_parser([&](common_chat_peg_builder & p) {
    auto location_arg = p.tool_arg(
        p.tool_arg_open("<parameter name=\"" + p.tool_arg_name(p.literal("location")) + "\">"),
        p.tool_arg_string_value(p.until("</parameter>")),
        p.tool_arg_close(p.literal("</parameter>"))
    );

    auto get_weather_tool = p.tool(p.sequence({
        p.tool_open("<function name=\"" + p.tool_name(p.literal("get_weather")) + "\">"),
        location_arg,
        p.tool_close(p.literal("</function>"))
    }));

    return p.sequence({
        p.content(p.until("<tool_call>")),
        p.literal("<tool_call>"),
        get_weather_tool,
        p.literal("</tool_call>"),
        p.end()
    });
});
```
