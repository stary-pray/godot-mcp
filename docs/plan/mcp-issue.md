# MCP (Mission Control Protocol) 使用记录与问题

本文档记录了使用 MCP 控制 yuki-godot 游戏的尝试过程，包括成功和失败的情况，以及可能的改进建议。

## 环境信息

- 游戏：yuki-godot
- Godot 版本：4.3.stable.official.77dcf97d8
- WebSocket 端口：9080

## 测试过程

### 1. 启动游戏

首先使用 Godot 运行项目，游戏成功启动并初始化了 WebSocket 服务器：

```
[NetworkManager] 初始化完成
[NetworkManager] 调试模式已启用
[NetworkManager] 默认端口: 9080
[NetworkManager] WebSocket服务器已启动
[NetworkManager] 监听地址: *:9080
[NetworkManager] WebSocket服务器自动启动成功
```

### 2. 获取游戏状态

使用 `mcp_godot_get_game_state_godot` 命令获取游戏状态：

```json
{
  "status": "success",
  "message": "游戏状态获取成功",
  "data": {
    "message": "游戏状态获取成功",
    "state": {
      "current_day": 0,
      "discard_pile_count": 0,
      "draw_pile_count": 7,
      "enemies": [
        {
          "block": 0,
          "buffs_debuffs": [],
          "entity_id": 2,
          "hp": 189,
          "intent": {
            "cooldown_remaining": 0,
            "target_id": -1,
            "type": "ATTACK",
            "value": 6
          },
          "max_hp": 200,
          "name": "史莱姆"
        }
      ],
      "exhaust_pile_count": 0,
      "hand": [
        {
          "card_data_id": "strike",
          "cost": 1,
          "description": "造成8点伤害。",
          "entity_id": 9,
          "is_playable": true,
          "name": "打击",
          "runtime_values": {
            "block_amount": 0,
            "dmg_amount": 8
          },
          "type": "ATTACK"
        },
        {
          "card_data_id": "strike",
          "cost": 1,
          "description": "造成8点伤害。",
          "entity_id": 10,
          "is_playable": true,
          "name": "打击",
          "runtime_values": {
            "block_amount": 0,
            "dmg_amount": 8
          },
          "type": "ATTACK"
        },
        {
          "card_data_id": "strike",
          "cost": 1,
          "description": "造成7点伤害。",
          "entity_id": 11,
          "is_playable": true,
          "name": "打击",
          "runtime_values": {
            "block_amount": 0,
            "dmg_amount": 7
          },
          "type": "ATTACK"
        }
      ],
      "is_paused": false,
      "is_targeting": false,
      "player": {
        "block": 0,
        "buffs_debuffs": [],
        "energy": 3,
        "entity_id": 1,
        "hp": 74,
        "max_energy": 3,
        "max_hp": 80
      },
      "selected_card_entity_id": -1,
      "targeting_source_card_entity_id": null,
      "valid_target_entity_ids": []
    },
    "status": "success"
  }
}
```

### 3. 获取可用操作

使用 `get_available_actions_godot` 命令获取当前可用的操作：

```json
{
  "available_actions": [
    {
      "name": "toggle_pause"
    },
    {
      "name": "get_game_state"
    },
    {
      "name": "discard_and_draw"
    },
    {
      "name": "select_card",
      "valid_card_entity_ids": [
        9,
        10,
        11
      ]
    }
  ],
  "message": "Available actions retrieved.",
  "status": "success"
}
```

### 4. 尝试选择卡牌（失败案例）

#### 尝试 1：使用 `card_entity_id` 参数

```javascript
{
  "action": "select_card",
  "parameters": {"card_entity_id": 9}
}
```

错误响应：
```
Godot 运行时错误 (指令: 'select_card'): 缺少必要参数 'card_id'
```

#### 尝试 2：直接使用 `card_id` 参数（而非放在 parameters 中）

```javascript
{
  "action": "select_card",
  "card_id": 9
}
```

错误响应：
```
Godot 运行时错误 (指令: 'select_card'): 找不到指定的卡牌实体: 9
```

#### 尝试 3：使用 `card_id` 参数（放在 parameters 中）

