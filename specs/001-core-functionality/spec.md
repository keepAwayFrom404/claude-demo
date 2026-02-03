# Spec 001: Core Functionality - GitHub to Markdown Converter

**Status:** Draft
**Created:** 2026-02-03
**Priority:** P0 (MVP)

---

## 1. 用户故事 (User Stories)

### 1.1 CLI版本 (MVP)

**作为** 一名开发者或文档维护者

**我想要** 通过一个简单的命令行工具，输入GitHub Issue/PR/Discussion的URL

**以便于** 将这些讨论内容自动转换为格式化的Markdown文件，用于归档、文档编写或知识管理

**核心价值：**
- 一键归档重要的GitHub讨论，防止信息丢失
- 简化文档编写流程，直接复用GitHub上的高质量讨论
- 支持离线查看和分享技术讨论

### 1.2 Web版本 (未来用户故事)

**作为** 一名非技术用户或需要频繁转换的用户

**我想要** 通过Web界面输入URL并下载Markdown文件

**以便于** 无需安装CLI工具即可使用转换功能

**实现优先级:** P2 (MVP之后的迭代)

---

## 2. 功能性需求 (Functional Requirements)

### 2.1 URL自动识别与解析

**FR-1.1 URL类型识别**
- 工具必须自动识别以下URL类型：
  - **Issue URL:** `https://github.com/<owner>/<repo>/issues/<number>`
  - **PR URL:** `https://github.com/<owner>/<repo>/pull/<number>`
  - **Discussion URL:** `https://github.com/<org>/<repo>/discussions/<number>`

**FR-1.2 URL格式支持**
- 支持标准Web URL（上述格式）
- 暂不支持API URL或简化格式

**FR-1.3 解析逻辑**
- 通过URL路径段识别资源类型：
  - `/issues/` → Issue
  - `/pull/` → Pull Request
  - `/discussions/` → Discussion
- 解析 owner, repo, number 用于API调用

### 2.2 支持的资源类型与内容

**FR-2.1 Issue内容提取**
- 标题 (Title)
- 作者 (Author)
- 创建时间 (Created At)
- 状态 (State: Open/Closed)
- 主楼内容 (Body)
- 所有评论 (Comments)

**FR-2.2 Pull Request内容提取**
- 标题、作者、创建时间、状态
- 描述 (Description)
- 所有普通评论 (Comments)
- 所有Review评论 (Review Comments)
- **不包含:** diff信息、commits历史、文件变更列表

**FR-2.3 Discussion内容提取**
- 标题、作者、创建时间、状态
- 主楼内容
- 所有评论
- 如果有被标记为Answer的评论，添加显著标记

**FR-2.4 可选内容（通过Flag控制）**
- **Reactions统计:** `-enable-reactions` Flag开启时，在主楼和每个评论下方显示reactions（👍❤️等）
- **用户链接:** `-enable-user-links` Flag开启时，将用户名渲染为指向其GitHub主页的链接

### 2.3 GitHub API集成

**FR-3.1 API版本**
- 使用 GitHub REST API v3
- 基础URL: `https://api.github.com`

**FR-3.2 认证机制**
- **仅支持公有仓库** (MVP)
- 可选认证: 通过环境变量 `GITHUB_TOKEN` 传入Personal Access Token
- 如果设置token，在API请求头中添加: `Authorization: token <token>`
- **禁止**提供 `--token` 命令行参数（防止Shell历史泄露）

**FR-3.3 分页处理**
- 对于评论数量较多的资源，必须实现分页处理
- 默认每页30条（GitHub API默认值）
- 自动遍历所有页面直到获取完整数据

**FR-3.4 Rate Limiting**
- 遇到API限流（403或429状态码），直接透传GitHub错误信息给用户
- 不实现自动重试机制

### 2.4 内容处理规则

**FR-4.1 评论排序**
- 所有评论按时间**正序**排列（从旧到新）
- PR Review Comments与普通Comments合并，统一按时间线展示
- 不保留GitHub的嵌套回复结构，扁平化展示

