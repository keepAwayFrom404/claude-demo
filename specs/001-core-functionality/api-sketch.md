# API Sketch - Core Functionality

> **Version:** 1.0
> **Created:** 2026-02-03
> **Status:** Draft

本文档描述 `internal/` 下各包对外暴露的主要接口签名，作为后续开发的参考。

---

## 目录

- [1. 包结构概览](#1-包结构概览)
- [2. `internal/parser` - URL 解析](#2-internalparser---url-解析)
- [3. `internal/github` - GitHub API 客户端](#3-internalgithub---github-api-客户端)
- [4. `internal/converter` - Markdown 转换器](#4-internalconverter---markdown-转换器)
- [5. `internal/cli` - CLI 配置与入口](#5-internalcli---cli-配置与入口)
- [6. 数据流图](#6-数据流图)

---

## 1. 包结构概览

```
internal/
├── parser/      # URL 解析与类型识别
├── github/      # GitHub API 交互
├── converter/   # 数据转换为 Markdown
└── cli/         # CLI 配置与程序入口
```

**包依赖关系**（自底向上）：

```
parser
   ↓
github
   ↓
converter
   ↓
cli
```

---

## 2. `internal/parser` - URL 解析

### 2.1 核心类型

```go
// ResourceType 表示 GitHub 资源的类型
type ResourceType string

const (
    ResourceIssue       ResourceType = "issue"
    ResourcePullRequest ResourceType = "pull_request"
    ResourceDiscussion  ResourceType = "discussion"
)

// ParsedURL 表示解析后的 URL 信息
type ParsedURL struct {
    Type     ResourceType   // 资源类型
    Owner    string         // 仓库所有者
    Repo     string         // 仓库名称
    Number   int            // Issue/PR/Discussion 编号
    Original string         // 原始 URL
}
```

### 2.2 导出函数

```go
// ParseURL 解析 GitHub URL 并返回资源信息
//
// 支持的 URL 格式:
//   - https://github.com/{owner}/{repo}/issues/{number}
//   - https://github.com/{owner}/{repo}/pull/{number}
//   - https://github.com/{owner}/{repo}/discussions/{number}
//
// 返回错误:
//   - ErrInvalidURLFormat: URL 格式无效
//   - ErrUnsupportedResourceType: 不支持的资源类型
func ParseURL(url string) (*ParsedURL, error)
```

### 2.3 错误定义

```go
var (
    ErrInvalidURLFormat       = errors.New("invalid GitHub URL format")
    ErrUnsupportedResourceType = errors.New("unsupported resource type")
)
```

---

## 3. `internal/github` - GitHub API 客户端

### 3.1 核心类型

```go
// Client 表示 GitHub API 客户端
type Client struct {
    token      string           // 可选的 Personal Access Token
    httpClient *http.Client     // HTTP 客户端
    baseURL    string           // API 基础 URL
}

// Resource 表示从 GitHub 获取的统一资源数据
type Resource struct {
    Type        ResourceType
    Title       string
    Author      string
    CreatedAt   time.Time
    State       string  // "open", "closed", "merged"
    Body        string
    Comments    []Comment
    URL         string
    Number      int
}

// Comment 表示一条评论
type Comment struct {
    Author       string
    CreatedAt    time.Time
    Body         string
    IsAnswer     bool           // 仅用于 Discussion
    Reactions    *Reactions     // 可选
}

// Reactions 表示反应统计
type Reactions struct {
    ThumbsUp   int
    ThumbsDown int
    Laugh      int
    Hooray     int
    Confused   int
    Heart      int
    Rocket     int
    Eyes       int
}
```

### 3.2 导出函数

```go
// NewClient 创建一个新的 GitHub API 客户端
//
// token: 可选的 Personal Access Token，从环境变量 GITHUB_TOKEN 读取
// 如果 token 为空字符串，则不进行认证
func NewClient(token string) *Client

// FetchResource 根据 URL 信息获取 GitHub 资源
//
// 该方法会：
//   1. 根据 parsedURL.Type 调用对应的 API
//   2. 自动处理分页，获取所有评论
//   3. 返回统一的 Resource 结构
//
// 返回错误:
//   - 包装了 http 请求错误
//   - 包装了 JSON 解析错误
//   - GitHub API 返回的错误（404, 403, 429 等）
func (c *Client) FetchResource(parsedURL *parser.ParsedURL) (*Resource, error)
```

### 3.3 内部方法（不导出）

```go
// fetchIssue 获取 Issue 数据
func (c *Client) fetchIssue(owner, repo string, number int) (*Resource, error)

// fetchPullRequest 获取 PR 数据（包括 Review Comments）
func (c *Client) fetchPullRequest(owner, repo string, number int) (*Resource, error)

// fetchDiscussion 获取 Discussion 数据
func (c *Client) fetchDiscussion(owner, repo string, number int) (*Resource, error)

// fetchAllComments 获取所有评论（处理分页）
func (c *Client) fetchAllComments(apiURL string) ([]Comment, error)
```

---

## 4. `internal/converter` - Markdown 转换器

### 4.1 核心类型

```go
// Options 转换选项
type Options struct {
    EnableReactions    bool  // 是否包含 reactions
    EnableUserLinks    bool  // 是否渲染用户链接
}

// Converter 将 GitHub 资源转换为 Markdown
type Converter struct {
    options Options
}
```

### 4.2 导出函数

```go
// NewConverter 创建一个新的转换器
func NewConverter(opts Options) *Converter

// Convert 将资源转换为 Markdown 格式
//
// 返回的 Markdown 包含:
//   1. YAML frontmatter
//   2. 标题和元数据
//   3. 正文内容
//   4. 所有评论（按时间正序）
//
// 对于 PR，会合并普通 Comments 和 Review Comments
// 对于 Discussion，会标记 Accepted Answer
func (c *Converter) Convert(res *github.Resource) string

// ConvertToWriter 将资源直接写入 io.Writer
// 支持流式写入，适合大文件输出
func (c *Converter) ConvertToWriter(res *github.Resource, w io.Writer) error
```

### 4.3 内部方法（不导出）

```go
// renderFrontmatter 渲染 YAML frontmatter
func (c *Converter) renderFrontmatter(res *github.Resource) string

// renderHeader 渲染标题和元数据
func (c *Converter) renderHeader(res *github.Resource) string

// renderComments 渲染所有评论
func (c *Converter) renderComments(comments []github.Comment) string

// renderReactions 渲染 reactions 统计
func (c *Converter) renderReactions(r *github.Reactions) string

// formatUser 渲染用户名（根据选项决定是否添加链接）
func (c *Converter) formatUser(username string) string

// formatTime 格式化时间为本地化格式
func (c *Converter) formatTime(t time.Time) string
```

---

## 5. `internal/cli` - CLI 配置与入口

### 5.1 核心类型

```go
// Config 表示从 CLI 解析的配置
type Config struct {
    URL              string   // GitHub URL（位置参数）
    OutputFile       string   // 输出文件路径（可选位置参数）
    EnableReactions  bool     // -enable-reactions
    EnableUserLinks  bool     // -enable-user-links
}

// Runner 执行 CLI 程序
type Runner struct {
    config     *Config
    githubCli  *github.Client
    converter  *converter.Converter
    stdout     io.Writer
    stderr     io.Writer
}
```

### 5.2 导出函数

```go
// ParseConfig 从命令行参数解析配置
// 使用标准库 flag 包
func ParseConfig(args []string) (*Config, error)

// NewRunner 创建一个新的 Runner
func NewRunner(cfg *Config) *Runner

// Run 执行主程序逻辑
//
// 返回错误:
//   - URL 解析错误
//   - GitHub API 错误
//   - 文件写入错误
func (r *Runner) Run() error
```

### 5.3 主函数（在 cmd/issue2md 中）

```go
// main 函数逻辑:
// 1. 读取 GITHUB_TOKEN 环境变量
// 2. 解析命令行参数
// 3. 创建 Runner 并执行
// 4. 处理错误，输出到 stderr，设置正确的退出码
```

---

## 6. 数据流图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLI Entry                                  │
│                         (cmd/issue2md/main.go)                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           cli.ParseConfig()                             │
│  解析命令行参数和环境变量，返回 *cli.Config                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           parser.ParseURL()                             │
│  解析 URL，返回 *parser.ParsedURL                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       github.Client.FetchResource()                     │
│  调用 GitHub API，返回 *github.Resource                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       converter.Convert()                               │
│  将资源转换为 Markdown 字符串                                            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Output (stdout/file)                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 错误处理约定

所有包都应遵循以下错误处理约定：

```go
// 包装错误时使用 fmt.Errorf
return fmt.Errorf("failed to fetch issue: %w", err)

// 定义包级别的错误变量
var (
    ErrInvalidURL = errors.New("invalid URL format")
    ErrNotFound   = errors.New("resource not found")
)

// 使用 errors.Is 进行错误判断
if errors.Is(err, parser.ErrInvalidURL) {
    // 处理特定错误
}
```

---

## 8. 测试策略

### 8.1 单元测试

- **parser**: 表格驱动测试，覆盖各种 URL 格式
- **converter**: 表格驱动测试，测试各种 Markdown 输出场景
- **github**: 使用 httptest 模拟 API 响应

### 8.2 集成测试

- 使用真实的 GitHub API 测试完整的转换流程
- 测试公开仓库的 Issue/PR/Discussion

### 8.3 端到端测试

- 测试 CLI 完整流程
- 验证输出文件格式

---

## 9. 实现顺序

1. **Phase 1**: `parser` 包（无外部依赖，易测试）
2. **Phase 2**: `github` 包的核心功能（Issue 支持）
3. **Phase 3**: `converter` 包（基础 Markdown 渲染）
4. **Phase 4**: `cli` 包和 `cmd/issue2md`（完整的 CLI）
5. **Phase 5**: 扩展 `github` 包（PR、Discussion 支持）
6. **Phase 6**: 扩展 `converter` 包（Reactions、User Links）

---

## 附录：文件清单

```
internal/
├── parser/
│   ├── parser.go          # ParseURL 函数，类型定义
│   └── parser_test.go     # 表格驱动测试
├── github/
│   ├── client.go          # Client 类型，FetchResource 方法
│   ├── types.go           # Resource, Comment 等类型定义
│   ├── issue.go           # Issue 获取逻辑
│   ├── pull_request.go    # PR 获取逻辑
│   ├── discussion.go      # Discussion 获取逻辑
│   └── client_test.go     # 使用 httptest 的测试
├── converter/
│   ├── converter.go       # Convert 方法
│   ├── frontmatter.go     # frontmatter 渲染
│   ├── comments.go        # 评论渲染
│   ├── reactions.go       # reactions 渲染
│   └── converter_test.go  # 表格驱动测试
└── cli/
    ├── config.go          # Config 类型，ParseConfig 函数
    ├── runner.go          # Runner 类型，Run 方法
    └── main.go            // 实际在 cmd/issue2md/main.go

cmd/
├── issue2md/
│   └── main.go            # CLI 入口
└── issue2mdweb/
    └── main.go            # Web 入口（未来实现）
```
