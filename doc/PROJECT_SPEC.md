# 📘 渐进式披露知识库与 MCP 服务项目开发规范 (PROJECT_SPEC)

## 1. 项目概述
本项目是一个 **SaaS 级多租户知识库管理系统**，核心包含两大模块：
1.  **Web 管理端**：提供可视化的 Git 仓库挂载、L0~L5 知识树编排、分类拖拽排序、构建任务队列管理及变更记录治理功能。
2.  **MCP Server**：基于 Model Context Protocol 协议，为大模型 Agent 提供“文件系统语义”的渐进式知识检索服务。

### 1.1 核心技术栈
-   **后端**: Go 1.22+ / Gin / GORM / go-git / goldmark
-   **MCP**: Go 原生实现 JSON-RPC 2.0 over stdio/SSE (推荐 `mark3labs/mcp-go`)
-   **数据库**: MySQL 8.0+ (InnoDB, utf8mb4)
-   **前端**: Vue 3 (Composition API) + Vite + TypeScript + Pinia + Element Plus
-   **LLM**: OpenAI 兼容 API (强制 JSON Mode)
-   **任务调度**: Go 内存队列 (Map + Channel) + MySQL 持久化双缓冲架构

---

## 2. 核心概念定义 (L0~L5)

系统严格遵循以下渐进式层级定义，所有 LLM 解析与存储均以此为准：

| 层级 | 名称 | 核心内容 | 对应 Prompt 字段 | 排序规则 | 虚拟路径格式 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **L0** | 知识库索引 | 所有分类清单 + 分类简介 | 聚合 `L0_panoramic_summary` | Web 手动拖拽 | `/{kb_slug}/_index.md` |
| **L1** | 分类目录页 | 子分类清单 + 直属文档清单 + 简介 | 聚合子节点摘要 | **Web 手动拖拽** | `/{kb_slug}/{l1_nested_path}/_index.md` |
| **L2** | 文档首页 | 文档摘要 + 章节/知识点目录 | `L1_category_title/desc` + `L2_knowledge_directory` | **按更新时间倒序** | `/{kb_slug}/{l1_path}/{doc}.md` |
| **L3** | 知识点页 | 核心叙述、原理、业务逻辑 (剔除代码/表格) | `L3_full_content` | **按原文出现顺序** | `.../{doc}/{section}.l3.md` |
| **L4** | 知识详情 | 具体参数、配置、详细步骤 | `L4_detail_content` | 跟随所属 L3 | `.../{doc}/{sec}-{detail}.l4.md` |
| **L5** | 附加信息 | 代码示例、表格、扩展阅读 | 从 L4 按长度拆分 | 跟随所属 L4 | `.../{doc}/{sec}-{detail}.l5.md` |

> **⚠️ 完整性原则**
> `L1_metadata + L2_structure + L3_narrative + L4_details` 合并后必须能无损还原原文全部信息。Builder 引擎需执行校验，缺失则重试。L3 聚焦“为什么/是什么”，L4 聚焦“怎么做/具体参数”，两者内容严格互斥。

---

## 3. 路径与挂载体系

### 3.1 双轨路径模型
-   **Git Physical Path**: 文件在 Git 仓库中的真实相对路径。用于 Diff 定位、链接解析溯源、版本追溯。
-   **Virtual Nav Path**: MCP 暴露的导航路径。**不含租户标识**，纯知识组织结构。

### 3.2 Git → L1 挂载机制
通过 `repo_mounts` 表配置映射关系，支持**下级覆盖上级**（最长前缀匹配优先）。

**路径转换公式**：
```text
VirtualPath = /{kb_slug}/{l1_nested_path}/{git_physical_path - strip_prefix}
```

### 3.3 文件系统语义约定
-   **目录访问**: `explore_node("auth/")` → 自动读取 `auth/_index.md`
-   **文件访问**: `explore_node("auth/jwt.md")` → 精确匹配 L2 节点
-   **拆解文件命名**: L3/L4/L5 使用 `.l3.md` / `.l4.md` / `.l5.md` 后缀，避免与原始文件冲突。
-   **链接零重写**: 因拆解内容与原文同目录，Markdown 内相对链接 `[t](./a.md)` 保持原样，Agent 可直接用作下次 `node_path`。

---

## 4. 数据库设计 (MySQL)

### 4.1 核心表结构