```javascript
{
  "action": "select_card",
  "parameters": {"card_id": 9}
}
```

错误响应：
```
Godot 运行时错误 (指令: 'select_card'): 找不到指定的卡牌实体: 9
```

#### 尝试 4：使用卡牌类型作为 ID

```javascript
{
  "action": "select_card",
  "parameters": {"card_id": "strike"}
}
```

错误响应：
```
Godot 运行时错误 (指令: 'select_card'): 找不到指定的卡牌实体: strike
```

### 5. 成功案例：使用 discard_and_draw 命令

```javascript
{
  "action": "discard_and_draw"
}
```

成功响应：
```json
{
  "status": "success",
  "message": "运行时命令 'discard_and_draw' 已成功执行",
  "command_response": {
    "command_response": {
      "status": "success"
    },
    "message": "运行时命令 'discard_and_draw' 已成功执行",
    "status": "success"
  },
  "game_state": {
    "message": "游戏状态获取成功",
    "state": {
      "current_day": 0,
      "discard_pile_count": 3,
      "draw_pile_count": 2,
      "enemies": [
        {
          "block": 5,
          "buffs_debuffs": [],
          "entity_id": 2,
          "hp": 109,
          "intent": {
            "cooldown_remaining": 0,
            "target_id": -1,
            "type": "DEFEND",
            "value": 5
          },
          "max_hp": 200,
          "name": "史莱姆"
        }
      ],
      "exhaust_pile_count": 0,
      "hand": [
        {
          "card_data_id": "defend",
          "cost": 1,
          "description": "获得5点格挡。",
          "entity_id": 25,
          "is_playable": true,
          "name": "防御",
          "runtime_values": {
            "block_amount": 5,
            "dmg_amount": 0
          },
          "type": "SKILL"
        },
        // ... 其他卡牌
      ],
      // ... 其他游戏状态
    },
    "status": "success"
  }
}
```

## 问题分析

### 1. 参数命名不一致

在 WebSocket 接口文档中，`select_card` 命令需要 `card_id` 参数，但实际上可能需要的是 `entity_id`。

### 2. 参数位置不明确

文档中没有明确说明参数是应该直接放在命令对象中，还是应该放在 `parameters` 对象中。

### 3. 实体 ID 类型混淆

文档中提到：
> `NetworkManager` 会首先尝试将传入的 ID 解析为整数，并使用 `IDManager` 查找。
> 如果解析失败或未找到，它会遍历相应的游戏对象分组（手牌中的卡牌、场景中的角色、奖励选项），并尝试匹配其数据组件中的 `id` 字符串。

但实际上，即使使用正确的数字 ID（如 9），系统仍然无法找到对应的卡牌实体。

## 改进建议

1. **统一参数命名**：
   - 在 `get_available_actions` 返回的 `select_card` 动作中，参数名为 `valid_card_entity_ids`
   - 但在 `select_card` 命令中，参数名却是 `card_id`
   - 建议统一为 `card_entity_id` 或 `card_id`

2. **明确参数位置**：
   - 明确说明参数应该放在 `parameters` 对象中，还是直接放在命令对象中
   - 最好支持两种方式，并在文档中明确说明

3. **改进错误消息**：
   - 当找不到卡牌实体时，错误消息应该更具体，例如提供当前可用的卡牌实体 ID 列表
   - 可以在错误消息中包含更多上下文信息，帮助调试

4. **实体 ID 处理**：
   - 确保 `_find_card_entity` 方法能够正确处理数字 ID
   - 考虑在 `get_game_state` 响应中添加一个字段，明确说明每个卡牌的 `entity_id` 是什么，以及如何在 `select_card` 命令中使用

5. **添加调试模式**：
   - 添加一个调试模式，当找不到卡牌实体时，打印出所有可用的卡牌实体及其 ID
   - 这将有助于理解为什么特定的 ID 无法找到对应的实体

## 结论

虽然 `discard_and_draw` 命令能够成功执行，但 `select_card` 命令存在参数传递问题。通过统一参数命名、明确参数位置、改进错误消息和实体 ID 处理，可以提高 MCP 接口的易用性和可靠性。
