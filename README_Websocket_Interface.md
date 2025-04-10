## Godot WebSocket 控制接口文档

**版本:** 1.0
**最后更新:** 2024-04-12

### 1. 概述

本文档描述了如何通过 WebSocket 连接到正在运行的 Godot 游戏实例，并发送指令来模拟玩家操作。这主要用于外部工具（如 MCP - Master Control Program）对游戏进行运行时控制、测试自动化或集成其他系统。

### 2. 连接信息

*   **协议:** WebSocket (`ws://`)
*   **主机:** 运行 Godot 游戏实例的 IP 地址。服务器默认监听所有可用 IP (`*`)。
*   **端口:** `9080` (默认值，可在 `NetworkManager.gd` 中修改 `DEFAULT_PORT`)

### 3. 通信格式

#### 3.1 客户端 (MCP) -> 服务器 (Godot)

*   **格式:** 所有发送给 Godot 的指令必须是 **UTF-8 编码的 JSON 字符串**。
*   **结构:** JSON 对象必须包含以下字段：
    *   `action` (String, 必需): 表示要执行的操作名称。
    *   `parameters` (Object, 可选): 包含该操作所需的参数键值对。

*   **示例:**
    ```json
    // 无参数指令
    {
      "action": "toggle_pause"
    }

    // 带参数指令
    {
      "action": "select_card",
      "parameters": {
        "card_id": "strike_card_instance_123"
      }
    }
    ```

#### 3.2 服务器 (Godot) -> 客户端 (MCP)

*   **格式:** Godot 会对收到的每个有效指令发送一个 JSON 响应。
*   **成功响应:** 表明指令已被接收并成功转发给内部处理逻辑（但不保证游戏逻辑一定能成功执行）。
    ```json
    {
      "status": "success"
    }
    ```
*   **失败响应:** 表明指令解析失败、格式错误、缺少必要字段或内部处理过程中出现错误。
    ```json
    {
      "status": "error",
      "message": "具体的错误描述信息..."
    }
    ```
    常见的错误信息包括：`"Invalid JSON format"`, `"Message must be a JSON object"`, `"Missing 'action' field"`, `"Command processing failed"` (可能由于缺少参数、实体未找到等原因)。

### 4. 可用指令 (`action`) 详解

以下是当前支持的 `action` 及其参数：

| Action (字符串)     | 描述                                     | 参数 (`parameters`)                                     | 状态/备注                                               |
| :------------------ | :--------------------------------------- | :------------------------------------------------------ | :------------------------------------------------------ |
| `toggle_pause`      | 切换游戏的暂停/恢复状态。                | 无                                                      | :white_check_mark: 可用                                     |
| `draw_card`         | 请求执行一次抽牌动作。                   | 无                                                      | :white_check_mark: 可用 (受游戏内冷却时间 `8s` 限制)       |
| `select_card`       | 模拟玩家选择一张手牌。                   | `card_id`: String (要选择的卡牌的唯一标识符)           | :white_check_mark: 可用。`card_id` 应为卡牌实例的 `card_data.id`。 |
| `discard_card`      | 请求弃掉指定的手牌。                     | `card_id`: String (要弃置的卡牌的唯一标识符)           | :white_check_mark: 可用 (受游戏内冷却时间 `8s` 限制)。`card_id` 同上。 |
| `select_target`     | 模拟玩家选择一个目标实体（通常是敌人）。 | `target_id`: String (目标实体的唯一标识符)             | :white_check_mark: 可用。`target_id` 可为实体的 `entity_id` (数字) 或 `character_data.id` (字符串)。 |
| `select_reward`     | 在奖励界面选择一个奖励。                 | `reward_id`: String (奖励选项的唯一标识符，可能是卡牌 ID) | :white_check_mark: 可用。`reward_id` 的具体格式待定（可能是卡牌 ID 或奖励槽位索引）。 |
| `get_game_state`    | 获取当前游戏的详细状态。                 | `pause_game`: Boolean (可选，默认 false) - 如果为 true，在收集状态前暂停游戏 | :white_check_mark: 可用。返回包含游戏状态的 JSON 对象。 |

**关于实体 ID (`card_id`, `target_id`, `reward_id`):**

