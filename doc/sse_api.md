# SSE (Server-Sent Events) 接口文档

## 概述

TimeCapsule 使用 SSE (Server-Sent Events) 实现实时状态更新，前端可以通过 SSE 连接接收服务器端的实时事件推送。

## 连接端点

### SSE 连接
```
GET /api/v1/events
```

**请求头：**
- `X-Client-ID`: 可选，客户端唯一标识符（如果不提供，服务器会自动生成）

**响应头：**
- `Content-Type`: `text/event-stream`
- `Cache-Control`: `no-cache`
- `Connection`: `keep-alive`
- `Access-Control-Allow-Origin`: `*`

## 消息格式

所有 SSE 消息都遵循以下格式：

```json
{
  "type": "message_type",
  "data": {},
  "timestamp": 1234567890
}
```

## 消息类型

### 1. file_update
文件更新事件

**数据结构：**
```json
{
  "type": "file_update",
  "data": {
    "files": [...],
    "dirs": [...],
    "total": 100
  },
  "timestamp": 1234567890
}
```

**字段说明：**
- `files`: 文件列表
- `dirs`: 目录列表
- `total`: 总数

### 2. tag_update
标签更新事件

**数据结构：**
```json
{
  "type": "tag_update",
  "data": {
    "tags": [...]
  },
  "timestamp": 1234567890
}
```

**字段说明：**
- `tags`: 标签列表

### 3. project_update
项目更新事件

**数据结构：**
```json
{
  "type": "project_update",
  "data": {
    "project": {
      "name": "project_name",
      "path": "/path/to/project"
    }
  },
  "timestamp": 1234567890
}
```

**字段说明：**
- `project`: 项目信息

### 4. heartbeat
心跳事件（每 30 秒发送一次）

**数据结构：**
```json
{
  "type": "heartbeat",
  "data": {
    "status": {
      "busy": false,
      "status": "idle",
      "logInfo": "...",
      "curTask": "...",
      "curCmd": "..."
    }
  },
  "timestamp": 1234567890
}
```

**字段说明：**
- `status`: 系统状态信息

### 5. log_update
日志更新事件

**数据结构：**
```json
{
  "type": "log_update",
  "data": {
    "log": "log message"
  },
  "timestamp": 1234567890
}
```

**字段说明：**
- `log`: 日志内容

## 前端使用示例

### 基本连接

```typescript
const eventSource = new EventSource('/api/v1/events');

eventSource.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
  
  switch (message.type) {
    case 'file_update':
      handleFileUpdate(message.data);
      break;
    case 'tag_update':
      handleTagUpdate(message.data);
      break;
    case 'project_update':
      handleProjectUpdate(message.data);
      break;
    case 'heartbeat':
      handleHeartbeat(message.data);
      break;
    case 'log_update':
      handleLogUpdate(message.data);
      break;
  }
};

eventSource.onerror = (error) => {
  console.error('SSE error:', error);
  eventSource.close();
};
```

### 带重连的连接

```typescript
class SSEService {
  private eventSource: EventSource | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private reconnectDelay = 1000;
  private isConnected = false;
  private url: string = '';

  connect(url: string) {
    if (this.isConnected) {
      console.warn('[SSE] Already connected');
      return;
    }

    this.url = url;
    console.log(`[SSE] Connecting to ${url}`);

    try {
      this.eventSource = new EventSource(url);
      this.setupEventHandlers();
    } catch (error) {
      console.error('[SSE] Connection failed:', error);
      this.reconnect();
    }
  }

  private setupEventHandlers() {
    if (!this.eventSource) return;

    this.eventSource.onopen = () => {
      console.log('[SSE] Connected');
      this.isConnected = true;
      this.reconnectAttempts = 0;
      this.reconnectDelay = 1000;
    };

    this.eventSource.onmessage = (event) => {
      this.handleMessage(event.data);
    };

    this.eventSource.onerror = (error) => {
      console.error('[SSE] Error:', error);
      this.isConnected = false;
      this.reconnect();
    };
  }

  private handleMessage(data: string) {
    try {
      const message = JSON.parse(data);
      console.log('[SSE] Received:', message);
      this.processMessage(message);
    } catch (error) {
      console.error('[SSE] Failed to parse message:', error);
    }
  }

  private processMessage(message: any) {
    switch (message.type) {
      case 'file_update':
        this.handleFileUpdate(message.data);
        break;
      case 'tag_update':
        this.handleTagUpdate(message.data);
        break;
      case 'project_update':
        this.handleProjectUpdate(message.data);
        break;
      case 'heartbeat':
        this.handleHeartbeat(message.data);
        break;
      case 'log_update':
        this.handleLogUpdate(message.data);
        break;
      default:
        console.warn('[SSE] Unknown message type:', message.type);
    }
  }

  private handleFileUpdate(data: any) {
    // 处理文件更新
    console.log('File update:', data);
  }

  private handleTagUpdate(data: any) {
    // 处理标签更新
    console.log('Tag update:', data);
  }

  private handleProjectUpdate(data: any) {
    // 处理项目更新
    console.log('Project update:', data);
  }

  private handleHeartbeat(data: any) {
    // 处理心跳
    console.log('Heartbeat:', data);
  }

  private handleLogUpdate(data: any) {
    // 处理日志更新
    console.log('Log update:', data);
  }

  private reconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('[SSE] Max reconnection attempts reached');
      return;
    }

    this.reconnectAttempts++;
    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
    
    console.log(`[SSE] Reconnecting... (${this.reconnectAttempts}/${this.maxReconnectAttempts}) in ${delay}ms`);

    setTimeout(() => {
      this.disconnect();
      this.connect(this.url);
    }, delay);
  }

  disconnect() {
    if (this.eventSource) {
      console.log('[SSE] Disconnecting');
      this.eventSource.close();
      this.eventSource = null;
      this.isConnected = false;
    }
  }

  getStatus() {
    return {
      isConnected: this.isConnected,
      reconnectAttempts: this.reconnectAttempts
    };
  }
}

// 使用示例
const sseService = new SSEService();
sseService.connect('/api/v1/events');
```