```sql
-- 租户表
CREATE TABLE tenants (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(50) NOT NULL UNIQUE,
    owner_user_id BIGINT UNSIGNED NOT NULL
);

-- 知识库表
CREATE TABLE knowledge_bases (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT UNSIGNED NOT NULL,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    description TEXT,
    visibility ENUM('private','shared','public') DEFAULT 'private',
    INDEX idx_tenant (tenant_id)
);

-- 分类表 (L0/L1 嵌套与排序)
CREATE TABLE categories (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    kb_id BIGINT UNSIGNED NOT NULL,
    parent_id BIGINT UNSIGNED DEFAULT 0,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    description TEXT,
    sort_order INT DEFAULT 0,
    INDEX idx_kb_parent_sort (kb_id, parent_id, sort_order)
);

-- Git 仓库与挂载
CREATE TABLE git_repos (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    kb_id BIGINT UNSIGNED NOT NULL,
    repo_url VARCHAR(255) NOT NULL,
    branch VARCHAR(50) DEFAULT 'main',
    last_sync_commit VARCHAR(50)
);

CREATE TABLE repo_mounts (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    repo_id BIGINT UNSIGNED NOT NULL,
    category_id BIGINT UNSIGNED NOT NULL,
    git_path VARCHAR(500) NOT NULL,
    strip_prefix VARCHAR(500) NOT NULL,
    priority INT GENERATED ALWAYS AS (LENGTH(git_path)) STORED,
    INDEX idx_repo_priority (repo_id, priority DESC)
);

-- 知识节点表 (核心)
CREATE TABLE knowledge_nodes (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    kb_id BIGINT UNSIGNED NOT NULL,
    virtual_path VARCHAR(500) NOT NULL UNIQUE,
    git_physical_path VARCHAR(500) NOT NULL,
    git_commit_hash VARCHAR(50) NOT NULL DEFAULT '' COMMENT '构建时的Git Commit Hash',
    level ENUM('L2','L3','L4','L5') NOT NULL COMMENT 'L0/L1为动态视图不存表',
    title VARCHAR(255) NOT NULL,
    content_md LONGTEXT,
    children_paths JSON,
    sort_order INT DEFAULT 0,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_git_path (git_physical_path),
    INDEX idx_kb_level (kb_id, level),
    INDEX idx_parent_sort (kb_id, virtual_path, sort_order)
);

-- 修订日志表 (独立存储)
CREATE TABLE node_revision_logs (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    kb_id BIGINT UNSIGNED NOT NULL,
    git_physical_path VARCHAR(500) NOT NULL,
    from_commit VARCHAR(50) DEFAULT '',
    to_commit VARCHAR(50) NOT NULL,
    committed_at DATETIME NOT NULL,
    author VARCHAR(100),
    change_type ENUM('added','modified','deleted','build_failed') NOT NULL,
    llm_summary TEXT,
    is_hidden TINYINT DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_path_time (git_physical_path, committed_at DESC)
);

-- 构建队列表 (仅持久化用途)
CREATE TABLE build_queue_items (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    kb_id BIGINT UNSIGNED NOT NULL,
    git_physical_path VARCHAR(500) NOT NULL,
    reason VARCHAR(50) NOT NULL,
    status ENUM('pending','processing','completed','failed') DEFAULT 'pending',
    enqueued_at DATETIME NOT NULL,
    started_at DATETIME DEFAULT NULL,
    finished_at DATETIME DEFAULT NULL,
    UNIQUE KEY uk_kb_path (kb_id, git_physical_path),
    INDEX idx_status (kb_id, status)
);

-- 授权分享表
CREATE TABLE kb_shares (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    kb_id BIGINT UNSIGNED NOT NULL,
    target_tenant_id BIGINT UNSIGNED NOT NULL,
    permission ENUM('read','mcp_only') DEFAULT 'mcp_only',
    expires_at DATETIME DEFAULT NULL,
    UNIQUE KEY uk_kb_target (kb_id, target_tenant_id)
);
```

---

## 5. 构建引擎 (Builder Pipeline)

### 5.1 解析与挂载解耦约束
-   Builder 引擎必须分为三个独立函数：`ResolveMountPath()`、`ParseDocumentContent()`、`AssembleAndPersist()`。
-   `ParseDocumentContent()` 的入参**严禁包含**任何路径信息，仅接收 Markdown 原文和分类候选列表 (`{{.CategoriesContext}}`)。
-   `ResolveMountPath()` **严禁读取文件内容**，仅基于 `git_physical_path` 和 `repo_mounts` 表做字符串匹配。