**FR-4.2 Discussion Answer标记**
- 如果Discussion中某评论被标记为Answer，使用以下格式突出显示：
  ```markdown
  > ✅ **[ACCEPTED ANSWER]**
  >
  > 评论内容...
  ```

**FR-4.3 特殊内容处理**
- 代码块: 保留原始语法高亮标记（如 \`\`\`go）
- 图片/附件: 保留原始链接，不下载到本地
- 链接: 保持Markdown格式不变

### 2.5 命令行接口设计

**FR-5.1 命令格式**
```bash
issue2md [flags] <url> [output_file]
```

**FR-5.2 参数说明**
- `<url>`: (必需) GitHub Issue/PR/Discussion的完整URL
- `[output_file]`: (可选) 输出文件路径
  - 如果提供，写入指定文件
  - 如果省略，输出到stdout

**FR-5.3 Flags**
```
-enable-reactions     Include reactions statistics (👍❤️🎉等)
-enable-user-links    Render usernames as links to GitHub profiles
```

**FR-5.4 环境变量**
```
GITHUB_TOKEN    (可选) Personal Access Token for API authentication
```

**FR-5.5 使用示例**
```bash
# 输出到stdout
issue2md https://github.com/owner/repo/issues/123

# 输出到文件
issue2md https://github.com/owner/repo/pull/456 output.md

# 启用reactions和用户链接
issue2md -enable-reactions -enable-user-links https://github.com/org/repo/discussions/78 discussion.md
```

### 2.6 Markdown输出格式

**FR-6.1 Frontmatter (YAML)**
每个输出文件必须包含YAML frontmatter:
```yaml
---
title: "[Issue/PR/Discussion] Title"
url: "https://github.com/owner/repo/issues/123"
type: "issue" | "pull_request" | "discussion"
author: "username"
created_at: "2024-01-15T10:30:00Z"
state: "open" | "closed" | "merged"
comments_count: 42
generated_at: "2024-01-23T15:45:00Z"
---
```

**FR-6.2 正文结构**
```markdown
# [Type #123] Title

**Author:** @username
**Created:** 2024-01-15 10:30 UTC
**State:** Open

## Description

[主楼内容]

---

## Comments

### @username on 2024-01-15 11:00 UTC

[评论内容]

### @otheruser on 2024-01-15 12:30 UTC

