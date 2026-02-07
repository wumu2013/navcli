# NavCLI PRD

## 概述

**NavCLI** - 一个可交互、可探索的浏览器命令行工具，专为 AI Agent 设计。

通过命令行控制浏览器，让 Agent 能够：
- 像人类一样浏览网页
- 探索式获取页面信息
- 基于反馈持续与页面交互

## 核心价值

| 现有方案 | NavCLI |
|---------|--------|
| 直接调用 HTTP API | ✅ 支持 JS 渲染的 SPA |
| Playwright MCP (工具调用) | ✅ 会话状态持久化 |
| 无头浏览器脚本 | ✅ 交互式 CLI + 可探索 |

## 架构设计

### C/S 架构

```
┌──────────────────┐      HTTP       ┌──────────────────┐
│   CLI Client     │◄──────────────►│  Browser Server  │
│   (前台交互)      │   localhost:   │  (后台守护进程)   │
│   $ nav browse   │     8765       │   $ nav-server   │
└──────────────────┘                 └──────────────────┘
```

**为什么需要独立进程：**
- 维护 Playwright BrowserContext 状态
- 持久化 cookies、session
- 支持长时间运行的探索会话

### 单页面上下文

```
Browser Server
└── BrowserContext (单例)
    └── Page (当前页面)
        ├── url
        ├── title
        ├── html (DOM 快照)
        ├── variables (页面变量)
        └── cookies
```

**新窗口处理**：阻塞式，自动在当前页面跳转。后续可扩展 `new_tab` 命令。

## 状态管理

### StateSnapshot

```python
class StateSnapshot:
    url: str              # 当前 URL
    title: str            # 页面标题
    dom: DOMTree          # DOM 树
    variables: dict       # 页面变量
```

### DOMTree

```python
class DOMTree:
    version: int              # 增量版本号
    full: bool                # 是否完整 DOM
    html: str                 # 完整或增量 HTML
    changes: list[DOMChange]  # SPA 增量变化
    elements: list[Element]   # 可操作元素列表
```

### SPA Diff

- 首次加载返回完整 DOM
- 后续操作返回增量变化
- 变化超过 50% 时回退到完整 DOM

## 命令语法

### 导航命令

| 命令 | 别名 | 说明 |
|------|------|------|
| `goto <url>` | `g` | 导航到 URL |
| `back` | `b` | 后退 |
| `forward` | `f` | 前进 |
| `reload` | `r` | 刷新 |

### 交互命令

| 命令 | 别名 | 说明 |
|------|------|------|
| `click <selector> [text]` | `c` | 点击元素 |
| `dblclick <selector>` | - | 双击 |
| `rightclick <selector>` | - | 右键 |
| `type <selector> <text>` | `t` | 输入文本 |
| `clear <selector>` | - | 清空输入 |
| `upload <selector> <path>` | - | 上传文件 |

### 滚动命令

| 命令 | 说明 |
|------|------|
| `scroll top\|bottom\|up\|down` | 滚动方向 |
| `scroll <selector> into view` | 滚动到元素 |
| `scroll x=<num> y=<num>` | 滚动坐标 |

### 信息命令

| 命令 | 说明 |
|------|------|
| `elements` | 返回可操作元素列表 |
| `text` | 返回页面主体文本 |
| `text <selectors>` | 返回指定元素文本 |
| `text --structured` | 返回结构化文本 |
| `html` | 返回完整 DOM |
| `screenshot` | 返回 base64 图片 |
| `links` | 返回所有链接 |
| `forms` | 返回所有表单 |
| `url` | 返回当前 URL |
| `title` | 返回页面标题 |
| `state` | 返回完整状态 |

### 探索命令

| 命令 | 说明 |
|------|------|
| `find <text>` | 查找包含文本的元素 |
| `findall <text>` | 查找所有匹配元素 |
| `inspect <selector>` | 查看元素详情 |
| `wait <selector>` | 等待元素出现 |
| `wait <seconds>s` | 等待秒数 |

### 控制命令