### 5.2 手动触发与全量重构规范
-   **手动触发制**：系统不提供自动构建。所有更新必须由用户在 Web 端显式触发“重新构建”操作。
-   **全量重构**：每次更新均调用 `parse-progressive.yaml` 对目标文件进行全量解析，禁止局部更新或 Diff 幅度判断。
-   **禁止内容编辑**：Web 端不提供节点内容编辑功能。`knowledge_nodes.content_md` 仅由 Builder 引擎写入。
-   **物理删除 + 日志留存**：文档删除时物理删除 `knowledge_nodes` 记录，但 `node_revision_logs` 永久保留。Git 回滚后重新构建可自动恢复节点并合并历史日志。

### 5.3 Git 版本追溯规范
-   `knowledge_nodes` 必须记录 `git_commit_hash`，作为增量 Diff 的基准版本。
-   构建时先比对旧 Hash 与当前 HEAD，相同则跳过；不同则计算 Diff **仅用于生成 `revision_summary`**。
-   LLM 全量解析始终接收完整原文，不依赖 Diff 内容。
-   Diff 计算失败不阻塞构建，仅在日志中标记警告。
-   `node_revision_logs` 记录 `from_commit` 和 `to_commit`，支持精确版本对比追溯。
-   版本号与节点内容必须在同一数据库事务内原子写入。

### 5.4 _index.md 动态生成规范
-   Builder 引擎**禁止**写入任何 `_index.md` 节点到 `knowledge_nodes` 表。
-   MCP `explore_node` 检测到目录或 `_index.md` 请求时，实时查询 `categories` 表 + 直属 `knowledge_nodes`，动态组装 Markdown 内容返回。
-   必须建立覆盖索引 `(kb_id, parent_virtual_path, sort_order, title, level)` 保障查询性能。
-   必须实现进程内缓存（TTL≤30s），构建完成时主动失效受影响目录的缓存。
-   L0 知识库首页采用相同机制。

---

## 6. 任务队列与调度规范

### 6.1 构建触发与扫描
-   支持目录级（递归）和文件级两种触发粒度。仓库级构建视为根目录触发。
-   扫描阶段必须前置执行 `repo_mounts` 匹配，未挂载文件直接跳过。
-   已初始化节点仅在检测到 Git Diff 时加入构建清单；未初始化节点始终加入。
-   扫描产物为去重后的 Build Manifest，作为队列唯一输入源。

### 6.2 队列存储与运行时规范
-   **禁止使用 MySQL 作为运行时任务队列**。MySQL 仅用于任务持久化、历史查询和崩溃恢复。
-   运行时调度必须基于 Go 进程内数据结构（Map + Channel），所有入队、去重、暂停、分发、删除操作在内存中完成。
-   入队后异步落盘 MySQL，不阻塞 API 响应。
-   服务启动时从 MySQL 加载 `status=pending` 的任务恢复到内存队列，`processing` 状态的任务重置为 `pending`。

### 6.3 单库串行与并发控制
-   **单库串行**：同一知识库同时仅允许一个构建 Session。后续触发合并入现有队列，不创建新 Session。
-   **文件并发**：Session 内通过可配置 Worker Pool 并发处理文件，默认并发数 3。
-   **幂等去重**：队列以 `git_physical_path` 为唯一键。待处理文件替换；处理中文件忽略；已完成文件正常入队。

### 6.4 检查点暂停恢复规范
-   暂停信号不影响正在处理的任务。Worker 完成当前文件后，检查暂停标志，若为 true 则停止拉取新任务。
-   恢复后 Worker 立即从队列头部继续消费。
-   Web 端提供队列管理页：列表展示、手动删除、SSE 实时推送。

### 6.5 配置项
```yaml
builder:
  concurrency_per_kb: 3      # 单知识库并发 Worker 数
  max_queue_size: 1000       # 单知识库队列上限
  scan_timeout: 30s          # 目录扫描超时
  llm_retry_max: 2           # LLM 解析失败重试次数
  pause_poll_interval: 2s    # 暂停状态下 Worker 轮询间隔
```

---

## 7. MCP Server 规范

### 7.1 鉴权
-   连接时通过 Header/Env 传入 `X-Tenant-Slug` + `API-Key`。
-   中间件解析出 `current_tenant_id`，注入 Context。

### 7.2 Tools 定义

