## Godot-MCP: Yuki-Godot 运行时控制 - 蓝图

**核心目标:**

扩展 `godot-mcp` 服务器，使其能够通过 WebSocket 连接向运行中的 `yuki-godot` 实例发送运行时指令，并接收其状态响应。通信协议遵循 `yuki-godot/docs/current/README_Websocket_Interface.md` 文件中定义的规范。

**核心策略:**

1.  **新的 MCP 工具:** 在 `godot-mcp` 中引入一个名为 `send_runtime_command` 的新工具，供 AI 助手（如通过 Cursor 的 Claude）调用。
2.  **WebSocket 客户端:** 在 `godot-mcp` 的工具处理器内部实现一个 WebSocket 客户端，负责建立与 `yuki-godot` 实例的连接、发送指令以及等待响应。
3.  **标准通信:** 确保发送的指令和预期的响应严格遵守 `README_Websocket_Interface.md` 中定义的 JSON 格式。

**组件与数据流 (在 `godot-mcp` 内部):**

1.  **AI 助手 (例如 Claude/Cursor):**
    *   生成运行时控制请求（例如，“暂停游戏”、“选择卡牌 X”）。
    *   使用适当的参数调用 `godot-mcp` 中的 `send_runtime_command` 工具。

2.  **`godot-mcp` 服务器 (`index.ts`):**
    *   **工具定义:** 定义 `send_runtime_command` 工具及其输入模式（`host`, `port`, `action`, `parameters`, `projectPath` (可选)）。
    *   **工具处理器 (`handleSendRuntimeCommand`):**
        *   接收来自 AI 的工具调用请求。
        *   提取连接信息（`host`, `port`）和指令内容（`action`, `parameters`）。
        *   **WebSocket 客户端逻辑:**
            *   连接到 `yuki-godot` WebSocket 服务器 (默认 `<ws://localhost:9080`>)。
            *   根据规范将指令格式化为 JSON 字符串。
            *   通过 WebSocket 连接发送 JSON 指令。
            *   等待 `yuki-godot` 的响应（实现超时机制）。
            *   接收并解析 JSON 响应 (`{"status": "success"}` 或 `{"status": "error", "message": "..."}`)。
            *   关闭 WebSocket 连接。
        *   将结果（成功或带消息的错误）格式化为 MCP 响应。
        *   将响应发送回 AI。

3.  **WebSocket 客户端库 (例如 `ws`):**
    *   Node.js 模块，负责处理 WebSocket 通信的技术细节（连接、发送、接收、错误处理）。

**依赖项 (对于 `godot-mcp`):**

*   Node.js WebSocket 客户端库 (推荐: `ws`)。

---

## Godot-MCP: Yuki-Godot 运行时控制 - 路线图

**阶段 0: 前提条件 (Godot 端已满足)**

*   `yuki-godot` 已准备好在端口 9080 上接受 WebSocket 连接。
*   `yuki-godot` 已实现 `README_Websocket_Interface.md` 中描述的指令和响应结构。
*   `yuki-godot` 已能将接收到的指令正确路由到 `PlayerInputDispatcher`。

**阶段 1: MCP 工具定义与设置 (约 25%)**

1.  **添加依赖:** :white_check_mark: 在 `godot-mcp/package.json` 的 `dependencies` 中添加 `ws` 库 (`npm install ws`)。在 `devDependencies` 中添加 `@types/ws` (`npm install --save-dev @types/ws`)。(完成 ✅)
2.  **定义新工具:** :white_check_mark: 在 `godot-mcp/src/index.ts` 中，扩展 `ListToolsRequestSchema` 处理逻辑：
    *   为 `send_runtime_command` 添加新的工具对象。
    *   定义 `inputSchema`，包含以下属性:
        *   `projectPath` (string, optional, description: "相关 yuki-godot 项目的路径，用于提供上下文")
        *   `host` (string, optional, default: "localhost", description: "运行中 Godot 实例的主机名或 IP")
        *   `port` (integer, optional, default: 9080, description: "运行中 Godot 实例的 WebSocket 端口")
        *   `action` (string, required, description: "要发送的操作指令 (参见 yuki-godot WebSocket 接口文档)")
        *   `parameters` (object, optional, description: "包含操作所需参数的 JSON 对象")
3.  **工具处理器结构:** :white_check_mark: 在 `index.ts` 的 `CallToolRequestSchema` 处理器中：
    *   添加 `case 'send_runtime_command':` 分支。
    *   创建一个新的异步函数 `handleSendRuntimeCommand(args: any)`，并在 `case` 中调用它。

**阶段 2: WebSocket 客户端实现 (约 45%)**

