## 修复说明：godot-mcp (TypeScript/Node.js 项目)

**目标：** 配合 `yuki-godot` 项目的修复，确保 MCP 服务器能正确地发送命令和处理响应。

**参考问题描述：** `mcp-issue.md`

**主要问题点：** 虽然 MCP 发送格式基本正确，但需确保使用的参数名与 Godot 端最终统一的名称一致。

**修复步骤：**

1.  **【一致性】（条件性）调整发送的参数名称 (位于 `src/index.ts`)**
    *   **目标：** 确保 MCP 发送给 Godot 的参数名（如 `card_id` 或 `card_entity_id`）**(决定使用 card_entity_id 了)** 与 `yuki-godot` 项目最终统一使用的名称一致。
    *   **操作：**
        *   **确认 Godot 端使用的名称：** 与负责 `yuki-godot` 的开发人员沟通，确认他们最终统一使用的参数名（例如，是 `card_id` 还是 `card_entity_id`）。
        *   **检查 MCP 定义：** 在 `src/index.ts` 的 `setupToolHandlers` 函数中，找到 `mcp_godot_send_runtime_command` 的 `inputSchema`。
        *   **检查 MCP 实现：** 在 `src/index.ts` 的 `handleSendRuntimeCommand` 函数中，检查传递给 `client.sendCommand` 的 `args.parameters` 对象。
        *   **修改（如果需要）：** 如果 Godot 端修改了参数名，确保 `inputSchema` 的描述和 `handleSendRuntimeCommand` 中使用的参数键名与之匹配。**注意：参数必须始终嵌套在 `parameters` 对象中发送。**
    *   **验证：** 确保调用 `mcp_godot_send_runtime_command` 并传递相应参数时，Godot 端能正确接收并识别。

2.  **【健壮性】（可选）增强 WebSocket 错误日志 (位于 `src/index.ts`)**
    *   **目标：** 在 WebSocket 连接或发送失败时，提供更详细的调试信息。
    *   **操作：** 在 `WebSocketClient.sendCommand` 函数的错误处理部分（`catch` 块，`ws.on('error')`，`ws.on('close')` 的超时/未响应逻辑），考虑加入对 `messageBody`（即尝试发送的 JSON 对象）的日志记录。
    *   **示例：**
        ```typescript
        // 在 ws.on('error') 或 catch 块中
        reject(new Error(`WebSocket 错误: ${error.message}. Attempted command: ${JSON.stringify(messageBody)}`));

        // 在超时或意外关闭的 reject 中
        reject(new Error(`WebSocket ... 关闭 ... Attempted command: ${JSON.stringify(messageBody)}`));
        ```

3.  **【文档/Schema】更新 MCP 工具描述 (位于 `src/index.ts`)**
    *   **目标：** 确保 `mcp_godot_send_runtime_command` 工具的描述清晰准确。
    *   **操作：** 在 `setupToolHandlers` 中，更新 `mcp_godot_send_runtime_command` 的 `description` 和 `inputSchema.properties.parameters.description`，明确指出参数需要嵌套在 `parameters` 对象内，并列出关键命令（如 `select_card`）预期使用的参数名（与 Godot 端统一后的名称）。