[评论内容]
```

---

## 3. 非功能性需求 (Non-Functional Requirements)

### 3.1 架构设计

**NFR-1.1 关注点分离**
- **GitHub客户端层:** 负责API调用、认证、分页
- **URL解析层:** 负责URL类型识别和参数提取
- **内容渲染层:** 负责将GitHub数据转换为Markdown
- **CLI层:** 负责参数解析和用户交互

**NFR-1.2 可测试性**
- 核心逻辑与CLI层解耦，便于单元测试
- GitHub客户端层应定义接口，支持mock测试

### 3.2 错误处理

**NFR-2.1 错误场景**
- **URL格式无效:** 输出清晰错误信息，退出码1
- **资源不存在 (404):** 提示用户检查URL，退出码1
- **私有仓库未授权:** 提示设置GITHUB_TOKEN，退出码1
- **网络超时:** 显示网络错误，退出码1
- **API错误:** 透传GitHub错误信息，退出码1

**NFR-2.2 错误输出格式**
- 所有错误信息输出到stderr
- 错误信息格式: `error: <具体错误信息>`
- 示例: `error: failed to fetch issue: 404 Not Found`

### 3.3 性能考虑

**NFR-3.1 响应时间**
- 对于包含100条评论的资源，总处理时间应 < 5秒

**NFR-3.2 内存占用**
- 优化内存使用，避免一次性加载超大Issue到内存
- 考虑流式写入Markdown文件

### 3.4 代码质量

**NFR-4.1 测试覆盖**
- 核心逻辑单元测试覆盖率 ≥ 80%
- 必须包含集成测试（使用真实的GitHub API或mock）

**NFR-4.2 代码规范**
- 遵循Go官方代码风格
- 遵循项目宪法（简单性、测试先行、明确性）

---

## 4. 验收标准 (Acceptance Criteria)

### 4.1 功能验收

**AC-1: URL识别**
- [ ] 能正确识别Issue URL并提取数据
- [ ] 能正确识别PR URL并提取数据
- [ ] 能正确识别Discussion URL并提取数据
- [ ] 对无效URL返回错误信息

**AC-2: Issue转换**
- [ ] 能正确转换公开Issue的标题、作者、时间、状态
- [ ] 能正确转换Issue主楼内容和所有评论
- [ ] 输出包含正确的YAML frontmatter

**AC-3: PR转换**
- [ ] 能正确转换PR的标题、作者、时间、状态
- [ ] 能正确合并普通Comments和Review Comments按时间排序
- [ ] 不包含diff和commits信息

**AC-4: Discussion转换**
- [ ] 能正确转换Discussion主楼和评论
- [ ] 被标记为Answer的评论有显著标记

**AC-5: Flags功能**
- [ ] `-enable-reactions` 能正确显示reactions统计
- [ ] `-enable-user-links` 能正确渲染用户链接

**AC-6: 输出控制**
- [ ] 不指定output_file时，正确输出到stdout
- [ ] 指定output_file时，正确写入文件
- [ ] 环境变量GITHUB_TOKEN能正确传递给API请求

### 4.2 边缘场景验收

**AC-7: 边缘数据**
- [ ] 能处理只有主楼无评论的Issue
- [ ] 能处理评论数超过100条的资源（分页）
- [ ] 能处理包含代码块、图片的评论

**AC-8: 错误处理**
- [ ] 输入不存在的URL，返回404错误
- [ ] 访问私有仓库未提供token，返回403错误提示
- [ ] URL格式错误，返回清晰错误信息

### 4.3 测试用例示例

```gherkin
Scenario: 成功转换Issue
  Given 一个公开的GitHub Issue URL
  When 执行命令 issue2md <URL>
  Then 正确输出包含标题、作者、时间、状态、内容、评论的Markdown
  And 包含正确的YAML frontmatter

Scenario: 成功转换PR并合并评论
  Given 一个包含Review Comments的PR URL
  When 执行命令 issue2md <PR_URL> -enable-reactions
  Then 所有评论按时间正序排列
  And Review Comments和普通Comments混合展示
  And 显示reactions统计

Scenario: 访问私有仓库未授权
  Given 一个私有仓库的Issue URL
  And 未设置GITHUB_TOKEN环境变量
  When 执行命令 issue2md <URL>
  Then 返回错误提示设置GITHUB_TOKEN
  And 退出码为1

Scenario: 标记Discussion Answer
  Given 一个有Accepted Answer的Discussion
  When 执行命令 issue2md <DISCUSSION_URL>
  Then Answer评论有 ✅ **[ACCEPTED ANSWER]** 标记
```

---

## 5. 输出格式示例

### 5.1 Issue输出示例

```markdown
---
title: "[Issue #123] Feature: Add dark mode support"
url: "https://github.com/example/app/issues/123"
type: "issue"
author: "johndoe"
created_at: "2024-01-15T10:30:00Z"
state: "open"
comments_count: 3
generated_at: "2024-01-23T15:45:00Z"
---

# [Issue #123] Feature: Add dark mode support

**Author:** @johndoe
**Created:** 2024-01-15 10:30 UTC
**State:** Open

## Description

It would be great to add dark mode support to our application. This will improve user experience in low-light environments.

### Proposed implementation

- Use CSS variables for theming
- Add a toggle button in settings
- Persist user preference in localStorage

---

## Comments

### @janedoe on 2024-01-15 11:00 UTC

Great idea! I'd like to work on this. Should we use a specific color palette?