| 命令 | 说明 |
|------|------|
| `quit` | 退出 CLI 并关闭服务器 |
| `exit` | 退出 CLI，保留服务器 |
| `clear` | 清空当前输入 |

### 变量命令

| 命令 | 说明 |
|------|------|
| `set <name> <value>` | 设置变量 |
| `$<name>` | 使用变量 |
| `get <expr>` | 获取 JS 表达式 |
| `evaluate <js>` | 执行 JS |

## Element 设计

### 返回规则

只返回"可操作 + 有意义"的元素：

| 元素类型 | 返回？ | 示例 |
|---------|--------|------|
| `button` | ✅ | `<button>提交</button>` |
| `a` (有 href) | ✅ | `<a href="/login">登录</a>` |
| `input` | ✅ | `<input type="email">` |
| `select` | ✅ | 下拉选择框 |
| `textarea` | ✅ | 文本域 |
| `h1`-`h3` | ✅ | 导航/标题 |
| `img` (有 alt) | ✅ | 有描述的图片 |
| `nav`, `menu` | ✅ | 导航区域 |
| `div` (无内容) | ❌ | 装饰容器 |
| `span` | ❌ | 文本片段 |
| `i`, `small` | ❌ | 样式/辅助文本 |

### Element 结构

```python
class Element:
    selector: str      # CSS 选择器
    tag: str           # 标签名
    text: str          # 元素文本
    clickable: bool    # 是否可点击
    input: bool        # 是否可输入
    type: str | None   # input 类型
```

## Feedback 设计

每个命令返回 `CommandResult`：

```python
class CommandResult:
    success: bool
    command: str
    state: StateSnapshot    # 页面状态
    feedback: Feedback      # 操作反馈
    error: str | None       # 错误信息

class Feedback:
    action: str             # 执行的动作
    result: str             # 执行结果
    changes: list           # DOM 变化
```

### Feedback 示例

```bash
> click .btn-primary

{
  "success": true,
  "command": "click",
  "feedback": {
    "action": "clicked .btn-primary",
    "result": "form submitted"
  },
  "state": {
    "url": "https://example.com/submit",
    "title": "Submit Result",
    "dom": {
      "version": 2,
      "full": false,
      "changes": [...],
      "elements": [...]
    },
    "variables": {}
  }
}
```

## Token 优化

| 场景 | 返回内容 |
|------|----------|
| 日常操作 | elements |
| 需要理解内容 | `text` |
| 需要精确定位 | `html` |

**分离设计**：
- `elements` - 每次返回（轻量）
- `text` - 按需获取
- `html` - 按需获取

## Agent 工作流示例

```bash
> g https://example.com
> elements           # 查看可操作元素
> text               # 理解页面内容
> c .btn-login       # 点击登录
> elements           # 查看登录表单
> t #email "test@example.com"
> t #password "123456"
> c button[type="submit"]
> text               # 确认登录结果
```

## 异步加载处理

### 等待机制

```bash
# 自动等待（默认）
> goto https://example.com          # 等待 network_idle
> click .btn                        # 等待网络空闲后点击

# 显式等待
> wait .dynamic-content             # 等待元素出现
> wait 3s                           # 等待 3 秒

# 强制执行（不等待）
> click .btn --force                # 立即点击
```

### 状态检测

```python
# 返回加载状态
{
  "state": {
    "url": "...",
    "loading": false,        # 是否仍在加载
    "network_idle": true,    # 网络是否空闲
    "dom_stable": true,      # DOM 是否稳定
    "pending_requests": 0    # 待处理请求数
  }
}
```

### 命令级别处理

| 命令 | 等待策略 |
|------|----------|
| `goto <url>` | 等待 `network_idle`（默认 3s） |
| `click <sel>` | 等待 `network_idle` |
| `click <sel> --force` | 不等待 |
| `goto <url> --wait-until=load` | 等待 load 事件 |
| `wait <selector>` | 等待元素出现（默认 10s 超时） |
| `wait <seconds>s` | 等待指定秒数 |

### SPA 场景

```python
# 返回增量 DOM 时附带加载状态
{
  "dom": {
    "version": 5,
    "full": false,
    "changes": [...],
    "async_loaded": true,    # 标记是异步加载的内容
    "loading": false         # 当前是否仍在加载
  }
}
```

