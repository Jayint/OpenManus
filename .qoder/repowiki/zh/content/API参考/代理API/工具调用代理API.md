# 工具调用代理API

<cite>
**本文档中引用的文件**  
- [toolcall.py](file://app/agent/toolcall.py)
- [react.py](file://app/agent/react.py)
- [tool_collection.py](file://app/tool/tool_collection.py)
- [terminate.py](file://app/tool/terminate.py)
- [schema.py](file://app/schema.py)
</cite>

## 目录
1. [简介](#简介)
2. [核心属性详解](#核心属性详解)
3. [思考流程解析](#思考流程解析)
4. [执行流程解析](#执行流程解析)
5. [工具执行机制](#工具执行机制)
6. [特殊工具处理](#特殊工具处理)
7. [代码示例](#代码示例)
8. [性能优化建议](#性能优化建议)

## 简介
工具调用代理（ToolCallAgent）是OpenManus框架中的核心组件，继承自ReActAgent，专门用于处理工具/函数调用。该代理通过与大型语言模型（LLM）的交互，能够智能地选择和执行各种工具，实现复杂任务的自动化处理。本API文档详细介绍了ToolCallAgent的核心属性、方法流程和工作机制，为开发者提供全面的使用指导。

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L17-L249)

## 核心属性详解
ToolCallAgent包含多个关键属性，用于配置其行为和功能。

### 可用工具集合（available_tools）
`available_tools`属性定义了代理可以使用的工具集合，类型为`ToolCollection`。该集合在代理初始化时创建，包含所有可用的工具实例。例如，默认配置包含`CreateChatCompletion`和`Terminate`工具。

```python
available_tools: ToolCollection = ToolCollection(
    CreateChatCompletion(), Terminate()
)
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L26-L28)

### 工具选择策略（tool_choices）
`tool_choices`属性控制代理的工具使用行为，类型为`TOOL_CHOICE_TYPE`，支持三种模式：
- `NONE`：禁止使用工具
- `AUTO`：自动选择是否使用工具
- `REQUIRED`：必须使用工具

```python
tool_choices: TOOL_CHOICE_TYPE = ToolChoice.AUTO
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L29-L29)

### 特殊工具列表（special_tool_names）
`special_tool_names`属性定义了需要特殊处理的工具名称列表。这些工具的执行可能会触发代理状态的改变，如终止执行。默认情况下，`Terminate`工具被列为特殊工具。

```python
special_tool_names: List[str] = Field(default_factory=lambda: [Terminate().name])
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L30-L30)

### 当前工具调用列表（tool_calls）
`tool_calls`属性存储当前待执行的工具调用列表，类型为`List[ToolCall]`。该列表在`think`方法中由LLM的响应填充，并在`act`方法中被逐一执行。

```python
tool_calls: List[ToolCall] = Field(default_factory=list)
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L32-L32)

### 观察结果截断长度（max_observe）
`max_observe`属性控制工具执行结果的截断长度。当设置为整数时，结果将被截断到指定长度；设置为`None`或`False`时，不进行截断。

```python
max_observe: Optional[Union[int, bool]] = None
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L36-L36)

## 思考流程解析
`think`方法是ToolCallAgent的核心，负责处理当前状态并决定下一步行动。

### 构造带工具选项的LLM请求
`think`方法首先构造一个包含工具选项的LLM请求。它使用`available_tools.to_params()`获取工具参数，并根据`tool_choices`策略发送请求。

```python
response = await self.llm.ask_tool(
    messages=self.messages,
    system_msgs=([Message.system_message(self.system_prompt)] if self.system_prompt else None),
    tools=self.available_tools.to_params(),
    tool_choice=self.tool_choices,
)
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L38-L128)

### 异常处理
方法中包含了对`TokenLimitExceeded`等异常的处理。当遇到token限制错误时，代理会将状态设置为`FINISHED`，并返回`False`。

```python
if hasattr(e, "__cause__") and isinstance(e.__cause__, TokenLimitExceeded):
    token_limit_error = e.__cause__
    self.memory.add_message(
        Message.assistant_message(
            f"Maximum token limit reached, cannot continue execution: {str(token_limit_error)}"
        )
    )
    self.state = AgentState.FINISHED
    return False
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L38-L128)

### 解析LLM返回的工具调用指令
LLM的响应被解析为`tool_calls`列表，并根据`tool_choice`模式决定执行路径：
- `NONE`模式：如果存在工具调用，则发出警告
- `REQUIRED`模式：即使没有工具调用也返回`True`，由`act`方法处理
- `AUTO`模式：如果没有工具调用但有内容，则继续执行

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L38-L128)

## 执行流程解析
`act`方法负责执行`tool_calls`中的每个工具调用。

### 遍历并执行工具调用
方法遍历`tool_calls`列表，对每个工具调用执行`execute_tool`方法，并将结果写入记忆。

```python
for command in self.tool_calls:
    self._current_base64_image = None
    result = await self.execute_tool(command)
    if self.max_observe:
        result = result[: self.max_observe]
    tool_msg = Message.tool_message(
        content=result,
        tool_call_id=command.id,
        name=command.function.name,
        base64_image=self._current_base64_image,
    )
    self.memory.add_message(tool_msg)
    results.append(result)
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L130-L163)

## 工具执行机制
`execute_tool`方法实现了工具执行的健壮性设计。

### 参数解析
方法首先尝试解析工具调用的参数。如果JSON解析失败，会捕获`JSONDecodeError`并返回错误信息。

```python
try:
    args = json.loads(command.function.arguments or "{}")
except json.JSONDecodeError:
    error_msg = f"Error parsing arguments for {name}: Invalid JSON format"
    return f"Error: {error_msg}"
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L165-L207)

### 工具执行与结果格式化
解析参数后，方法通过`available_tools.execute`执行工具，并格式化结果。如果结果包含base64图像，会将其存储以供后续使用。

```python
result = await self.available_tools.execute(name=name, tool_input=args)
if hasattr(result, "base64_image") and result.base64_image:
    self._current_base64_image = result.base64_image
observation = f"Observed output of cmd `{name}` executed:\n{str(result)}" if result else f"Cmd `{name}` completed with no output"
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L165-L207)

## 特殊工具处理
ToolCallAgent通过特殊机制处理特定工具，如`Terminate`。

### 特殊工具识别
`_is_special_tool`方法检查工具名称是否在`special_tool_names`列表中。

```python
def _is_special_tool(self, name: str) -> bool:
    return name.lower() in [n.lower() for n in self.special_tool_names]
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L224-L226)

### 执行终止机制
`_handle_special_tool`方法在检测到特殊工具时调用`_should_finish_execution`，如果返回`True`，则将代理状态设置为`FINISHED`。

```python
async def _handle_special_tool(self, name: str, result: Any, **kwargs):
    if not self._is_special_tool(name):
        return
    if self._should_finish_execution(name=name, result=result, **kwargs):
        self.state = AgentState.FINISHED
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L209-L217)

## 代码示例
以下示例展示了如何配置和使用ToolCallAgent。

### 配置代理
```python
agent = ToolCallAgent(
    available_tools=ToolCollection(
        PythonExecute(),
        BrowserUseTool(),
        Terminate()
    ),
    tool_choices=ToolChoice.AUTO,
    max_observe=10000
)
```

### 添加自定义工具
```python
class CustomTool(BaseTool):
    name: str = "custom_tool"
    description: str = "A custom tool for specific tasks"
    parameters: dict = {"type": "object", "properties": {"param": {"type": "string"}}}

    async def execute(self, param: str) -> str:
        return f"Custom tool executed with {param}"

agent.available_tools.add_tool(CustomTool())
```

### 工具调用-执行-结果反馈循环
```python
# 启动代理
result = await agent.run("Analyze the data and create a report")

# 代理内部流程：
# 1. think() - LLM决定使用PythonExecute工具
# 2. act() - 执行PythonExecute工具
# 3. execute_tool() - 解析参数并执行
# 4. 结果写入记忆，循环继续
# 5. 最终调用Terminate工具，代理状态变为FINISHED
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L17-L249)
- [tool_collection.py](file://app/tool/tool_collection.py#L8-L70)
- [terminate.py](file://app/tool/terminate.py#L8-L25)

## 性能优化建议
为避免资源耗尽，建议合理设置以下参数：

### 设置最大步数（max_steps）
合理设置`max_steps`可以防止代理陷入无限循环。建议根据任务复杂度设置合理的上限。

```python
max_steps: int = 30  # 默认值，可根据需要调整
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L35-L35)

### 设置观察结果截断长度（max_observe）
对于可能产生大量输出的工具，设置`max_observe`可以控制内存使用。

```python
max_observe: Optional[Union[int, bool]] = 15000  # 限制输出为15000字符
```

**Section sources**
- [toolcall.py](file://app/agent/toolcall.py#L36-L36)

### 监控token使用
通过LLM的token计数功能监控输入和输出token，避免超出模型限制。

```python
# 在LLM类中实现的token计数
def count_message_tokens(self, messages: List[dict]) -> int:
    # 计算消息列表的总token数
    pass
```

**Section sources**
- [llm.py](file://app/llm.py#L18-L187)