### @johndoe on 2024-01-15 11:30 UTC

@janedoe Yes, let's use the Material Design dark theme colors. Here's a reference:

```css
:root {
  --background: #121212;
  --surface: #1e1e1e;
}
```

### @janedoe on 2024-01-15 14:00 UTC

Perfect, I'll start working on it! 👍
```

### 5.2 PR输出示例

```markdown
---
title: "[PR #456] Implement dark mode toggle"
url: "https://github.com/example/app/pull/456"
type: "pull_request"
author: "janedoe"
created_at: "2024-01-20T09:00:00Z"
state: "merged"
comments_count: 5
generated_at: "2024-01-23T15:45:00Z"
---

# [PR #456] Implement dark mode toggle

**Author:** @janedoe
**Created:** 2024-01-20 09:00 UTC
**State:** Merged

## Description

This PR implements the dark mode feature as discussed in #123.

### Changes

- Added CSS variables for light/dark themes
- Implemented theme toggle in settings page
- Added localStorage persistence

---

## Comments

### @johndoe on 2024-01-20 09:30 UTC

Thanks for working on this! The approach looks good.

### @reviewer1 on 2024-01-20 10:00 UTC

**Review Comment:** In `theme.css` line 15, consider using a more descriptive variable name.

### @janedoe on 2024-01-20 10:15 UTC

@reviewer1 Good catch! I'll update it to `--color-background-primary`.

### @johndoe on 2024-01-20 11:00 UTC

Looks ready to merge! 🚀

### @janedoe on 2024-01-20 11:30 UTC

Updated the variable names as suggested. Ready for review.
```

### 5.3 Discussion输出示例

```markdown
---
title: "[Discussion #78] Best practices for error handling?"
url: "https://github.com/example/framework/discussions/78"
type: "discussion"
author: "newdev"
created_at: "2024-01-10T08:00:00Z"
state: "open"
comments_count: 4
generated_at: "2024-01-23T15:45:00Z"
---

# [Discussion #78] Best practices for error handling?

**Author:** @newdev
**Created:** 2024-01-10 08:00 UTC
**State:** Open

## Description

I'm new to this framework and wondering about the best practices for error handling. Should we use try-catch or the Result type?

---

## Comments

### @seniordev on 2024-01-10 08:30 UTC

Great question! For this framework, we recommend using the Result type. Here's why:

1. **Explicit error handling:** Forces you to handle errors
2. **Better type safety:** The compiler ensures you handle both cases
3. **Composable:** You can chain operations easily

Example:
```typescript
const result = await fetchUser(id)
  .map(user => user.posts)
  .andThen(posts => validatePosts(posts));
```

> ✅ **[ACCEPTED ANSWER]**
>
> This answer was marked as accepted by the discussion author.

### @newdev on 2024-01-10 09:00 UTC

@seniordev This is super helpful! Thank you.

### @anotherdev on 2024-01-10 10:00 UTC

I'd add that you should also consider using the `?` operator if you're using TypeScript 5+. It makes the code much cleaner.

### @seniordev on 2024-01-10 10:30 UTC

@anotherdev Excellent point! The `?` operator is definitely more ergonomic for linear error propagation.
```

---

## 6. 实现优先级

### Phase 1: 核心功能 (MVP)
- P0: URL识别和解析
- P0: GitHub API客户端（Issue/PR）
- P0: 基础Markdown渲染
- P0: CLI参数解析
- P0: 错误处理

### Phase 2: 完善功能
- P1: Discussion支持
- P1: Reactions Flag
- P1: User Links Flag
- P1: 完整测试覆盖

### Phase 3: 增强体验
- P2: 进度条（处理大量评论时）
- P2: 缓存机制（避免重复API调用）
- P2: 批量处理多个URL

---

## 7. 未来扩展

- Web界面支持
- 支持私有仓库（改进认证流程）
- 支持GitLab等其他平台
- 自定义Markdown模板
- 导出为PDF/HTML