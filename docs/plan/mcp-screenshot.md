
**任务：使 `godot-mcp` WebSocket 接口实现与 README V1.4 保持一致**

**目标：**

修改 `src/index.ts` 中的 MCP 服务器实现，确保其处理 WebSocket 命令（特别是 `get_screenshot`, `get_scene_tree`, `get_game_state`）的行为和响应格式与 `README_Websocket_Interface.md` (版本 1.4) 中的定义完全匹配。

**背景：**

当前 `src/index.ts` 中的实现与最新的 README V1.4 在以下几个关键方面存在差异：

1.  `get_screenshot` 功能在 README 中已定义，但在代码中完全缺失。
2.  `get_scene_tree` 命令的成功响应格式不正确（数据未放在 `data` 字段下）。
3.  `get_game_state` 命令的成功响应格式与 README V1.4 Section 10 中的示例结构存在细微差别。

**具体任务：**

请在 `src/index.ts` 文件中完成以下修改：

1.  **实现 `get_screenshot` 功能 (必需)**
    *   **添加 MCP 工具定义:** 在 `setupToolHandlers` 方法内的 `ListToolsRequestSchema` 处理器中，添加一个新的工具定义对象：
        *   `name`: `"get_screenshot"`
        *   `description`: (参考 README Section 4) "获取当前游戏画面的截图。"
        *   `inputSchema`:
            *   `type`: `"object"`
            *   `properties`:
                *   `host` (String, optional, default: 'localhost')
                *   `port` (Integer, optional, default: 9080)
                *   `format` (String, optional, default: 'jpg', enum: ['jpg', 'png'])
                *   `quality` (Integer, optional, default: 75, minimum: 0, maximum: 100) - 描述中注明仅 jpg 有效
                *   `encode` (String, optional, default: 'base64', enum: ['base64']) - 描述中注明目前仅支持 base64
            *   `required`: [] (因为都有默认值或非必需)
    *   **添加工具处理器:**
        *   创建一个新的异步方法 `handleGetScreenshot(args: any)`。
        *   在 `CallToolRequestSchema` 处理器的 `switch` 语句中添加 `case 'get_screenshot': return await this.handleGetScreenshot(request.params.arguments);`。
    *   **实现 `handleGetScreenshot` 逻辑:**
        *   接收参数并进行标准化（确保 `host`, `port`, `format`, `quality`, `encode` 使用正确的默认值）。
        *   创建 `WebSocketClient` 实例。
        *   **重要:** 准备发送给 Godot 的参数 `godotParams`。根据 README Section 4，这些参数名是 `format`, `quality`, `encode` (均为小写)。确保将 MCP 工具接收到的参数正确映射并传递给 `sendCommand`。
        *   调用 `client.sendCommand('get_screenshot', godotParams)`。
        *   处理成功响应 (`response.status === 'success'`)：
            *   构造返回给 MCP 客户端的 JSON 对象。
            *   **必须** 严格按照 README Section 11 的格式：
                ```json
                {
                  "status": "success",
                  "message": "截图获取成功",
                  "data": {
                    "format": response.data.format, // 从 Godot 响应中获取
                    "encoding": response.data.encoding, // 从 Godot 响应中获取
                    "image_data": response.data.image_data // 从 Godot 响应中获取
                  }
                }
                ```
            *   将此 JSON 对象字符串化后放入 `content[0].text`。
        *   处理错误响应 (`response.status === 'error'`) 或捕获异常：使用 `this.createErrorResponse` 返回适当的错误信息。

2.  **修正 `get_scene_tree` 响应格式 (必需)**
    *   **定位:** 找到 `handleGetSceneTree` 方法。
    *   **修改:** 在处理成功响应的代码块 (`if (response.status === 'success')`) 中，修改返回给 MCP 客户端的 JSON 结构。
    *   **当前错误结构 (类似):**
        ```javascript
        {
          status: "success",
          message: "Scene tree data retrieved.",
          sceneTree: response.data // <--- 错误字段名
        }
        ```
    *   **目标正确结构 (参考 README Section 7):**
        ```javascript
        {
          status: "success",
          message: "Scene tree data retrieved successfully.", // 可以更新 message
          command_response: response.command_response, // 保留 Godot 返回的命令响应
          data: response.data // <--- 正确字段名，直接使用 Godot 返回的树数据
        }
        ```
        *注意：确保将 Godot 响应中实际的树数据 (`response.data`) 赋值给 MCP 响应的 `data` 字段。*

3.  **调整 `get_game_state` 响应格式 (推荐)**
    *   **定位:** 找到 `handleGetGameState` 方法。
    *   **当前结构 (类似):**
        ```javascript
        {
          status: "success",
          message: "游戏状态获取成功",
          data: response // <--- 包含 Godot 的整个响应对象
        }
        ```
    *   **目标结构 (更符合 README Section 10 示例):**
        ```javascript
         {
          status: "success",
          message: "游戏状态获取成功",
          data: response.data // <--- 直接使用 Godot 响应中的 data 字段内容
        }
        ```
        *说明：假设从 Godot 收到的 `response` 结构是 `{ status: 'success', message: '...', data: { status: 'success', state: { ... } } }`。调整后，MCP 响应的 `data` 字段将直接包含 `{ status: 'success', state: { ... } }` 这部分，使 AI 更容易访问 `state`。*
    *   **注意:** 这个修改是为了更好地匹配 README 中的示例，提高 AI 使用的便利性。

**测试要求：**

*   在修改后，请务必彻底测试所有涉及的 WebSocket 命令 (`get_screenshot`, `get_scene_tree`, `get_game_state` 以及通过 `mcp_godot_send_runtime_command` 调用的其他命令)。
*   验证命令的参数传递（包括大小写转换，如 `get_scene_tree` 和 `get_screenshot` 需要将 MCP 的 camelCase 转为 Godot 的 snake_case）是否正确。
*   验证成功和失败情况下的响应格式是否严格符合 `README_Websocket_Interface.md` V1.4 的定义。
*   特别关注 `get_screenshot` 的不同 `format` 和 `quality` 参数的效果。
*   验证 `mcp_godot_send_runtime_command` 在成功后是否仍能正确附加 `game_state`。

**参考文件：**

*   `src/index.ts` (待修改)
*   `README_Websocket_Interface.md` (版本 1.4，作为规范)

请在完成后通知我，以便进行代码审查和合并。