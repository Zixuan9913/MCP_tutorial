# Jenkins MCP 测试项目

## 目标
测试 Jenkins MCP Server 与 Claude Code 的集成，验证通过后接入公司正式 Jenkins 环境。

## 环境配置

- **Jenkins**: 本地 Docker 容器 (`jenkins/jenkins:lts`)
  - 地址: http://localhost:8080
  - 用户名: Zixuan
  - 容器名: jenkins
  - 启动命令: `docker run -d -p 8080:8080 -p 50000:50000 --name jenkins -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts`

- **MCP Server**: `mcp-jenkins`（开源社区项目）
  - 源码: https://github.com/lanbaoshen/mcp-jenkins.git
  - PyPI 包名: `mcp-jenkins`（通过 `uvx` 自动从 PyPI 下载运行，无需手动安装）
  - 配置文件: `.mcp.json`
  - uvx 路径: `/Users/zixuanliu/.local/bin/uvx`

- **`.mcp.json` 配置注意事项**:
  - 认证信息必须通过 CLI 参数传入（`--jenkins-url`, `--jenkins-username`, `--jenkins-password`），不支持环境变量
  - API Token 作为 `--jenkins-password` 的值传入

## 进度

- [x] 安装 uvx
- [x] Docker 启动本地 Jenkins
- [x] Jenkins 初始化（插件安装、管理员账号创建）
- [x] 生成 Jenkins API Token
- [x] 创建 `.mcp.json` 配置文件
- [x] 重启 Claude Code 加载 MCP
- [x] 测试 MCP 连接（成功）
- [x] 创建测试 Pipeline Job（`test-pipeline`），验证查看构建状态/日志/触发构建功能
- [ ] 接入公司 Jenkins 环境

## MCP 原理

```
你（自然语言）
  ↓
Claude Code（理解意图，选择工具）
  ↓ MCP 协议（JSON-RPC，通过 stdin/stdout）
mcp-jenkins（翻译层，Python 进程）
  ↓ HTTP REST API（requests 库）
Jenkins Server
```

- Claude Code 启动时读 `.mcp.json`，启动 `mcp-jenkins` 子进程
- `mcp-jenkins` 基于 FastMCP 框架，用 `@mcp.tool()` 装饰器注册工具
- 启动后通过 stdin/stdout 向 Claude Code 报告可用工具列表
- 调用时：Claude Code 发 JSON-RPC 请求 → mcp-jenkins 翻译成 Jenkins REST API 调用 → 返回结果
- **本质就是一个翻译器**，把 Claude Code 的工具调用翻译成 Jenkins HTTP 请求

## MCP 全部功能（27 个工具）

### Jobs/Items（7 个）
| 工具 | 用途 | 读/写 |
|------|------|-------|
| `get_all_items` | 获取所有 Job | 读 |
| `get_item` | 获取指定 Job 详情 | 读 |
| `get_item_config` | 获取 Job 的 XML 配置 | 读 |
| `set_item_config` | 修改 Job 配置 | 写 |
| `query_items` | 按正则过滤 Job（名称、类型、颜色） | 读 |
| `build_item` | 触发构建（支持带参数构建） | 写 |
| `get_item_parameters` | 获取 Job 的参数定义 | 读 |

### Builds（7 个）
| 工具 | 用途 | 读/写 |
|------|------|-------|
| `get_build` | 获取构建详情（不传 number 则取最近一次） | 读 |
| `get_running_builds` | 获取所有正在运行的构建 | 读 |
| `get_build_console_output` | 获取构建日志（支持正则过滤 + offset/limit 分页） | 读 |
| `get_build_test_report` | 获取测试报告 | 读 |
| `get_build_parameters` | 获取构建使用的参数值 | 读 |
| `get_build_scripts` | 获取构建使用的脚本（replay） | 读 |
| `stop_build` | 停止正在运行的构建 | 写 |

### Nodes（4 个）
| 工具 | 用途 | 读/写 |
|------|------|-------|
| `get_all_nodes` | 获取所有构建节点 | 读 |
| `get_node` | 获取指定节点详情（含 executor 状态） | 读 |
| `get_node_config` | 获取节点 XML 配置 | 读 |
| `set_node_config` | 修改节点配置 | 写 |

### Queue（3 个）
| 工具 | 用途 | 读/写 |
|------|------|-------|
| `get_all_queue_items` | 获取排队中的构建 | 读 |
| `get_queue_item` | 获取指定排队项详情 | 读 |
| `cancel_queue_item` | 取消排队中的构建 | 写 |

### Views（2 个）
| 工具 | 用途 | 读/写 |
|------|------|-------|
| `get_all_views` | 获取所有视图 | 读 |
| `get_view` | 获取指定视图（支持嵌套路径如 `frontend/nightly`） | 读 |

### 重要 CLI 参数
- `--read-only`: 只暴露读取工具，禁止写操作（适合生产环境初期接入）
- `--jenkins-verify-ssl`: 可关闭 SSL 验证
- `--jenkins-timeout`: 请求超时时间（默认 5s）
- `--transport`: 通信方式 stdio / sse / streamable-http

## 公司实用 Workflow

### 1. 构建失败排查（最高频）
> "test-service 最近一次构建挂了，帮我看看什么原因"

流程：`get_build` 发现失败 → `get_build_console_output` 用正则过滤 `ERROR|Exception` → 定位问题给出修复建议。省去翻几百行日志的时间。

### 2. 部署状态巡检
> "帮我看看 production 相关的 Job 最近状态怎么样"

流程：`query_items` 正则过滤 `prod|production` → 批量 `get_build` 检查结果 → 汇总成功/失败/排队状态。适合每天早上或发版前快速了解全局。

### 3. 触发构建 + 等结果
> "帮我 build 一下 deploy-staging，带上 branch=feature-xyz 参数"

流程：`build_item` 触发 → `get_build` 轮询状态 → 成功/失败后自动拉日志汇报。省去切浏览器填参数的过程。

### 4. 批量修改 Job 配置
> "把所有名字带 legacy 的 Job 配置里的 JDK 版本从 11 改成 17"

流程：`query_items` 过滤 → 批量 `get_item_config` 拿 XML → 修改 → `set_item_config` 写回。手动改几十个要半天，这里几分钟。

### 5. 节点健康检查
> "看看哪些 build agent 离线了"

流程：`get_all_nodes` → 过滤 `offline=true` → 汇报哪些节点挂了、上面还有没有跑着的任务。

## 公司接入建议

| 阶段 | 做法 |
|------|------|
| 第一步 | `--read-only` 接入，只看不动，让团队先体验查日志、查状态 |
| 第二步 | 开放 `build_item`，允许触发构建 |
| 第三步 | 按需开放配置修改（`set_item_config`） |

### 部署方式选择
- **个人/小团队**: 直接 `uvx mcp-jenkins`，每人本地运行，简单够用
- **需要统一管控时**: 拉源码部署为 HTTP 服务（`--transport streamable-http`），Token 集中管理，用户不直接持有
