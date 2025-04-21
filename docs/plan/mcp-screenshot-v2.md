
**任务：更新 `godot-mcp` 以符合 WebSocket 接口文档 V1.5**

**目标：**

修改 `src/index.ts` 中的 MCP 服务器实现，使其完全符合 `README_Websocket_Interface.md` **版本 1.5** 的规范，特别是 `get_screenshot` action 的新实现方式（保存文件并返回路径）以及之前指出的 `get_scene_tree` 和 `get_game_state` 的响应格式问题。

**背景：**

Godot 端的 WebSocket 接口文档已更新至 V1.5。最显著的变化是 `get_screenshot` action：

*   它现在将截图**保存到本地文件系统**（在 Godot 的 `user://` 目录下）。
*   它不再返回 Base64 编码的图像数据，而是返回包含所保存文件**绝对路径**的 JSON 响应。

此外，之前指出的 `get_scene_tree` 和 `get_game_state` 响应格式与 README 不完全匹配的问题仍需修复，以符合 V1.5 文档中的定义（Sections 7 和 10）。

**具体任务：**

请在 `src/index.ts` 文件中完成以下修改：

1.  **实现/更新 `get_screenshot` 功能 (必需 - 适配 V1.5)**
    *   **更新/添加 MCP 工具定义:** 在 `setupToolHandlers` 方法内的 `ListToolsRequestSchema` 处理器中，找到或添加 `get_screenshot` 工具定义。确保其 `description` 和 `inputSchema` 反映 V1.5 的要求：
        *   `name`: `"get_screenshot"`
        *   `description`: (参考 README V1.5 Section 4) "获取当前游戏画面的截图，将其保存到本地文件，并返回文件的绝对路径。"
        *   `inputSchema`:
            *   `type`: `"object"`
            *   `properties`:
                *   `host` (String, optional, default: 'localhost')
                *   `port` (Integer, optional, default: 9080)
                *   `format` (String, optional, default: 'png', enum: ['png', 'jpg']) - 截图格式
                *   `quality` (Integer, optional, default: 75, minimum: 0, maximum: 100) - JPG 质量 (仅 format='jpg' 时有效)
                *   `save_dir` (String, optional, default: 'screenshots/') - 相对于 Godot `user://` 的保存目录
                *   `filename` (String, optional) - 不含扩展名的文件名 (若省略，Godot 会生成时间戳文件名)
            *   `required`: []
    *   **实现/更新工具处理器:**
        *   创建或修改异步方法 `handleGetScreenshot(args: any)`。
        *   在 `CallToolRequestSchema` 处理器的 `switch` 语句中确保 `case 'get_screenshot': return await this.handleGetScreenshot(request.params.arguments);` 存在且正确。
    *   **实现 `handleGetScreenshot` 逻辑 (适配 V1.5):**
        *   接收参数并进行标准化（处理 `host`, `port` 及 `get_screenshot` 特定参数的默认值）。
        *   创建 `WebSocketClient` 实例。
        *   **重要:** 准备发送给 Godot 的 `parameters` 对象 (`godotParams`)。键名必须是 **snake_case**，与 README V1.5 Section 4 中定义的参数名一致：`format`, `quality`, `save_dir`, `filename`。确保将 MCP 工具接收到的参数正确映射到这些 snake_case 键上。
        *   调用 `client.sendCommand('get_screenshot', godotParams)`。
        *   处理成功响应 (`response.status === 'success'`)：
            *   构造返回给 MCP 客户端（LLM）的 JSON 对象。
            *   **必须** 严格按照 README V1.5 Section 11 的格式：
                ```json
                {
                  "status": "success",
                  "message": "截图已保存", // 或使用 Godot 返回的 message
                  "data": {
                    "format": response.data.format,       // 从 Godot 响应中获取
                    "absolute_path": response.data.absolute_path // 从 Godot 响应中获取
                  }
                }
                ```
            *   将此 JSON 对象字符串化后放入 `content[0].text`。
        *   处理错误响应 (`response.status === 'error'`) 或捕获异常：使用 `this.createErrorResponse` 返回适当的错误信息，可以包含 Godot 返回的 `message`。

2.  **修正 `get_scene_tree` 响应格式 (必需 - 保持与 V1.5 一致)**
    *   **定位:** 找到 `handleGetSceneTree` 方法。
    *   **修改:** 在处理成功响应的代码块 (`if (response.status === 'success')`) 中，确保返回给 MCP 客户端的 JSON 结构符合 README V1.5 Section 7：
        ```javascript
        {
          status: "success",
          message: response.message || "Scene tree data retrieved successfully.",
          command_response: response.command_response, // 保留 Godot 返回的命令响应
          data: response.data // <--- 确保场景树数据在此字段
        }
        ```

3.  **调整 `get_game_state` 响应格式 (必需 - 保持与 V1.5 一致)**
    *   **定位:** 找到 `handleGetGameState` 方法。
    *   **修改:** 在处理成功响应的代码块 (`if (response.status === 'success')`) 中，确保返回给 MCP 客户端的 JSON 结构符合 README V1.5 Section 10 的示例，即将 Godot 响应中的 `data` 部分直接赋值给 MCP 响应的 `data` 字段：
        ```javascript
         {
          status: "success",
          message: response.message || "游戏状态获取成功",
          data: response.data // <--- 直接使用 Godot 响应中的 data 字段内容 ({status: 'success', state: {...}})
        }
        ```

**测试要求：**

*   在修改后，请务必彻底测试所有涉及的 WebSocket 命令 (`get_screenshot`, `get_scene_tree`, `get_game_state` 以及通过 `mcp_godot_send_runtime_command` 调用的其他命令)。
*   **重点测试 `get_screenshot`:**
    *   验证不同参数（`format`, `quality`, `save_dir`, `filename`，包括省略时的默认行为）是否按预期工作。
    *   验证返回的 `absolute_path` 是否正确且文件确实存在于该路径。
    *   验证返回的 `format` 是否与请求或默认值一致。
    *   测试错误情况（例如，无效格式、保存失败）。
*   验证 `get_scene_tree` 和 `get_game_state` 的成功响应格式是否严格符合 README V1.5 Sections 7 和 10。
*   验证参数大小写转换（MCP camelCase -> Godot snake_case）在 `get_screenshot` 和 `get_scene_tree` 中是否正确。
*   验证 `mcp_godot_send_runtime_command` 在成功后是否仍能正确附加 `game_state`。
*   **环境假设:** 测试时需确保运行 MCP 服务器的 Node.js 进程有权限读取 Godot 保存截图的目标路径（通常在同一台机器上运行即可）。

**参考文件：**

*   `src/index.ts` (待修改)
*   `README_Websocket_Interface.md` (**版本 1.5**，作为规范)

请在完成后通知我们，以便进行代码审查和合并。