1.  **实现 `handleSendRuntimeCommand(args: any)`:** :white_large_square:
    *   从 `ws` 库导入 `WebSocket` 类。
    *   从 `args` 中提取 `host`, `port`, `action`, `parameters`。为 `host` 和 `port` 设置默认值。
    *   验证 `action` 是否存在。
    *   构建要发送的 JSON 对象: `{ action: args.action, parameters: args.parameters || {} }`。
    *   **使用 `Promise` 实现核心逻辑:**
        ```typescript
        return new Promise((resolve, reject) => {
            const wsUrl = `ws://${host}:${port}`;
            const ws = new WebSocket(wsUrl);
            let responseReceived = false; // 标记是否已收到响应

            // 设置超时
            const timeout = setTimeout(() => {
                if (!responseReceived) {
                    ws.terminate(); // 强制关闭连接
                    // 使用中文错误信息
                    reject(this.createErrorResponse(`连接到 ${wsUrl} 的 WebSocket 请求超时 (5秒)`, ["检查 yuki-godot 是否正在运行", "确认主机和端口号是否正确", "检查防火墙设置"]));
                }
            }, 5000); // 5 秒超时

            ws.on('open', () => {
                this.logDebug(`WebSocket 已连接到 ${wsUrl}`);
                // 构建消息体，确保 parameters 至少是空对象
                const messageBody = { action: args.action, parameters: args.parameters || {} };
                const message = JSON.stringify(messageBody);
                this.logDebug(`发送 WebSocket 消息: ${message}`);
                ws.send(message);
            });

            ws.on('message', (data) => {
                responseReceived = true; // 标记收到响应
                clearTimeout(timeout); // 清除超时计时器
                this.logDebug(`收到 WebSocket 消息: ${data.toString()}`);
                try {
                    const response = JSON.parse(data.toString());
                    if (response.status === 'success') {
                        resolve({
                            content: [{ type: 'text', text: `指令 '${args.action}' 已成功执行。` }]
                        });
                    } else if (response.status === 'error') {
                        // Godot 报告的错误
                        reject(this.createErrorResponse(`Godot 运行时错误 (指令: '${args.action}'): ${response.message || '未知错误'}`, ["检查发送的参数", "确认 yuki-godot 中的游戏状态"]));
                    } else {
                        // 非预期的响应格式
                        reject(this.createErrorResponse(`从 Godot 收到非预期的响应格式: ${data.toString()}`, ["检查 yuki-godot NetworkManager 的响应格式"]));
                    }
                } catch (e) {
                    reject(this.createErrorResponse(`解析来自 Godot 的 JSON 响应失败: ${data.toString()}`, ["检查 yuki-godot NetworkManager 的响应格式"]));
                } finally {
                     ws.close(); // 收到响应后关闭连接
                }
            });

            ws.on('error', (error) => {
                clearTimeout(timeout); // 清除超时计时器
                this.logDebug(`WebSocket 错误: ${error.message}`);
                reject(this.createErrorResponse(`连接 ${wsUrl} 时发生 WebSocket 错误: ${error.message}`, ["检查 yuki-godot 是否正在运行", "确认主机和端口号是否正确"]));
            });

            ws.on('close', (code, reason) => {
                 this.logDebug(`WebSocket 已关闭。代码: ${code}, 原因: ${reason}`);
                 // 如果连接在收到响应前意外关闭，且不是由我们主动关闭的
                 if (!responseReceived && ws.readyState !== WebSocket.OPEN && ws.readyState !== WebSocket.CONNECTING) {
                    clearTimeout(timeout);
                    reject(this.createErrorResponse(`WebSocket 连接在收到响应前意外关闭 (代码: ${code})，目标: ${wsUrl}`, ["检查 yuki-godot 的稳定性", "网络问题"]));
                 }
            });
        });
        ```
    *   确保 `this.logDebug` 和 `this.createErrorResponse` 方法已正确实现且可用。

**阶段 3: 集成测试与错误处理 (约 20%)**

1.  **测试环境:** :white_large_square: 启动 `yuki-godot` (确保 WebSocket 服务器运行)。启动 `godot-mcp` (例如通过 Cursor 或 Cline)。
2.  **测试调用:** :white_large_square: 通过 AI 接口调用 `send_runtime_command` 工具，测试各种场景：
    *   无参数的有效指令 (例如 `action: "toggle_pause"`)。
    *   带参数的有效指令 (例如 `action: "select_card", parameters: {"card_id": "有效的卡牌ID"}`)。
    *   带有无效参数或 ID 的指令。
    *   缺少 `action` 字段的指令。
    *   尝试连接未运行的 Godot 服务器。
    *   尝试连接错误的主机或端口。
3.  **验证:** :white_large_square:
    *   检查 `godot-mcp` 返回的响应 (成功/错误)。
    *   检查 `yuki-godot` 的控制台输出，确认指令是否被正确处理，以及是否有错误日志。
    *   观察 `yuki-godot` 中的游戏行为，确保指令触发了预期的动作。
4.  **错误处理优化:** :white_large_square: 根据测试结果，改进 `handleSendRuntimeCommand` 中的错误信息，使其对 AI 更清晰、更有指导性。

**阶段 4: 文档与收尾 (约 10%)**

1.  **更新 README:** :white_large_square: 在 `godot-mcp/README.md` 中添加关于新工具 `send_runtime_command` 的说明、参数解释和使用示例。
2.  **代码注释:** :white_large_square: 为 `handleSendRuntimeCommand` 中的代码添加注释，解释其逻辑。
3.  **代码清理:** :white_large_square: 移除不必要的调试代码，优化实现逻辑。

**预期成果:**

`godot-mcp` 服务器将具备通过 WebSocket向 `yuki-godot` 发送运行时控制指令的能力，并将执行结果（成功/失败）反馈给调用它的 AI 助手，从而实现对游戏的外部控制。