#### `list_knowledge_bases`
-   **描述**: 获取当前用户有权访问的知识库列表。
-   **逻辑**: 查询 `owned OR shared OR public` 的 KB。
-   **返回**: `[{kb_slug, name, description, access_type}]`

#### `explore_node`
-   **参数**: `kb_slug` (string), `node_path` (string)
-   **路径解析**:
    ```go
    if !strings.HasSuffix(nodePath, ".md") {
        nodePath = filepath.Join(nodePath, "_index.md")
    }
    virtualPath := fmt.Sprintf("/%s/%s", kbSlug, strings.Trim(nodePath, "/"))
    ```
-   **权限**: 校验 KB 归属/授权/Public。
-   **响应**:
    ```json
    {
      "title": "JWT 认证指南",
      "level": "L2",
      "content": "...(含动态拼接的修订附录)...",
      "children": ["jwt/token-verify.l3.md", "jwt/refresh.l3.md"],
      "internal_links": [{"text":"SSO配置", "path":"sso-config.md"}]
    }
    ```
-   **修订附录拼接**: 若 `node_revision_logs` 有未隐藏记录，在 `content` 末尾动态追加 Markdown 表格格式的变更记录。

---

## 8. Web 管理端功能清单

| 模块 | 核心功能 | 技术要点 |
| :--- | :--- | :--- |
| **分类管理** | L1 树形拖拽排序、嵌套编辑、简介修改 | Vue Draggable, PUT /categories/sort |
| **Git 挂载** | 仓库绑定、目录浏览、Strip Prefix 设置、冲突检测 | Tree Select, 实时路径预览 |
| **构建监控** | 任务列表、进度条、日志查看、手动触发、暂停/恢复 | SSE 推送, 检查点状态展示 |
| **队列管理** | 待处理文件列表、手动删除、优先级查看 | SSE 实时刷新, 批量操作 |
| **变更记录** | 按文件分组展示修订日志、隐藏/删除错误条目、Diff 预览 | 批量筛选, go-git Diff 渲染 |
| **授权管理** | 分享 KB 给其他租户、设置过期时间、撤销授权 | Tenant Selector, DatePicker |

---

## 9. LLM Prompt 规范 (parse-progressive.yaml)

系统内置以下 Prompt 模板用于文档解析，**严禁修改字段名与结构**：

```yaml
system: |
  角色与目标
  你是一个专业的文档解析与知识结构化引擎。你的任务是对输入的文档进行深度分析，提取核心信息，并严格按照【渐进式披露架构】输出标准JSON格式数据。
  
  🚨 强制输出约束（最高优先级）
  - 仅输出合法 JSON：严禁包含任何 Markdown 标记、解释性文字。输出首字符必须是 `{`，末字符必须是 `}`。
  - 🚫 绝对禁止键名重复：严格遵循 RFC 8259 JSON 规范。同类多项内容必须使用数组 `[]` 包裹。
  - 字段名严格匹配：所有字段名必须与下方模板完全一致。
  - 转义规范：字符串内换行使用 `\n`，双引号转义为 `\"`。
  - 空值处理：若某字段无内容，返回空字符串 `""` 或空数组 `[]`，绝不可省略字段。
  
  {{.CategoriesContext}}
  
  🔄 渐进式披露架构解析规则
  【全局索引层】
  - L0_panoramic_summary: 分类定义描述（20-50字）
  
  【文档元数据层】
  - L1_category_title: 文档标题
  - L1_category_description: 文档功能定位（30-60字）
  - L1_metadata: 文档级元数据对象（created_time, creator, version, tags等，无则返{}）
  
  【文档结构层】
  - L2_knowledge_directory: 数组，每个元素包含：
    - L2_section_title: 章节标题
    - L2_section_summary: 章节详细介绍（100-200字）
    - L2_section_metadata: 章节级元数据（可选，无则返{}）
    - L3_knowledge_points: 数组，嵌套知识点
  
  【知识单元层】
  - L3_point_title: 知识点标题
  - L3_point_summary: 知识点摘要（50-100字）
  - L3_full_content: 核心叙述（⚠️必须剔除已抽离至L4的代码/表格/配置）
  - L4_detail_items: 数组，嵌套原子细节
  
  【原子细节层】
  - L4_detail_title: 细节项标题
  - L4_detail_content: 完整原始内容（代码/表格/配置/SOP）
  
  ⚠️ L3与L4抽离边界：L3聚焦"为什么/是什么"，L4聚焦"怎么做/具体参数"。两者互斥。
  ✅ 完整性校验：L1+L2+L3+L4 合并必须无损还原原文。
  
  📐 输入文档内容
  {{.DocumentContent}}
  
  🎯 执行指令
  直接输出符合上述模板的合法 JSON 数据，无任何额外内容。