*   `NetworkManager` 会首先尝试将传入的 ID 解析为整数，并使用 `IDManager` 查找。
*   如果解析失败或未找到，它会遍历相应的游戏对象分组（手牌中的卡牌、场景中的角色、奖励选项），并尝试匹配其数据组件中的 `id` 字符串。
*   **建议:** 为了明确性和效率，优先使用**数字实体 ID** (`entity_id`) 进行通信（如果 MCP 端可以获取到）。如果只能获取资源 ID (如 `strike`, `defend`)，请确保传递的是**卡牌实例对应的数据 ID** (`card_instance.card_data.id`) 或**角色数据 ID** (`character_data_component.id`)。

**重要提示:**

*   所有指令的执行仍然受到 Godot 游戏内部逻辑和状态的约束（例如，游戏暂停时、卡牌冷却时、能量不足时、目标无效时，操作可能失败或被忽略）。即使 WebSocket 响应 `{"status": "success"}`，也仅表示指令被正确接收和分发，不代表游戏逻辑层面的操作一定成功。
*   Godot 端通过 `PlayerInputDispatcher` (Autoload) 将接收到的网络指令与本地输入（键盘、鼠标）统一处理，再调用相应的游戏系统（如 `CardSystem`, `CardActionSystem`）执行逻辑。

### 5. 错误处理

客户端 (MCP) 应准备处理 Godot 返回的错误响应，并根据 `message` 字段判断失败原因。

### 6. 未来扩展

未来可能会添加更多指令，例如：

*   查询游戏状态（当前能量、手牌列表、敌人状态等）。
*   更复杂的卡牌交互指令。
*   控制游戏设置或流程的指令。

### 7. `get_game_state` 响应结构

`get_game_state` 命令的成功响应包含一个详细的游戏状态对象：

```json
{
  "status": "success",
  "state": {
    "is_paused": true,
    "current_day": 2,
    "project_health": 95,
    "player": {
      "entity_id": 1,
      "hp": 75,
      "max_hp": 80,
      "block": 5,
      "energy": 2.8,
      "max_energy": 3,
      "stamina": 88.5,
      "max_stamina": 100.0,
      "stress": 15,
      "buffs_debuffs": [
        {"id": "strength", "name": "力量", "stacks": 2}
      ]
    },
    "hand": [
      {
        "entity_id": 101,
        "instance_id": "...",
        "card_data_id": "strike",
        "name": "打击",
        "cost": 1,
        "type": "ATTACK",
        "description": "造成 6 点伤害。",
        "runtime_values": {"dmg_amount": 6, "block_amount": 0},
        "is_playable": true
      }
    ],
    "draw_pile_count": 12,
    "discard_pile_count": 4,
    "exhaust_pile_count": 0,
    "selected_card_entity_id": null,
    "is_targeting": false,
    "targeting_source_card_entity_id": null,
    "valid_target_entity_ids": [],
    "enemies": [
      {
        "entity_id": 201,
        "name": "Slime",
        "hp": 18,
        "max_hp": 20,
        "block": 0,
        "intent": {
          "type": "ATTACK",
          "value": 5,
          "target_id": 1,
          "cooldown_remaining": 3.2
        },
        "buffs_debuffs": []
      }
    ]
  }
}
```

**状态字段说明:**

*   `is_paused`: 游戏是否处于暂停状态
*   `current_day`: 当前 Sprint 的天数
*   `project_health`: 当前项目健康度
*   `player`: 玩家状态信息
    *   `entity_id`: 玩家实体的唯一 ID
    *   `hp`, `max_hp`: 当前/最大生命值
    *   `block`: 当前格挡值
    *   `energy`, `max_energy`: 当前/最大能量值
    *   `stamina`, `max_stamina`: 当前/最大精力值
    *   `stress`: 当前压力值
    *   `buffs_debuffs`: 状态效果列表
*   `hand`: 手牌列表
    *   每张卡牌包含：`entity_id`, `instance_id`, `card_data_id`, `name`, `cost`, `type`, `description`, `runtime_values`, `is_playable`
*   `draw_pile_count`, `discard_pile_count`, `exhaust_pile_count`: 各牌堆的卡牌数量
*   `selected_card_entity_id`: 当前选中的卡牌实体 ID
*   `is_targeting`: 是否处于目标选择模式
*   `targeting_source_card_entity_id`: 发起目标选择的卡牌实体 ID
*   `valid_target_entity_ids`: 当前可选的目标实体 ID 列表
*   `enemies`: 敌人列表
    *   每个敌人包含：`entity_id`, `name`, `hp`, `max_hp`, `block`, `intent`, `buffs_debuffs`
    *   `intent` 包含：`type`, `value`, `target_id`, `cooldown_remaining`