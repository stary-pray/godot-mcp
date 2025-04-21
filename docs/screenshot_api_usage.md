# MCP 游戏截图功能使用指南

本文档提供了关于如何使用 MCP 游戏截图功能的详细指南，包括实际应用场景、最佳实践和常见问题解答。

## 实际应用场景

### 1. 游戏测试与调试

截图功能可以帮助开发者和测试人员捕获游戏中的特定状态，用于：
- 记录游戏 bug 或异常状态
- 对比不同版本的视觉变化
- 创建测试报告的视觉证据

```typescript
// 在遇到 bug 时自动截图
function onBugDetected(bugInfo: BugInfo) {
  const timestamp = new Date().toISOString();
  client.sendCommand('get_screenshot', {
    filename: `bug_${bugInfo.id}_${timestamp}`,
    save_dir: 'debug/bugs/'
  }).then(response => {
    bugInfo.screenshotPath = response.data.absolute_path;
    submitBugReport(bugInfo);
  });
}
```

### 2. 游戏内截图功能

为玩家提供游戏内截图功能，允许他们：
- 保存游戏中的精彩瞬间
- 分享游戏成就或进度
- 创建自定义游戏内容集

```typescript
// 游戏内截图按钮处理函数
function onScreenshotButtonPressed() {
  showScreenshotEffect(); // 显示截图动画效果
  
  client.sendCommand('get_screenshot', {
    format: userPreferences.screenshotFormat,
    quality: userPreferences.jpegQuality,
    save_dir: 'user_screenshots/'
  }).then(response => {
    showNotification(`截图已保存到: ${getRelativePath(response.data.absolute_path)}`);
    updateScreenshotGallery(response.data.absolute_path);
  });
}
```

### 3. 自动化测试与 CI/CD

在自动化测试流程中使用截图功能：
- 视觉回归测试
- 自动生成游戏文档
- 持续集成过程中的视觉验证

```typescript
// 自动化测试中的视觉比较
async function visualRegressionTest(testCase: TestCase) {
  await navigateToScene(testCase.sceneName);
  
  const response = await client.sendCommand('get_screenshot', {
    filename: `test_${testCase.id}`,
    save_dir: 'test_results/'
  });
  
  const currentScreenshot = response.data.absolute_path;
  const referenceScreenshot = `./references/${testCase.id}.png`;
  
  const difference = await compareImages(currentScreenshot, referenceScreenshot);
  expect(difference.percentage).toBeLessThan(testCase.threshold);
}
```

## 最佳实践

### 性能优化

1. **合理控制截图频率**
   - 避免在短时间内频繁截图，特别是在性能敏感的场景
   - 考虑使用节流（throttling）技术限制截图频率

2. **选择合适的图像格式**
   - 对于需要高质量图像的场景，使用 PNG 格式
   - 对于一般用途，使用 JPEG 格式并调整适当的质量参数（推荐 75-90）

3. **管理存储空间**
   - 实现自动清理机制，删除过期或不需要的截图
   - 监控截图目录大小，避免占用过多磁盘空间

```typescript
// 清理过期截图
async function cleanupOldScreenshots() {
  const screenshotDir = getUserScreenshotsDir();
  const files = await fs.readdir(screenshotDir);
  
  const now = Date.now();
  const oneMonthAgo = now - (30 * 24 * 60 * 60 * 1000);
  
  for (const file of files) {
    const filePath = path.join(screenshotDir, file);
    const stats = await fs.stat(filePath);
    
    if (stats.mtimeMs < oneMonthAgo) {
      await fs.unlink(filePath);
    }
  }
}
```

### 错误处理

1. **实现健壮的错误处理**
   - 处理网络错误、超时和服务器错误
   - 提供用户友好的错误消息
   - 实现重试机制

