## Godot WebSocket 控制接口文档

**版本:** 1.4
**最后更新:** 2025-04-21

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
        "card_entity_id": 9
      }
    }
    ```

#### 3.2 服务器 (Godot) -> 客户端 (MCP)

*   **格式:** Godot 会对收到的每个有效指令发送一个标准化的 JSON 响应。
*   **标准响应格式:** 所有响应都遵循以下标准化格式：
    ```json
    {
      "status": "success" 或 "error",
      "message": "操作描述信息",
      "command_response": { /* 原始命令的完整响应 */ },
      "data": { /* 游戏状态的完整响应（仅适用于 get_game_state 命令）*/ }
    }
    ```

*   **成功响应示例:** 表明指令已被接收并成功转发给内部处理逻辑（但不保证游戏逻辑一定能成功执行）。
    ```json
    {
      "status": "success",
      "message": "运行时命令 'toggle_pause' 已成功执行",
      "command_response": {
        "status": "success"
      }
    }
    ```

*   **失败响应示例:** 表明指令解析失败、格式错误、缺少必要字段或内部处理过程中出现错误。
    ```json
    {
      "status": "error",
      "message": "找不到指定的卡牌实体: strike_1"
    }
    ```
    常见的错误信息包括：`"Invalid JSON format"`, `"Message must be a JSON object"`, `"Missing 'action' field"`, `"缺少必要参数"`, `"找不到指定的实体"` 等。

### 4. 可用指令 (`action`) 详解

以下是当前支持的 `action` 及其参数：

| Action (字符串)     | 描述                                     | 参数 (`parameters`)                                     | 状态/备注                                               |
| :------------------ | :--------------------------------------- | :------------------------------------------------------ | :------------------------------------------------------ |
| `toggle_pause`      | 切换游戏的暂停/恢复状态。                | 无                                                      | :white_check_mark: 可用                                     |
| `discard_and_draw`  | 请求执行弃牌并重抽动作。                | 无                                                      | :white_check_mark: 可用 (受每小节使用次数限制)       |
| `select_card`       | 模拟玩家选择一张手牌。                   | `card_entity_id`: int (要选择的卡牌的实体ID)           | :white_check_mark: 可用。使用卡牌的数字实体ID。 |
| `select_target`     | 模拟玩家选择一个目标实体（通常是敌人）。 | `target_entity_id`: int (目标实体的实体ID)             | :white_check_mark: 可用。使用目标的数字实体ID。 |
| `cancel_targeting`  | 取消当前的目标选择状态。                | 无                                                      | :white_check_mark: 可用。仅在目标选择模式下有效。 |
| `select_reward`     | 在奖励界面选择一个奖励。                 | `reward_entity_id`: int/String (奖励选项的实体ID或标识符) | :white_check_mark: 可用。使用奖励的实体ID或特殊标识符（如"skip"）。 |
| `get_game_state`    | 获取当前游戏的详细状态。                 | `pause_game`: Boolean (可选，默认 false) - 如果为 true，在收集状态前暂停游戏 | :white_check_mark: 可用。返回包含游戏状态的 JSON 对象。 |
| `get_available_actions` | 获取当前可用的所有玩家操作及其参数。 | 无 | :white_check_mark: 可用。返回包含所有可用操作的数组。 |
| `get_scene_tree`    | 获取场景树节点结构和基本属性的结构化数据。 | `start_node_path`: String (可选, 默认为根节点), `max_depth`: int (可选, 默认-1表示无限深度) | :white_check_mark: 可用。返回包含树结构数据的 JSON 对象 (`data` 字段)。 |
| `get_screenshot`    | 获取当前游戏画面的截图。                 | `format`: String (可选, 默认 "jpg") - 截图格式，支持 "png" 或 "jpg"<br>`quality`: int (可选, 默认 75) - 仅当 format 为 "jpg" 时有效，范围 0-100<br>`encode`: String (可选, 默认 "base64") - 编码方式，目前仅支持 "base64" | :white_check_mark: 可用。返回 Base64 编码的图像数据。 |

**关于实体 ID (`card_entity_id`, `target_entity_id`, `reward_entity_id`):**

*   `NetworkManager` 会将传入的 ID 解析为整数，并使用 `IDManager` 查找对应的实体。
*   **重要说明:** 所有参数必须放在 `parameters` 对象中，而不是直接放在命令对象中。
*   **参数命名统一:** 所有实体ID参数都使用 `*_entity_id` 格式，例如 `card_entity_id`, `target_entity_id`, `reward_entity_id`。
*   **建议:** 始终使用数字实体ID进行通信。这些 ID 可以从 `get_available_actions` 或 `get_game_state` 响应中获取。

**重要提示:**

*   所有指令的执行仍然受到 Godot 游戏内部逻辑和状态的约束（例如，游戏暂停时、卡牌冷却时、能量不足时、目标无效时，操作可能失败或被忽略）。即使 WebSocket 响应 `{"status": "success"}`，也仅表示指令被正确接收和分发，不代表游戏逻辑层面的操作一定成功。
*   Godot 端通过 `PlayerInputDispatcher` (Autoload) 将接收到的网络指令与本地输入（键盘、鼠标）统一处理，再调用相应的游戏系统（如 `CardSystem`, `CardActionSystem`）执行逻辑。

### 5. 错误处理

客户端 (MCP) 应准备处理 Godot 返回的错误响应，并根据 `message` 字段判断失败原因。

### 6. 自动获取游戏状态

当执行运行时命令（如 `toggle_pause`, `select_card` 等）时，MCP 服务器会自动尝试获取最新的游戏状态，并将其包含在响应中。这样，MCP 客户端可以在执行命令后立即获得最新的游戏状态，而无需再次发送 `get_game_state` 命令。

示例响应：

```json
{
  "status": "success",
  "message": "运行时命令 'toggle_pause' 已成功执行",
  "command_response": {
    "status": "success"
  },
  "game_state": {
    "status": "success",
    "state": {
      "is_paused": false,
      /* 其他游戏状态数据 */
    }
  }
}
```

### 7. `get_scene_tree` 响应格式

`get_scene_tree` 命令返回的数据结构如下：

```json
{
  "status": "success",
  "message": "Scene tree data retrieved successfully.",
  "command_response": {
    "start_node_path": "(root)",
    "max_depth": -1
  },
  "data": {
    "name": "root",
    "class": "Node",
    "instance_id": 1234567,
    "children": [
      {
        "name": "Main",
        "class": "Node2D",
        "instance_id": 7654321,
        "script": "res://scenes/main.gd",
        "position": {"x": 0, "y": 0},
        "rotation_deg": 0,
        "scale": {"x": 1, "y": 1},
        "visible": true,
        "children": [
          /* 子节点数据... */
        ]
      },
      /* 其他子节点... */
    ]
  }
}
```

**节点数据字段说明：**

* **基础字段（所有节点都有）：**
  * `name`: 节点名称
  * `class`: 节点类型
  * `instance_id`: 节点实例 ID
  * `children`: 子节点数组

* **脚本相关（如果有脚本）：**
  * `script`: 脚本资源路径

* **BaseEntity 特有（如果是 BaseEntity 子类）：**
  * `entity_id`: 实体 ID

* **CanvasItem 特有（Node2D, Control 等）：**
  * `visible`: 可见性

* **Node2D 特有：**
  * `position`: 位置坐标 {"x": float, "y": float}
  * `rotation_deg`: 旋转角度
  * `scale`: 缩放 {"x": float, "y": float}

* **Node3D 特有：**
  * `position`: 位置坐标 {"x": float, "y": float, "z": float}
  * `rotation_deg`: 旋转角度 {"x": float, "y": float, "z": float}
  * `scale`: 缩放 {"x": float, "y": float, "z": float}

* **Control 特有：**
  * `size`: 大小 {"x": float, "y": float}
  * `anchors`: 锚点 {"left": float, "top": float, "right": float, "bottom": float}

### 8. 未来扩展

未来可能会添加更多指令，例如：

*   更复杂的卡牌交互指令。
*   控制游戏设置或流程的指令。

### 9. `get_available_actions` 响应结构

`get_available_actions` 命令的成功响应包含一个当前可用操作的数组：

```json
{
  "status": "success",
  "message": "Available actions retrieved.",
  "available_actions": [
    // 总是可用的基础动作
    {"name": "toggle_pause"},
    {"name": "get_game_state"},

    // 有条件可用的动作
    {"name": "discard_and_draw"}, // 当 DiscardAndDrawSystem.is_available() 为 true 时

    // 带参数的动作
    {
      "name": "select_card",
      "valid_card_entity_ids": [101, 105, 108] // 仅包含当前可打出的卡牌实体 ID
    },

    // 目标选择模式下的动作
    {
      "name": "select_target",
      "valid_target_entity_ids": [201, 202] // 包含所有有效目标的实体 ID
    },
    {"name": "cancel_targeting"}, // 当处于目标选择模式时

    // 奖励选择模式下的动作
    {
      "name": "select_reward",
      "valid_reward_entity_ids": ["reward_card_strike", "reward_card_defend", "skip"] // 奖励选项的唯一标识符
    }
  ]
}
```

**注意事项：**

* 返回的动作列表是动态生成的，只包含当前游戏状态下真正可以执行的操作。
* 对于需要参数的动作（如 `select_card`, `select_target`, `select_reward`），响应中会包含一个有效参数值的列表。
* 当处于目标选择模式时，会返回 `cancel_targeting` 动作，允许取消当前选择。

### 10. `get_game_state` 响应结构

`get_game_state` 命令的成功响应遵循标准化格式，包含一个详细的游戏状态对象：

```json
{
  "status": "success",
  "message": "游戏状态获取成功",
  "data": {
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

### 11. `get_screenshot` 响应结构

`get_screenshot` 命令的成功响应包含 Base64 编码的图像数据：

```json
{
  "status": "success",
  "message": "截图获取成功",
  "data": {
    "format": "jpg",  // 或 "png"，取决于请求参数
    "encoding": "base64",
    "image_data": "..." // Base64 编码的图像数据字符串
  }
}
```

**响应数据字段说明：**

* `format`: 图像格式，"jpg" 或 "png"
* `encoding`: 编码方式，目前仅支持 "base64"
* `image_data`: Base64 编码的图像数据，可以直接用于 HTML `<img>` 标签的 src 属性（需添加前缀 "data:image/jpeg;base64,"或"data:image/png;base64,"）

**错误响应示例：**

```json
{
  "status": "error",
  "message": "无效的截图格式 'gif'，支持 'png' 或 'jpg'"
}
```

**注意事项：**

* 根据游戏图像质量和分辨率，返回的 Base64 字符串可能较大，特别是使用 PNG 格式时。
* JPEG 格式提供更小的文件大小，但有损压缩；PNG 提供无损压缩但文件较大。
* 可以通过 `quality` 参数调整 JPEG 压缩质量，平衡大小和质量。
