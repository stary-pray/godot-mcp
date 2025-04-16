# MCP 与 Godot 端修复对齐说明

本文档说明了 MCP 端的修复如何与 Godot 端的修复对齐，确保两端的参数命名和处理方式保持一致。

## Godot 端主要修改

根据 `docs/plan/mcp-fix-summary-godot-side.md`，Godot 端进行了以下主要修改：

1. **统一参数处理方式**：修改了 `_process_command` 中的所有命令处理逻辑，**仅**从 `parameters` 对象中获取参数
2. **统一参数命名**：将所有实体 ID 参数统一为 `*_entity_id` 格式
   - `card_id` → `card_entity_id`
   - `target_id` → `target_entity_id`
   - `reward_id` → `reward_entity_id`
3. **改进错误消息**：当找不到实体时，提供更详细的错误信息

## MCP 端对应修改

为了与 Godot 端的修改保持一致，我们在 MCP 端进行了以下修改：

### 1. 更新工具描述和参数说明

- 在 `mcp_godot_send_runtime_command` 工具的描述中明确说明所有命令参数必须嵌套在 `parameters` 对象中
- 在参数描述中明确说明所有实体 ID 参数均使用 `*_entity_id` 格式：
  - 卡牌选择：使用 `card_entity_id`（而非 `card_id`）
  - 目标选择：使用 `target_entity_id`（而非 `target_id`）
  - 奖励选择：使用 `reward_entity_id`（而非 `reward_id`）

### 2. 改进错误提示信息

- 在 `handleSendRuntimeCommand` 函数的错误处理中添加了更详细的提示信息
- 对于 Godot 运行时返回的错误，添加了关于实体 ID 参数命名和参数嵌套的具体提示
- 在一般性错误处理中也添加了关于实体 ID 参数命名和参数嵌套的提示

### 3. 增强 WebSocket 错误日志

- 修改了 `WebSocketClient.sendCommand` 方法中的错误处理逻辑
- 在所有错误消息中添加了尝试发送的命令详情，便于调试

## 参数命名对照表

| 旧参数名 | 新参数名 | 用途 |
|---------|---------|------|
| `card_id` | `card_entity_id` | 选择卡牌 |
| `target_id` | `target_entity_id` | 选择目标 |
| `reward_id` | `reward_entity_id` | 选择奖励 |

## 正确的参数格式示例

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

## 测试建议

建议通过以下步骤测试修复效果：

1. 启动 yuki-godot 游戏
2. 使用 `mcp_godot_get_game_state_godot` 获取当前游戏状态
3. 使用 `get_available_actions_godot` 获取可用操作
4. 尝试使用 `mcp_godot_send_runtime_command_godot` 发送 `select_card` 命令，使用 `card_entity_id` 参数
5. 验证命令是否成功执行