response_format:
  type: object
  properties:
    category: { type: string }
    L0_panoramic_summary: { type: string }
    L1_category_title: { type: string }
    L1_category_description: { type: string }
    L1_metadata: { type: object }
    L2_knowledge_directory:
      type: array
      items:
        type: object
        properties:
          L2_section_title: { type: string }
          L2_section_summary: { type: string }
          L2_section_metadata: { type: object }
          L3_knowledge_points:
            type: array
            items:
              type: object
              properties:
                L3_point_title: { type: string }
                L3_point_summary: { type: string }
                L3_full_content: { type: string }
                L4_detail_items:
                  type: array
                  items:
                    type: object
                    properties:
                      L4_detail_title: { type: string }
                      L4_detail_content: { type: string }
  required: [category, L0_panoramic_summary, L1_category_title, L1_category_description, L1_metadata, L2_knowledge_directory]
```

---

## 10. AI 开发调度计划 (Phased Execution)

请严格按照以下阶段向 AI 编程工具下发指令，每阶段完成后进行验证再进入下一阶段。

### Phase 1: 基础设施与数据层
> **Prompt**: "根据 PROJECT_SPEC.md 第1、4章，初始化 Go+Gin+GORM 项目和 Vue3+Vite 项目。创建所有 MySQL 表结构对应的 GORM Model 和 Migration 文件。实现基础的数据库连接和多租户中间件骨架。"

### Phase 2: 分类与挂载管理 (Web)
> **Prompt**: "根据 PROJECT_SPEC.md 第3、8章，实现 categories 和 repo_mounts 的 CRUD API。在 Vue3 中实现 L1 分类树拖拽组件和 Git 挂载配置表单。确保拖拽排序能正确更新 sort_order。"

### Phase 3: 构建引擎与队列核心
> **Prompt**: "根据 PROJECT_SPEC.md 第5、6章，实现 Builder 引擎和内存队列。包括：Go 内存队列(Map+Channel)+MySQL持久化双缓冲、go-git 同步、Mount 路径解析、两阶段 LLM 调用封装、L2~L5 节点生成与写入、Git 版本追溯、检查点暂停恢复。集成 parse-progressive.yaml 作为 Prompt 模板。"

### Phase 4: MCP Server 实现
> **Prompt**: "根据 PROJECT_SPEC.md 第7章，实现 MCP Server。包括：鉴权中间件、list_knowledge_bases 和 explore_node Tool、路径解析器、_index.md 动态生成与缓存、修订日志动态拼接。使用 mark3labs/mcp-go 库。"

### Phase 5: Web 管理端完善与联调
> **Prompt**: "根据 PROJECT_SPEC.md 第8章，实现构建监控页、队列管理页、变更记录治理页。完善 SSE 实时推送。编写单元测试覆盖路径解析、队列去重、权限校验和 _index.md 动态生成逻辑。"

---

## 11. 关键约束清单 (AI 必读)

1.  **JSON Only**: LLM 调用必须强制 JSON Mode，解析失败需有重试机制。
2.  **No Tenant in Path**: 虚拟路径严禁包含租户标识，权限全靠 DB 校验。
3.  **Filesystem Semantics**: MCP 必须模拟文件系统，目录自动补 `_index.md`。
4.  **Zero Link Rewrite**: 不修改 Markdown 原文中的相对链接。
5.  **Dynamic Index**: `_index.md` 为运行时动态视图，禁止写入 DB。
6.  **Manual Trigger Only**: 无自动构建，全量重构，禁止内容编辑。
7.  **Checkpoint Pause**: 暂停不中断当前任务，完成后停止拉取。
8.  **Memory Queue**: 运行时调度走内存，MySQL 仅做持久化。
9.  **Version Tracking**: 节点记录 `git_commit_hash`，Diff 仅用于变更摘要。
10. **Revision Log Independent**: 修订日志独立表存储，支持人工隐藏/删除。
11. **L3/L4 Naming**: 拆解文件必须使用 `.l3.md` / `.l4.md` / `.l5.md` 后缀。
12. **Sort Rules**: L1=Web拖拽, L2=更新时间, L3=原文顺序。
13. **Parse/Mount Decoupled**: 路径解析与内容解析严格分离，互不依赖。
