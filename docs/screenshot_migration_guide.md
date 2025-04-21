# MCP 游戏截图 API 迁移指南

本文档提供了从旧版 base64 编码截图 API 迁移到新版基于文件系统的截图 API 的详细指南。

## 变更概述

MCP 游戏截图 API 已经进行了重大更新，主要变更如下：

1. **移除 base64 编码支持**：不再支持通过 `encode` 参数请求 base64 编码的图像数据
2. **仅返回文件路径**：API 现在只返回保存到文件系统的截图路径
3. **性能优化**：新 API 显著减少了内存使用和网络传输量

## 为什么进行这些变更？

### 性能考虑

旧版 API 中的 base64 编码方式存在以下问题：

- **内存占用高**：base64 编码会使数据大小增加约 33%
- **网络传输效率低**：通过 WebSocket 传输大量 base64 数据可能导致延迟和连接问题
- **客户端解码开销**：客户端需要额外的处理步骤来解码 base64 数据

### 更好的架构

新 API 采用基于文件系统的方法，具有以下优势：

- **更低的资源消耗**：减少内存使用和 CPU 负载
- **更快的响应时间**：API 调用完成更快，因为不需要编码大量数据
- **更灵活的文件管理**：客户端可以根据需要访问或处理截图文件

## 迁移步骤

### 1. 更新 API 调用

#### 旧版 API 调用（使用 base64）

```javascript
// 旧版 - 请求 base64 编码的图像数据
client.sendCommand('get_screenshot', {
  format: 'png',
  encode: 'base64'
}).then(response => {
  // 处理 base64 编码的图像数据
  const imageData = response.data.base64;
  displayImage(`data:image/png;base64,${imageData}`);
});
```

#### 新版 API 调用（使用文件路径）

```javascript
// 新版 - 获取文件路径
client.sendCommand('get_screenshot', {
  format: 'png',
  save_dir: 'screenshots/',
  filename: 'game_screenshot'
}).then(response => {
  // 处理文件路径
  const imagePath = response.data.absolute_path;
  displayImageFromPath(imagePath);
});
```

### 2. 更新图像显示逻辑

#### 旧版图像显示（使用 base64）

```javascript
function displayImage(base64Data) {
  const img = document.createElement('img');
  img.src = base64Data;
  document.getElementById('screenshot-container').appendChild(img);
}
```

#### 新版图像显示（使用文件路径）

根据您的应用架构，有几种处理文件路径的方法：

##### 方法 1: 使用文件 URL（本地应用）