## 后端 API

### 广播函数

#### BroadcastFileUpdate
广播文件更新事件

```go
api.BroadcastFileUpdate(files, dirs, total)
```

**参数：**
- `files`: 文件列表
- `dirs`: 目录列表
- `total`: 总数

#### BroadcastTagUpdate
广播标签更新事件

```go
api.BroadcastTagUpdate(tags)
```

**参数：**
- `tags`: 标签列表

#### BroadcastProjectUpdate
广播项目更新事件

```go
api.BroadcastProjectUpdate(project)
```

**参数：**
- `project`: 项目信息

#### BroadcastLogUpdate
广播日志更新事件

```go
api.BroadcastLogUpdate(log)
```

**参数：**
- `log`: 日志内容

#### broadcastHeartbeat
广播心跳事件

```go
api.broadcastHeartbeat()
```

## 使用场景

### 1. 文件扫描进度更新

```go
// 在文件扫描过程中
api.BroadcastFileUpdate(scannedFiles, scannedDirs, totalFiles)
```

### 2. 标签管理

```go
// 添加或删除标签后
api.BroadcastTagUpdate(allTags)
```

### 3. 项目切换

```go
// 切换项目后
api.BroadcastProjectUpdate(currentProject)
```

### 4. 日志实时显示

```go
// 添加日志时
api.BroadcastLogUpdate(logMessage)
```

### 5. 系统状态监控

```go
// 定期发送心跳
api.broadcastHeartbeat()
```

## 注意事项

1. **连接稳定性**
   - SSE 连接可能会断开，需要实现自动重连机制
   - 建议使用指数退避算法进行重连

2. **消息顺序**
   - SSE 保证消息按顺序发送
   - 但网络延迟可能导致消息乱序到达

3. **性能考虑**
   - 避免频繁发送大量数据
   - 合理设置消息发送频率

4. **错误处理**
   - 监听 `onerror` 事件
   - 实现适当的错误恢复机制

5. **客户端 ID**
   - 建议使用唯一的客户端 ID
   - 便于追踪和管理连接

## 测试

### 测试 SSE 连接

```bash
# 使用 curl 测试
curl -N -H "Accept: text/event-stream" http://localhost:38080/api/v1/events
```

### 测试消息广播

```go
// 在代码中测试广播
api.BroadcastFileUpdate([]string{"test.jpg"}, []string{"test"}, 1)
api.BroadcastTagUpdate([]string{"tag1", "tag2"})
api.BroadcastProjectUpdate(map[string]interface{}{"name": "test"})
api.BroadcastLogUpdate("Test log message")
```

## 故障排除

### 问题 1: 连接断开
**解决方案：**
- 检查网络连接
- 实现自动重连机制
- 检查服务器日志

### 问题 2: 消息未收到
**解决方案：**
- 检查消息格式是否正确
- 检查客户端事件监听器
- 查看浏览器控制台错误

### 问题 3: 性能问题
**解决方案：**
- 减少消息发送频率
- 优化消息大小
- 使用消息队列

## 相关文档

- [前端 SSE 实现](../frontend/sse-service.md)
- [API 文档](api.md)
- [实时通信最佳实践](realtime-best-practices.md)
