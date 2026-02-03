# 技术实现方案 - Spec 001

**文档版本:** 1.0
**状态:** 草稿
**创建日期:** 2026-02-03
**作者:** issue2md 项目架构组

---

## 目录

1. [技术上下文总结](#1-技术上下文总结)
2. [合宪性审查](#2-合宪性审查)
3. [项目结构细化](#3-项目结构细化)
4. [核心数据结构](#4-核心数据结构)
5. [接口设计](#5-接口设计)
6. [实施阶段](#6-实施阶段)
7. [测试策略](#7-测试策略)
8. [依赖管理](#8-依赖管理)

---

## 1. 技术上下文总结

### 1.1 技术栈

| 组件 | 技术选型 | 选择理由 |
|------|----------|----------|
| **编程语言** | Go >= 1.21.0 | 类型安全、标准库优秀、部署简单 |
| **Web 框架** | `net/http` (仅标准库) | 符合宪法"简单性优先"原则 |
| **GitHub API 客户端** | `google/go-github` v69+ | 官方客户端，久经考验 |
| **GraphQL 客户端** | `google/go-github` + GraphQL 支持 | 统一的 REST/GraphQL 访问 |
| **Markdown 处理** | 标准库 `text/template` | 无需第三方 markdown 库 |
| **数据存储** | 无 (仅 API 实时获取) | 无状态设计 |

### 1.2 外部依赖

```go
require (
    github.com/google/go-github v69.0.0
    golang.org/x/oauth2 v0.20.0  // GitHub 认证支持
)
```

**设计理由:** 最小化依赖。仅使用官方 GitHub 客户端和 OAuth2 支持。

### 1.3 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI 入口点                              │
│                      (cmd/issue2md/main.go)                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    配置层 (internal/cli)                        │
│  • 解析命令行参数                                                │
│  • 从环境变量读取 GITHUB_TOKEN                                   │
│  • 验证输入                                                      │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   URL 解析层 (internal/parser)                  │
│  • 解析 GitHub URL                                              │
│  • 识别资源类型 (Issue/PR/Discussion)                           │
│  • 提取 owner, repo, number                                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                GitHub API 客户端层 (internal/github)            │
│  • REST API 调用 (Issues, PullRequests)                         │
│  • GraphQL 调用 (Discussions)                                   │
│  • 分页处理                                                      │
│  • 认证 (通过 token)                                            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              数据转换层 (internal/converter)                    │
│  • 生成 YAML frontmatter                                        │
│  • 渲染带 reactions 的评论                                       │
│  • 格式化时间戳和用户链接                                        │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       输出层                                    │
│  • stdout (默认)                                                │
│  • 文件 (通过 -o 参数)                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 合宪性审查

### 2.1 第一条：简单性原则 (Simplicity First)

| 原则 | 合规性检查 | 证据 |
|------|-----------|------|
| **1.1 YAGNI** | ✅ 通过 | 仅实现 spec.md 明确要求的功能。无过度抽象。 |
| **1.2 标准库优先** | ✅ 通过 | Web 服务器使用 `net/http`。CLI 使用 `flag` 包。不使用 Gin/Echo。 |
| **1.3 反过度工程** | ✅ 通过 | 简单函数优于复杂接口。直接类型优于泛型。 |

**结论:** 完全合规

### 2.2 第二条：测试先行铁律 (Test-First Imperative)

| 原则 | 合规性检查 | 证据 |
|------|-----------|------|
| **2.1 TDD 循环** | ✅ 通过 | 每个组件实现前先编写失败的测试。 |
| **2.2 表格驱动测试** | ✅ 通过 | Parser 测试使用表格驱动模式。参见 api-sketch.md。 |
| **2.3 拒绝 Mocks** | ⚠️ 有条件通过 | 集成测试使用真实 GitHub API (公开仓库)。单元测试使用 `httptest` 进行 API 模拟。 |

**结论:** 合规 (有说明例外情况)

### 2.3 第三条：明确性原则 (Clarity and Explicitness)

| 原则 | 合规性检查 | 证据 |
|------|-----------|------|
| **3.1 错误处理** | ✅ 通过 | 所有错误使用 `fmt.Errorf("context: %w", err)` 包装。无静默失败。 |
| **3.2 无全局变量** | ✅ 通过 | 依赖通过结构体构造函数注入。Token 显式传递。 |

**结论:** 完全合规

---

## 3. 项目结构细化

### 3.1 目录结构

```
my-issue2md-project/
├── cmd/
│   ├── issue2md/
│   │   └── main.go              # CLI 入口
│   └── issue2mdweb/
│       └── main.go              # 未来: Web 界面
│
├── internal/
│   ├── parser/
│   │   ├── parser.go            # URL 解析逻辑
│   │   ├── parser_test.go       # 表格驱动测试
│   │   └── types.go             # ResourceType, ParsedURL
│   │
│   ├── github/
│   │   ├── client.go            # Client 结构体, FetchResource()
│   │   ├── types.go             # Resource, Comment, Reactions
│   │   ├── issue.go             # Issue API 调用
│   │   ├── pullrequest.go       # PR API 调用 (含 review 评论)
│   │   ├── discussion.go        # Discussion GraphQL 调用
│   │   ├── pagination.go        # 分页工具
│   │   └── client_test.go       # 基于 httptest 的测试
│   │
│   ├── converter/
│   │   ├── converter.go         # Converter 结构体, Convert()
│   │   ├── frontmatter.go       # YAML frontmatter 生成
│   │   ├── content.go           # 正文/内容渲染
│   │   ├── comments.go          # 评论渲染 (含排序)
│   │   ├── reactions.go         # Reactions 格式化
│   │   ├── options.go           # Options 结构体
│   │   └── converter_test.go    # 表格驱动输出测试
│   │
│   └── cli/
│       ├── config.go            # Config 结构体, ParseConfig()
│       ├── runner.go            # Runner 结构体, Run()
│       └── errors.go            # CLI 专用错误格式化
│
├── pkg/
│   └── markdown/                # (可选) 共享 markdown 工具
│       └── writer.go
│
├── web/
│   ├── templates/               # 未来: HTML 模板
│   └── static/                  # 未来: CSS/JS 资源
│
├── specs/
│   └── 001-core-functionality/
│       ├── spec.md
│       ├── api-sketch.md
│       └── plan.md              # 本文档
│
├── go.mod
├── go.sum
├── Makefile
├── README.md
├── CLAUDE.md
└── constitutiom.md
```

### 3.2 包职责

| 包 | 职责 | 依赖于 |
|-----|------|--------|
| `parser` | URL 解析、类型识别、验证 | 无 |
| `github` | API 通信、数据获取、分页 | `parser` |
| `converter` | Markdown 生成、格式化、模板渲染 | `github` |
| `cli` | 用户交互、配置、编排 | `parser`, `github`, `converter` |

### 3.3 依赖流向

```
parser (无依赖)
   ↓
github (依赖于: parser)
   ↓
converter (依赖于: github)
   ↓
cli (依赖于: parser, github, converter)
```

---

## 4. 核心数据结构

### 4.1 Parser 类型

```go
// Package: internal/parser

// ResourceType 表示 GitHub 资源的类型
type ResourceType string

const (
    ResourceIssue       ResourceType = "issue"
    ResourcePullRequest ResourceType = "pull_request"
    ResourceDiscussion  ResourceType = "discussion"
)

// ParsedURL 包含从 GitHub URL 解析出的信息
type ParsedURL struct {
    Type     ResourceType // "issue", "pull_request", 或 "discussion"
    Owner    string       // 仓库所有者 (用户或组织)
    Repo     string       // 仓库名称
    Number   int          // Issue/PR/Discussion 编号
    Original string       // 原始 URL 字符串
}
```

### 4.2 GitHub 客户端类型

```go
// Package: internal/github

// Client 处理 GitHub API 交互
type Client struct {
    httpClient *http.Client
    token      string           // 可选的 GITHUB_TOKEN
    baseURL    string           // "https://api.github.com"
    graphqlURL string           // "https://api.github.com/graphql"
}

// Resource 表示统一的 GitHub 资源 (Issue/PR/Discussion)
type Resource struct {
    // 元数据
    Type        ResourceType
    URL         string
    Number      int
    Title       string
    Author      string
    AuthorURL   string          // GitHub 个人主页 URL
    CreatedAt   time.Time
    UpdatedAt   time.Time
    State       string          // "open", "closed", "merged"
    StateReason string          // 例如: "completed", "not_planned"

    // 内容
    Body        string
    Comments    []Comment

    // 统计
    CommentsCount    int
    ReactionsSummary *Reactions

    // Discussion 特有
    IsAnswered    bool            // 是否有被采纳的答案
    AnswerURL     string          // 被采纳答案的 URL (如果有)
}

// Comment 表示单条评论
type Comment struct {
    ID           int64           // GitHub 评论 ID
    Author       string
    AuthorURL    string
    Body         string
    CreatedAt    time.Time
    UpdatedAt    time.Time

    // PR Review 评论特有
    IsReviewComment bool         // PR review 评论为 true
    CommitID        string       // 关联的 commit SHA
    Path            string       // 文件路径 (review 评论)
    Line            int          // 行号 (review 评论)

    // Discussion 特有
    IsAnswer     bool           // 如果是被采纳的答案则为 true

    // Reactions
    Reactions    *Reactions      // 可选，启用时
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
    Total      int
}
```

### 4.3 Converter 类型

```go
// Package: internal/converter

// Options 控制转换行为
type Options struct {
    EnableReactions     bool  // 在输出中包含 reactions
    EnableUserLinks     bool  // 将用户名渲染为链接
    IncludeAnswerBadge  bool  // 为 Discussion 答案显示特殊徽章 (Discussion 始终为 true)
    TimestampFormat     string // 时间格式字符串
}

// Converter 将 GitHub 资源转换为 Markdown
type Converter struct {
    options      Options
    template     *template.Template  // Markdown 模板
}

// FrontmatterData 表示 YAML frontmatter 字段
type FrontmatterData struct {
    Title         string    `yaml:"title"`
    URL           string    `yaml:"url"`
    Type          string    `yaml:"type"`
    Author        string    `yaml:"author"`
    CreatedAt     string    `yaml:"created_at"`
    State         string    `yaml:"state"`
    CommentsCount int       `yaml:"comments_count"`
    GeneratedAt   string    `yaml:"generated_at"`
}
```

### 4.4 CLI 类型

```go
// Package: internal/cli

// Config 表示解析后的 CLI 配置
type Config struct {
    URL              string   // GitHub URL (位置参数)
    OutputFile       string   // 输出文件路径 (可选位置参数)
    EnableReactions  bool     // -enable-reactions 标志
    EnableUserLinks  bool     // -enable-user-links 标志
    Token            string   // 从环境变量读取的 GITHUB_TOKEN
}

// Runner 编排转换流程
type Runner struct {
    config    *Config
    parser    *parser.Parser
    client    *github.Client
    converter *converter.Converter
    stdout    io.Writer
    stderr    io.Writer
}
```

---

## 5. 接口设计

### 5.1 Parser 接口

```go
// Package: internal/parser

// ParseURL 解析 GitHub URL 并提取资源信息
//
// 支持的 URL 格式:
//   - https://github.com/{owner}/{repo}/issues/{number}
//   - https://github.com/{owner}/{repo}/pull/{number}
//   - https://github.com/{owner}/{repo}/discussions/{number}
//
// 返回:
//   - *ParsedURL: 解析后的 URL 信息
//   - error: URL 无效时返回 ErrInvalidURLFormat
//           不支持的资源类型返回 ErrUnsupportedResourceType
func ParseURL(urlStr string) (*ParsedURL, error)

// Validate 检查解析后的 URL 是否有效
func (p *ParsedURL) Validate() error

// APIURL 返回此资源的 API 端点
func (p *ParsedURL) APIURL() string
```

### 5.2 GitHub 客户端接口

```go
// Package: internal/github

// NewClient 创建新的 GitHub API 客户端
//
// 参数:
//   - token: 可选的 Personal Access Token (可为空字符串)
//
// 返回:
//   - *Client: 配置好的客户端实例
func NewClient(token string) *Client

// FetchResource 根据 URL 获取 GitHub 资源
//
// 此方法:
//   1. 根据资源类型调用相应的 API
//   2. 自动处理分页
//   3. 合并 PR review 评论和普通评论
//   4. 返回统一的 Resource 结构体
//
// 返回:
//   - *Resource: 获取的资源数据
//   - error: API 调用、解析或网络错误的包装错误
func (c *Client) FetchResource(parsed *parser.ParsedURL) (*Resource, error)

// FetchIssue 获取 Issue 及所有评论
func (c *Client) FetchIssue(owner, repo string, number int) (*Resource, error)

// FetchPullRequest 获取 PR 及所有评论和 review 评论
func (c *Client) FetchPullRequest(owner, repo string, number int) (*Resource, error)

// FetchDiscussion 获取 Discussion 及所有评论 (通过 GraphQL)
func (c *Client) FetchDiscussion(owner, repo string, number int) (*Resource, error)

// fetchAllComments 分页获取所有评论
func (c *Client) fetchAllComments(apiURL string) ([]Comment, error)
```

### 5.3 Converter 接口

```go
// Package: internal/converter

// NewConverter 创建新的 Markdown 转换器
//
// 参数:
//   - opts: 转换选项
//
// 返回:
//   - *Converter: 配置好的转换器实例
func NewConverter(opts Options) *Converter

// Convert 将 GitHub Resource 转换为 Markdown 格式
//
// 输出包括:
//   1. YAML frontmatter
//   2. 标题和元数据头部
//   3. 正文内容
//   4. 所有评论 (按时间排序)
//
// 返回:
//   - string: 完整的 Markdown 文档
func (c *Converter) Convert(res *github.Resource) string

// ConvertToWriter 直接将 Markdown 写入 io.Writer
//
// 对于大型资源更高效，避免在内存中构建整个文档。
//
// 返回:
//   - error: 任何写入或模板错误
func (c *Converter) ConvertToWriter(res *github.Resource, w io.Writer) error

// 渲染方法 (内部，不导出)
func (c *Converter) renderFrontmatter(res *github.Resource) string
func (c *Converter) renderHeader(res *github.Resource) string
func (c *Converter) renderBody(res *github.Resource) string
func (c *Converter) renderComments(comments []github.Comment) string
func (c *Converter) renderReactions(r *github.Reactions) string
func (c *Converter) formatUser(username, userURL string) string
func (c *Converter) formatTime(t time.Time) string
```

### 5.4 CLI 接口

```go
// Package: internal/cli

// ParseConfig 解析命令行参数和环境变量
//
// 参数:
//   - args: 命令行参数 (通常是 os.Args[1:])
//
// 返回:
//   - *Config: 解析后的配置
//   - error: 标志解析错误或验证错误
func ParseConfig(args []string) (*Config, error)

// Validate 检查配置是否有效
func (c *Config) Validate() error

// NewRunner 创建新的 CLI 运行器
//
// 参数:
//   - cfg: 解析后的配置
//   - stdout: 标准输出写入器 (默认 os.Stdout)
//   - stderr: 错误输出写入器 (默认 os.Stderr)
//
// 返回:
//   - *Runner: 配置好的运行器实例
func NewRunner(cfg *Config, stdout, stderr io.Writer) *Runner

// Run 执行转换流程
//
// 此方法:
//   1. 解析 GitHub URL
//   2. 从 GitHub API 获取资源
//   3. 转换为 Markdown
//   4. 写入 stdout 或文件
//
// 返回:
//   - error: 执行期间的任何错误 (已格式化为 stderr 输出)
func (r *Runner) Run() error
```

---

## 6. 实施阶段

### 第一阶段: 基础 (第 1 周)

| 任务 | 包 | 交付物 |
|------|-----|--------|
| URL 解析器 | `internal/parser` | 带表格驱动测试的 ParseURL 函数 |
| 基础客户端 | `internal/github` | 带 Issue 获取功能的 Client 结构体 |
| 简单转换器 | `internal/converter` | 基本 Markdown 输出 |
| CLI 骨架 | `internal/cli` | 配置解析和基础运行器 |

**退出标准:**
- 可以解析 Issue URL
- 可以获取公开 Issue
- 可以输出基本 Markdown

### 第二阶段: 核心功能 (第 2 周)

| 任务 | 包 | 交付物 |
|------|-----|--------|
| PR 支持 | `internal/github` | FetchPullRequest 带 review 评论 |
| 评论排序 | `internal/converter` | 按时间合并评论 |
| 分页处理 | `internal/github` | 带正确分页的 fetchAllComments |
| 错误处理 | 所有包 | 完善的错误包装 |

**退出标准:**
- 完整的 Issue/PR 支持
- 处理 100+ 条评论的资源
- 清晰的错误消息

### 第三阶段: 高级功能 (第 3 周)

| 任务 | 包 | 交付物 |
|------|-----|--------|
| Discussion 支持 | `internal/github` | 基于 GraphQL 的 discussion 获取 |
| Reactions | `internal/converter` | Reactions 渲染 |
| 用户链接 | `internal/converter` | 可配置的用户链接格式化 |
| 答案徽章 | `internal/converter` | Discussion 答案高亮 |

**退出标准:**
- 完整的 Discussion 支持
- 所有标志正常工作
- 完整的测试覆盖

### 第四阶段: 完善 (第 4 周)

| 任务 | 包 | 交付物 |
|------|-----|--------|
| 集成测试 | `internal/...` | 使用真实 GitHub 数据的端到端测试 |
| 文档 | `README.md` | 使用示例和安装指南 |
| 性能优化 | `internal/github` | 大型资源优化 |
| 发布 | `cmd/issue2md` | v1.0.0 标签发布 |

---

## 7. 测试策略

### 7.1 单元测试

| 包 | 测试类型 | 覆盖率目标 |
|-----|---------|-----------|
| `parser` | 表格驱动 | 100% |
| `converter` | 表格驱动 | 90%+ |
| `github` | `httptest` 模拟 | 80%+ |
| `cli` | 集成 + 模拟 | 70%+ |

### 7.2 集成测试

```go
// File: internal/integration_test.go
//go:build integration
// +build integration

package integration_test

func TestRealGitHubIssue(t *testing.T) {
    // 使用真实的公开 GitHub Issue 测试
    // 使用知名的稳定 Issue (例如 golang/go 仓库中的)
}

func TestRealGitHubPR(t *testing.T) {
    // 使用真实的公开 GitHub PR 测试
}

func TestRealGitHubDiscussion(t *testing.T) {
    // 使用真实的公开 GitHub Discussion 测试
}
```

### 7.3 表格驱动测试示例

```go
// File: internal/parser/parser_test.go

func TestParseURL(t *testing.T) {
    tests := []struct {
        name    string
        url     string
        want    *ParsedURL
        wantErr error
    }{
        {
            name: "有效的 Issue URL",
            url:  "https://github.com/golang/go/issues/12345",
            want: &ParsedURL{
                Type:   ResourceIssue,
                Owner:  "golang",
                Repo:   "go",
                Number: 12345,
            },
            wantErr: nil,
        },
        {
            name:    "无效的 URL",
            url:     "not-a-url",
            want:    nil,
            wantErr: ErrInvalidURLFormat,
        },
        // ... 更多测试用例
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseURL(tt.url)
            if tt.wantErr != nil {
                if !errors.Is(err, tt.wantErr) {
                    t.Errorf("ParseURL() error = %v, wantErr %v", err, tt.wantErr)
                }
                return
            }
            // ... 断言
        })
    }
}
```

---

## 8. 依赖管理

### 8.1 Go 模块

```go
// go.mod

module github.com/didi/issue2md

go 1.21

require (
    github.com/google/go-github v69.0.0
    golang.org/x/oauth2 v0.20.0
)
```

### 8.2 开发依赖

```go
// go.mod (开发)

require (
    github.com/stretchr/testify v1.9.0  // 用于断言 (可选，可用标准库)
)
```

**注意:** 项目宪法倾向于使用标准库。使用 `testify` 是可选的，可以用 `testing` 包的断言替代。

### 8.3 GitHub API 范围

`GITHUB_TOKEN` 所需的 OAuth 范围:
- **公开仓库:** 不需要 token (限制 60 请求/小时)
- **私有仓库:** `repo` 范围
- **提高速率限制:** `public_repo` 范围 (5000 请求/小时)

---

## 附录 A: 错误代码参考

| 退出码 | 场景 | 示例消息 |
|--------|------|----------|
| 1 | URL 格式无效 | `error: invalid GitHub URL format: not-a-url` |
| 1 | 不支持的资源类型 | `error: unsupported resource type: wiki` |
| 1 | 资源不存在 (404) | `error: failed to fetch issue: 404 Not Found` |
| 1 | 未授权 (401) | `error: unauthorized: set GITHUB_TOKEN for private repos` |
| 1 | 速率限制 (403) | `error: API rate limit exceeded: 403 Forbidden` |
| 1 | 网络错误 | `error: failed to connect: dial tcp: connection refused` |
| 1 | 文件写入错误 | `error: failed to write output: permission denied` |

---

## 附录 B: Makefile 模板

```makefile
# issue2md 的 Makefile

.PHONY: test build clean install integration-test

# Go 参数
GOCMD=go
GOBUILD=$(GOCMD) build
GOTEST=$(GOCMD) test
GOGET=$(GOCMD) get
GOMOD=$(GOCMD) mod

# 二进制名称
BINARY_NAME=issue2md

build:
	$(GOBUILD) -o $(BINARY_NAME) ./cmd/issue2md

test:
	$(GOTEST) -v ./...

integration-test:
	$(GOTEST) -v -tags=integration ./...

clean:
	$(GOCMD) clean
	rm -f $(BINARY_NAME)

install:
	$(GOBUILD) -o $(GOPATH)/bin/$(BINARY_NAME) ./cmd/issue2md

deps:
	$(GOMOD) download
	$(GOMOD) tidy
```

---

**文档结束**

*本方案是实现 Spec 001 的技术蓝图。所有开发应遵循本方案和项目宪法。*