### 轮询检测

```python
def wait_for_condition(condition: str, timeout: int = 10):
    """等待条件满足"""
    start = time.time()
    while time.time() - start < timeout:
        if condition == "selector_present":
            if page.query_selector(selector):
                return True
        elif condition == "text_present":
            if text in page.content():
                return True
        elif condition == "url_changed":
            if page.url != old_url:
                return True
        time.sleep(0.1)
    raise TimeoutError(f"Condition not met: {condition}")
```

### 策略选择

| 场景 | 推荐策略 |
|------|----------|
| 普通页面 | `wait_until=load` |
| SPA | `wait_until=domcontentloaded` + `network_idle` |
| 动态加载 | 显式 `wait <selector>` |
| 实时更新 | `poll <selector>` 轮询 |

**默认行为**：所有交互命令自动等待 `network_idle`（3秒无请求），超时后返回当前状态，让 agent 决定是否继续。

## API 协议

### HTTP Endpoints

```
# 导航
POST /cmd/goto?url=<url>
POST /cmd/back
POST /cmd/forward
POST /cmd/reload

# 交互
POST /cmd/click?selector=<sel>&force=<bool>
POST /cmd/type?selector=<sel>&text=<text>
POST /cmd/clear?selector=<sel>

# 等待
POST /wait?selector=<sel>&timeout=<seconds>
POST /wait/idle?timeout=<seconds>

# 信息
GET /elements
GET /text
GET /html
GET /screenshot
GET /state

# 控制
POST /shutdown
```

### 数据流向

```
┌─────────────────────────────────────────────────────────┐
│  Agent / User                                          │
│    │                                                    │
│    ▼ 输入命令                                           │
┌───────────────┐    HTTP Request    ┌──────────────────┐
│   CLI Client  │ ──────────────────► │  Browser Server  │
│  (cmd2 REPL)  │                     │  (Playwright)    │
└───────────────┘                     └──────────────────┘
       │                                      │
       │ 解析命令                             │ 执行操作
       │                                      │
       ▼ HTTP 调用                           ▼ 返回 State
┌───────────────┐    HTTP Response   ┌──────────────────┐
│   CLI Client  │ ◄────────────────── │  Browser Server  │
│               │                     │  (DOM + elements)│
└───────────────┘                     └──────────────────┘
       │                                      │
       │ 格式化输出                           │
       ▼
┌───────────────┐
│  Agent / User │
└───────────────┘
```

### 命令与 Endpoint 对应

| CLI 命令 | API Endpoint |
|----------|--------------|
| `goto <url>` | `POST /cmd/goto?url=<url>` |
| `click <sel>` | `POST /cmd/click?selector=<sel>` |
| `elements` | `GET /elements` |
| `text` | `GET /text` |
| `html` | `GET /html` |
| `wait <sel>` | `POST /wait?selector=<sel>` |
| `screenshot` | `GET /screenshot` |
| `state` | `GET /state` |

### 设计好处

| 好处 | 说明 |
|------|------|
| **解耦** | CLI 和 Server 可独立演进 |
| **可复用** | 其他客户端也能调用 HTTP 接口 |
| **测试友好** | 可以直接 curl 测试 API |
| **跨语言** | 任何语言都能调用 |


## 命令行接口

```bash
# 启动服务器
$ nav-server start     # 后台启动
$ nav-server stop      # 停止
$ nav-server status    # 状态

# 启动 CLI
$ nav browse           # 进入交互模式

# 组合命令
$ nav browse --exec "goto https://example.com && elements"
```

## 后续规划

### Phase 1 (MVP)
- [ ] HTTP Server + CLI Client
- [ ] 基础导航命令
- [ ] 基础交互命令
- [ ] 单页面上下文

### Phase 2
- [ ] SPA DOM diff
- [ ] 变量系统
- [ ] 新标签页支持
- [ ] 截图功能

### Phase 3
- [ ] 多上下文管理
- [ ] 浏览器配置（user-agent, viewport）
- [ ] 插件系统