```javascript
function displayImageFromPath(filePath) {
  const img = document.createElement('img');
  img.src = `file://${filePath}`; // 注意：这仅适用于本地应用，如 Electron
  document.getElementById('screenshot-container').appendChild(img);
}
```

##### 方法 2: 通过文件服务器（网络应用）

```javascript
function displayImageFromPath(filePath) {
  // 将绝对路径转换为相对于文件服务器的 URL
  const fileName = filePath.split('/').pop();
  const fileUrl = `http://your-file-server.com/screenshots/${fileName}`;
  
  const img = document.createElement('img');
  img.src = fileUrl;
  document.getElementById('screenshot-container').appendChild(img);
}
```

##### 方法 3: 实现文件传输（跨平台应用）

```javascript
async function displayImageFromPath(filePath) {
  // 请求服务器将文件传输到客户端
  const response = await fetch(`/api/transfer-file?path=${encodeURIComponent(filePath)}`);
  const blob = await response.blob();
  
  const img = document.createElement('img');
  img.src = URL.createObjectURL(blob);
  document.getElementById('screenshot-container').appendChild(img);
}
```

### 3. 处理错误和兼容性

为了确保平滑迁移，您可以添加兼容性代码来处理旧版和新版 API：

```javascript
async function takeScreenshot(options = {}) {
  try {
    // 移除旧版 encode 参数
    const { encode, ...validOptions } = options;
    
    if (encode) {
      console.warn('警告: encode 参数已被弃用，将被忽略');
    }
    
    const response = await client.sendCommand('get_screenshot', validOptions);
    
    // 检查响应格式，处理新旧版本
    if (response.data.base64) {
      // 旧版响应 - 为了向后兼容
      console.warn('收到旧版 API 响应，请更新您的服务器');
      return {
        type: 'base64',
        data: response.data.base64,
        format: response.data.format
      };
    } else if (response.data.absolute_path) {
      // 新版响应
      return {
        type: 'file',
        path: response.data.absolute_path,
        format: response.data.format
      };
    } else {
      throw new Error('未知的响应格式');
    }
  } catch (error) {
    console.error('截图失败:', error);
    throw error;
  }
}
```

## 常见迁移问题

### Q: 如何在网页应用中显示截图？

**A:** 由于浏览器安全限制，网页应用无法直接访问服务器文件系统。您需要：

1. 设置一个文件服务器来提供这些截图
2. 实现一个 API 端点来传输文件到客户端
3. 使用相对 URL 而不是绝对文件路径

```javascript
// 服务器端代码 (Node.js 示例)
app.get('/api/screenshots/:filename', (req, res) => {
  const screenshotDir = path.join(process.env.GODOT_USER_DIR, 'screenshots');
  const filePath = path.join(screenshotDir, req.params.filename);
  
  // 安全检查，确保文件在允许的目录中
  if (!filePath.startsWith(screenshotDir)) {
    return res.status(403).send('访问被拒绝');
  }
  
  res.sendFile(filePath);
});
```

### Q: 如何处理大量截图文件？

**A:** 随着时间推移，截图可能会占用大量磁盘空间。考虑实现：

1. **自动清理策略**：定期删除旧截图
2. **存储限制**：设置最大存储空间限制
3. **用户管理界面**：允许用户查看和管理他们的截图

```javascript
// 自动清理示例
function setupAutoCleaning() {
  // 每天检查一次
  setInterval(async () => {
    const screenshotDir = getUserScreenshotsDir();
    const files = await fs.readdir(screenshotDir);
    
    // 获取文件信息
    const fileStats = await Promise.all(
      files.map(async file => {
        const filePath = path.join(screenshotDir, file);
        const stats = await fs.stat(filePath);
        return { path: filePath, stats };
      })
    );
    
    // 按修改时间排序
    fileStats.sort((a, b) => b.stats.mtimeMs - a.stats.mtimeMs);
    
    // 保留最新的 100 个文件，删除其余的
    if (fileStats.length > 100) {
      const filesToDelete = fileStats.slice(100);
      for (const file of filesToDelete) {
        await fs.unlink(file.path);
      }
      console.log(`已清理 ${filesToDelete.length} 个旧截图`);
    }
  }, 24 * 60 * 60 * 1000); // 24小时
}
```

### Q: 如何在不同设备间共享截图？

**A:** 如果您的应用需要在不同设备间共享截图，考虑以下方法：

1. **云存储集成**：将截图上传到云存储服务
2. **分享链接生成**：创建可分享的临时链接
3. **社交媒体集成**：直接分享到社交平台

```javascript
// 云存储上传示例
async function uploadToCloud(localPath) {
  const fileName = path.basename(localPath);
  const fileStream = fs.createReadStream(localPath);
  
  // 上传到云存储 (示例使用 AWS S3)
  const uploadParams = {
    Bucket: 'your-screenshots-bucket',
    Key: `user_${userId}/${fileName}`,
    Body: fileStream
  };
  
  try {
    const result = await s3Client.upload(uploadParams).promise();
    return {
      success: true,
      url: result.Location,
      key: result.Key
    };
  } catch (error) {
    console.error('上传失败:', error);
    return {
      success: false,
      error: error.message
    };
  }
}
```

## 完整迁移示例

以下是一个完整的迁移示例，展示了如何从旧版 API 迁移到新版 API：

### 旧版代码

```javascript
// screenshot-service.js (旧版)
class ScreenshotService {
  constructor(client) {
    this.client = client;
  }
  