```typescript
async function takeScreenshotWithRetry(params, maxRetries = 3) {
  let retries = 0;
  
  while (retries < maxRetries) {
    try {
      return await client.sendCommand('get_screenshot', params);
    } catch (error) {
      retries++;
      
      if (retries >= maxRetries) {
        console.error('截图失败，已达到最大重试次数:', error);
        throw new Error('无法获取游戏截图，请稍后再试');
      }
      
      console.warn(`截图失败，正在重试 (${retries}/${maxRetries})...`);
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

### 用户体验

1. **提供视觉反馈**
   - 在截图过程中显示加载指示器或动画效果
   - 截图完成后提供成功通知
   - 允许用户预览和管理截图

2. **自定义选项**
   - 允许用户配置截图格式、质量和保存位置
   - 提供截图命名选项
   - 支持添加水印或自定义覆盖层

```typescript
// 截图设置界面
function ScreenshotSettingsUI() {
  return (
    <SettingsPanel title="截图设置">
      <DropdownSetting
        label="图像格式"
        options={[
          { value: 'png', label: 'PNG (高质量)' },
          { value: 'jpg', label: 'JPEG (小文件)' }
        ]}
        value={settings.format}
        onChange={value => updateSettings({ format: value })}
      />
      
      {settings.format === 'jpg' && (
        <SliderSetting
          label="JPEG 质量"
          min={50}
          max={100}
          value={settings.quality}
          onChange={value => updateSettings({ quality: value })}
        />
      )}
      
      <TextSetting
        label="保存目录"
        value={settings.saveDir}
        onChange={value => updateSettings({ saveDir: value })}
      />
      
      <ToggleSetting
        label="添加时间戳到文件名"
        value={settings.addTimestamp}
        onChange={value => updateSettings({ addTimestamp: value })}
      />
    </SettingsPanel>
  );
}
```

## 常见问题解答

### Q: 为什么不再支持 base64 编码的图像数据？

**A:** 移除 base64 编码支持是为了提高性能和减少内存使用。通过 WebSocket 传输大量 base64 编码数据可能导致内存问题，特别是对于高分辨率截图。文件系统方法更高效，并且避免了这些问题。

### Q: 如何在不同平台上处理文件路径？

**A:** 返回的 `absolute_path` 是服务器端的路径，如果客户端在不同的机器上，您需要：
1. 实现一个文件服务器来提供这些截图
2. 使用相对路径并在客户端重建完整路径
3. 实现文件传输机制将截图从服务器传输到客户端

### Q: 截图操作会暂停游戏吗？

**A:** 截图操作是异步的，不会暂停游戏。然而，在截图瞬间可能会有轻微的性能影响，特别是在资源受限的设备上。

### Q: 如何处理高 DPI 显示器上的截图？

**A:** 截图会捕获游戏的实际渲染分辨率，包括任何 DPI 缩放。如果需要特定分辨率的截图，您需要在游戏设置中调整渲染分辨率。

### Q: 可以截取特定游戏区域而不是整个屏幕吗？

**A:** 当前版本只支持捕获整个游戏窗口。如果需要特定区域，您需要在获取完整截图后进行裁剪处理。

## 示例代码库

以下是一个完整的 TypeScript 工具类，封装了截图功能：

```typescript
// screenshot-manager.ts
import { MCPClient } from '@yuki/mcp-client';

export interface ScreenshotOptions {
  format?: 'png' | 'jpg';
  quality?: number;
  saveDir?: string;
  filename?: string;
}

export interface ScreenshotResult {
  path: string;
  format: string;
  timestamp: Date;
}

export class ScreenshotManager {
  private client: MCPClient;
  private recentScreenshots: ScreenshotResult[] = [];
  
  constructor(host: string = 'localhost', port: number = 9080) {
    this.client = new MCPClient(host, port);
  }
  
  async takeScreenshot(options: ScreenshotOptions = {}): Promise<ScreenshotResult> {
    try {
      const response = await this.client.sendCommand('get_screenshot', {
        format: options.format || 'png',
        quality: options.format === 'jpg' ? (options.quality || 75) : undefined,
        save_dir: options.saveDir || 'screenshots/',
        filename: options.filename
      });
      
      if (response.status !== 'success') {
        throw new Error(`截图失败: ${response.message}`);
      }
      
      const result: ScreenshotResult = {
        path: response.data.absolute_path,
        format: response.data.format,
        timestamp: new Date()
      };
      
      this.recentScreenshots.unshift(result);
      if (this.recentScreenshots.length > 20) {
        this.recentScreenshots.pop();
      }
      
      return result;
    } catch (error) {
      console.error('截图过程中发生错误:', error);
      throw error;
    }
  }
  
  getRecentScreenshots(): ScreenshotResult[] {
    return [...this.recentScreenshots];
  }
  
  clearRecentScreenshots(): void {
    this.recentScreenshots = [];
  }
}
```

使用示例：

```typescript
// 使用截图管理器
import { ScreenshotManager } from './screenshot-manager';

const screenshotManager = new ScreenshotManager();

// 简单用法
async function quickScreenshot() {
  try {
    const screenshot = await screenshotManager.takeScreenshot();
    console.log(`截图已保存到: ${screenshot.path}`);
  } catch (error) {
    console.error('截图失败:', error);
  }
}

// 高级用法
async function customScreenshot() {
  try {
    const screenshot = await screenshotManager.takeScreenshot({
      format: 'jpg',
      quality: 90,
      saveDir: 'custom_screenshots/',
      filename: `gameplay_${Date.now()}`
    });
    
    displayScreenshotPreview(screenshot.path);
    addToScreenshotGallery(screenshot);
  } catch (error) {
    showErrorNotification('无法获取游戏截图，请稍后再试');
  }
}
```
