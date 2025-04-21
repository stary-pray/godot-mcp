# MCP 游戏截图 API 文档

## 功能概述

MCP 游戏截图 API 允许通过 WebSocket 接口获取游戏当前画面的截图。截图将保存到文件系统，并返回文件的绝对路径，而不是直接返回图像数据。

## API 端点

- **Action**: `get_screenshot`
- **接口**: WebSocket

## 参数

| 参数名 | 类型 | 必填 | 默认值 | 描述 |
|--------|------|------|--------|------|
| format | string | 否 | "png" | 截图格式，支持 "png" 或 "jpg" |
| quality | integer | 否 | 75 | 仅当 format 为 "jpg" 时有效，范围 0-100，表示 JPEG 压缩质量 |
| save_dir | string | 否 | "screenshots/" | 保存目录，相对于 Godot 的 `user://` 目录 |
| filename | string | 否 | 自动生成 | 不含扩展名的文件名，如果不提供，将使用时间戳自动生成 |

> **注意**: 之前版本支持的 `encode` 参数已被弃用，如果提供该参数将被忽略并发出警告。

## 响应

### 成功响应

```json
{
  "status": "success",
  "message": "截图已保存",
  "data": {
    "format": "png",
    "absolute_path": "/Users/username/Library/Application Support/Godot/app_userdata/project_name/screenshots/screenshot_20250421_125303.png"
  }
}
```

### 响应字段说明

| 字段 | 类型 | 描述 |
|------|------|------|
| status | string | 响应状态，成功为 "success"，失败为 "error" |
| message | string | 响应消息，提供操作结果的简短描述 |
| data | object | 响应数据对象 |
| data.format | string | 截图格式，"png" 或 "jpg" |
| data.absolute_path | string | 截图文件的绝对路径 |

### 错误响应

```json
{
  "status": "error",
  "message": "截图失败: 无法保存文件",
  "error": {
    "code": "SAVE_ERROR",
    "details": "无法写入指定目录"
  }
}
```

## 使用示例

### JavaScript 示例

```javascript
// 使用 WebSocket 客户端
const ws = new WebSocket('ws://localhost:9080');

ws.onopen = () => {
  // 基本用法
  ws.send(JSON.stringify({
    action: 'get_screenshot'
  }));
  
  // 高级用法
  ws.send(JSON.stringify({
    action: 'get_screenshot',
    parameters: {
      format: 'jpg',
      quality: 90,
      save_dir: 'my_screenshots/',
      filename: 'battle_scene'
    }
  }));
};

ws.onmessage = (event) => {
  const response = JSON.parse(event.data);
  if (response.status === 'success') {
    console.log('截图已保存到:', response.data.absolute_path);
  } else {
    console.error('截图失败:', response.message);
  }
};
```

### TypeScript 示例

```typescript
// 使用 MCP 客户端
import { MCPClient } from '@yuki/mcp-client';

async function captureScreenshot() {
  const client = new MCPClient('localhost', 9080);
  
  try {
    const response = await client.sendCommand('get_screenshot', {
      format: 'jpg',
      quality: 90,
      save_dir: 'screenshots/',
      filename: 'gameplay'
    });
    
    if (response.status === 'success') {
      console.log('截图已保存到:', response.data.absolute_path);
      return response.data.absolute_path;
    } else {
      throw new Error(response.message);
    }
  } catch (error) {
    console.error('截图请求失败:', error);
    throw error;
  }
}
```

## 注意事项

1. **文件路径**: 返回的 `absolute_path` 是服务器端的绝对路径，如果客户端在不同的机器上，可能无法直接访问该文件。

2. **目录创建**: 如果指定的 `save_dir` 不存在，系统会尝试创建该目录。如果创建失败，将返回错误。

3. **性能考虑**: 截图操作可能会暂时影响游戏性能，特别是在高分辨率或资源受限的环境中。

4. **存储空间**: 频繁截图可能会占用大量磁盘空间，特别是使用 PNG 格式时。建议定期清理不需要的截图。

5. **弃用说明**: 之前版本支持通过 `encode` 参数返回 base64 编码的图像数据，该功能已被弃用，以提高性能并减少内存使用。

## 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 2.0 | 2025-04-21 | 移除 base64 编码支持，改为仅返回文件路径 |
| 1.0 | 2025-03-15 | 初始版本，支持 base64 编码和文件保存 |