  async takeScreenshot() {
    try {
      const response = await this.client.sendCommand('get_screenshot', {
        format: 'png',
        encode: 'base64'
      });
      
      if (response.status !== 'success') {
        throw new Error(`截图失败: ${response.message}`);
      }
      
      return {
        imageData: `data:image/png;base64,${response.data.base64}`,
        timestamp: new Date()
      };
    } catch (error) {
      console.error('截图过程中发生错误:', error);
      throw error;
    }
  }
  
  displayScreenshot(container, imageData) {
    const img = document.createElement('img');
    img.src = imageData;
    img.className = 'screenshot-image';
    
    container.innerHTML = '';
    container.appendChild(img);
  }
}
```

### 新版代码

```javascript
// screenshot-service.js (新版)
class ScreenshotService {
  constructor(client, fileServerUrl = null) {
    this.client = client;
    this.fileServerUrl = fileServerUrl;
  }
  
  async takeScreenshot(options = {}) {
    try {
      const response = await this.client.sendCommand('get_screenshot', {
        format: options.format || 'png',
        quality: options.quality || 75,
        save_dir: options.saveDir || 'screenshots/',
        filename: options.filename
      });
      
      if (response.status !== 'success') {
        throw new Error(`截图失败: ${response.message}`);
      }
      
      return {
        filePath: response.data.absolute_path,
        format: response.data.format,
        timestamp: new Date()
      };
    } catch (error) {
      console.error('截图过程中发生错误:', error);
      throw error;
    }
  }
  
  async displayScreenshot(container, screenshotInfo) {
    container.innerHTML = '';
    
    // 根据应用环境选择适当的显示方法
    if (this.isElectronApp()) {
      // Electron 应用可以直接使用文件路径
      this._displayLocalFile(container, screenshotInfo.filePath);
    } else if (this.fileServerUrl) {
      // Web 应用使用文件服务器
      this._displayRemoteFile(container, screenshotInfo);
    } else {
      // 回退方案：请求文件传输
      await this._displayViaTransfer(container, screenshotInfo.filePath);
    }
  }
  
  // 检测是否在 Electron 环境中
  isElectronApp() {
    return window && window.process && window.process.type === 'renderer';
  }
  
  // 本地文件显示 (Electron)
  _displayLocalFile(container, filePath) {
    const img = document.createElement('img');
    img.src = `file://${filePath}`;
    img.className = 'screenshot-image';
    container.appendChild(img);
  }
  
  // 远程文件显示 (Web 应用 + 文件服务器)
  _displayRemoteFile(container, screenshotInfo) {
    const fileName = screenshotInfo.filePath.split('/').pop();
    const fileUrl = `${this.fileServerUrl}/screenshots/${fileName}`;
    
    const img = document.createElement('img');
    img.src = fileUrl;
    img.className = 'screenshot-image';
    container.appendChild(img);
  }
  
  // 通过文件传输显示 (通用回退方案)
  async _displayViaTransfer(container, filePath) {
    try {
      const response = await fetch(`/api/transfer-file?path=${encodeURIComponent(filePath)}`);
      
      if (!response.ok) {
        throw new Error(`文件传输失败: ${response.statusText}`);
      }
      
      const blob = await response.blob();
      const img = document.createElement('img');
      img.src = URL.createObjectURL(blob);
      img.className = 'screenshot-image';
      img.onload = () => URL.revokeObjectURL(img.src); // 清理内存
      
      container.appendChild(img);
    } catch (error) {
      console.error('无法加载截图:', error);
      
      // 显示错误消息
      const errorMsg = document.createElement('div');
      errorMsg.className = 'screenshot-error';
      errorMsg.textContent = '无法加载截图';
      container.appendChild(errorMsg);
    }
  }
}
```

## 结论

迁移到新版 MCP 游戏截图 API 需要对客户端代码进行一些调整，但这些更改将带来显著的性能改进和更好的用户体验。通过遵循本指南中的最佳实践，您可以平稳地完成迁移，并充分利用新 API 的优势。

如果您在迁移过程中遇到任何问题，请参考完整的 API 文档或联系技术支持团队获取帮助。
