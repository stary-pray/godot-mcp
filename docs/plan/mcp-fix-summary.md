# MCP (Mission Control Protocol) 修复总结

## 问题概述

在使用 WebSocket 接口控制游戏时，发现了以下几个主要问题：

1. **参数命名不一致**：`get_available_actions` 返回 `valid_card_entity_ids`，但 `select_card` 命令需要 `card_id` 参数。
2. **参数位置混乱**：有些命令接受直接放在命令对象中的参数，有些则需要放在 `parameters` 对象中。
3. **实体 ID 查找失败**：即使提供了正确的数字实体 ID，`_find_card_entity` 方法也无法找到对应的卡牌实体。
4. **错误信息不明确**：当找不到实体时，错误消息缺乏上下文信息，难以调试。

## 修复内容

### 1. 增强 `_find_card_entity` 方法的调试和错误处理

- 添加了详细的日志记录，跟踪实体查找的每个步骤
- 记录传入的 ID 类型、IDManager 查找结果、实体有效性检查和分组检查
- 当查找失败时，收集并显示当前可用的手牌信息，提供更有用的错误上下文

### 2. 统一参数处理方式

- 修改了 `_process_command` 中的所有命令处理逻辑，**仅**从 `parameters` 对象中获取参数
- 移除了对命令对象顶层参数的直接访问（如 `command.card_id`）
- 为了向后兼容，保留了对旧参数名的支持，但会输出警告信息

### 3. 统一参数命名

- 将所有实体 ID 参数统一为 `*_entity_id` 格式：
  - `card_id` → `card_entity_id`
  - `target_id` → `target_entity_id`
  - `reward_id` → `reward_entity_id`
- 更新了 `PlayerInputDispatcher._get_available_actions` 方法中的参数名
  - `valid_reward_ids` → `valid_reward_entity_ids`

### 4. 改进错误消息

- 当找不到实体时，错误消息现在包含：
  - 尝试查找的 ID
  - 当前可用的实体 ID 列表
  - 实体查找过程的详细日志

### 5. 更新文档

- 更新了 `docs/current/README_Websocket_Interface.md`，反映了新的参数命名和位置要求
- 明确说明了所有参数必须放在 `parameters` 对象中
- 更新了示例代码，使用新的参数名

## 测试

创建了 `ws_client/test_mcp_fixes.js` 测试脚本，用于验证修复后的功能：

1. 连接到游戏
2. 获取可用操作
3. 如果有可用卡牌，尝试使用新的参数格式选择卡牌
4. 如果没有可用卡牌，尝试执行弃牌并重抽操作

## 使用说明

### 正确的参数格式

```javascript
// 正确的格式
{
  "action": "select_card",
  "parameters": {
    "card_entity_id": 9  // 使用数字实体ID
  }
}

// 错误的格式（不要这样做）
{
  "action": "select_card",
  "card_id": 9  // 不要直接放在命令对象中
}
```

### 获取有效的实体 ID

始终使用 `get_available_actions` 命令获取当前可用的操作和有效的实体 ID：

```javascript
// 发送请求
{
  "action": "get_available_actions"
}

// 响应示例
{
  "available_actions": [
    {
      "name": "select_card",
      "valid_card_entity_ids": [9, 10, 11]  // 使用这些ID
    }
  ]
}
```

## 后续工作

1. 考虑实现 fallback 逻辑，支持通过卡牌数据 ID（如 "strike"）查找卡牌
2. 进一步改进错误处理和日志记录
3. 添加更多的单元测试和集成测试

## MCP 端修复内容 (2023-11-15)

根据 `docs/plan/mcp-fix_1.md` 中的要求，我们对 MCP 项目进行了以下修复：

### 1. 更新了 `mcp_godot_send_runtime_command` 工具的描述和参数说明

- 在工具描述中明确说明所有命令参数必须嵌套在 `parameters` 对象中
- 在 `parameters` 参数的描述中明确说明所有实体ID参数均使用 `*_entity_id` 格式：
  - 卡牌选择：使用 `card_entity_id` （而非 `card_id`）
  - 目标选择：使用 `target_entity_id` （而非 `target_id`）
  - 奖励选择：使用 `reward_entity_id` （而非 `reward_id`）
- 添加了具体的示例：`{"card_entity_id": 9}` 用于选择实体ID为9的卡牌

### 2. 增强了 WebSocket 错误日志

- 修改了 `WebSocketClient.sendCommand` 方法中的错误处理逻辑
- 在所有错误消息中添加了尝试发送的命令详情（`messageBody`）
- 将消息体的创建移到函数开始处，以便在所有错误处理中使用
- 增强了以下错误场景的日志：
  - WebSocket 连接超时
  - JSON 解析失败
  - WebSocket 错误
  - WebSocket 连接意外关闭

### 3. 改进了错误提示信息

- 在 `handleSendRuntimeCommand` 函数的错误处理中添加了更详细的提示信息
- 对于 Godot 运行时返回的错误，添加了关于实体ID参数命名和参数嵌套的具体提示：
  - 卡牌选择：使用 `card_entity_id` （而非 `card_id`）
  - 目标选择：使用 `target_entity_id` （而非 `target_id`）
  - 奖励选择：使用 `reward_entity_id` （而非 `reward_id`）
- 在一般性错误处理中也添加了关于实体ID参数命名和参数嵌套的提示
