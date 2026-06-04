---
name: writing-docs
description: Use when generating or updating project architectural documents in `docs/` from source code. Triggers on `/writing-docs` or when docs need to reflect codebase changes.
---

# Document

Generate or incrementally update project documents from the codebase into `docs/`.

## Decision Flow

```dot
digraph document_flow {
    "docs/ exists with docs?" [shape=diamond];
    "Full document generation" [shape=box];
    "Check git diff (last 3 commits)" [shape=box];
    "Meaningful changes?" [shape=diamond];
    "Delta update docs/" [shape=box];
    "Report: no changes" [shape=box];
    "Has frontend/UI code?" [shape=diamond];
    "Insert screenshot placeholders in UI docs" [shape=box];
    "Web App project?" [shape=diamond];
    "Auto-capture via take-screenshots skill" [shape=box];
    "Check staleness" [shape=diamond];
    "Ask user: refactor entire docs?" [shape=box];
    "Done" [shape=doublecircle];

    "docs/ exists with docs?" -> "Full document generation" [label="no or empty"];
    "docs/ exists with docs?" -> "Check git diff (last 3 commits)" [label="yes"];
    "Full document generation" -> "Has frontend/UI code?";
    "Check git diff (last 3 commits)" -> "Meaningful changes?";
    "Meaningful changes?" -> "Delta update docs/" [label="yes"];
    "Meaningful changes?" -> "Report: no changes" [label="no"];
    "Delta update docs/" -> "Has frontend/UI code?";
    "Has frontend/UI code?" -> "Insert screenshot placeholders in UI docs" [label="yes"];
    "Has frontend/UI code?" -> "Check staleness" [label="no"];
    "Insert screenshot placeholders in UI docs" -> "Web App project?";
    "Web App project?" -> "Auto-capture via take-screenshots skill" [label="yes"];
    "Web App project?" -> "Check staleness" [label="no"];
    "Auto-capture via take-screenshots skill" -> "Check staleness";
    "Report: no changes" -> "Check staleness";
    "Check staleness" -> "Ask user: refactor entire docs?" [label="most docs stale"];
    "Check staleness" -> "Done" [label="fresh enough"];
    "Ask user: refactor entire docs?" -> "Full document generation" [label="yes"];
    "Ask user: refactor entire docs?" -> "Done" [label="no"];
}
```

## Hard Rules (non-negotiable)

1. **AI-reimplementable fidelity.** Every doc must be detailed enough that another AI agent, given *only* the docs folder, can re-implement the entire project from scratch without reading the original source code.

2. **Output directory is `docs`.** All generated documents live under `docs/`.

3. **Chinese-only (简体中文).** All document filenames, directory names, and document content must use Simplified Chinese. No English names for files or folders.

4. **Numbered folder and file convention.** Every directory and file under `docs/` (except the root `README.md`) must follow the `{序号}_{中文名称}` format. Use zero-padded two-digit numbers (`01`, `02`, …). This applies to all levels: top-level category folders, sub-folders, and individual `.md` files.

5. **Delta-only when possible.** If `docs/` already contains detailed documentation, do NOT regenerate from scratch. Use git diff to detect changes and update only affected documents.

6. **Preserve manual edits.** When updating an existing doc file, preserve manually written sections. Only update content that corresponds to changed source code.

7. **No source code in docs.** Never include raw source code (no code blocks with implementation). Use mermaid diagrams for logic flow, and natural language for module/function descriptions. The doc describes *what* and *why*, never the literal *how* of the code.

7a. **Architecture and layout diagrams must use mermaid.** When representing system architecture, component layout, page structure, UI layout design, or any structural diagram, ALWAYS use mermaid diagrams (flowchart, graph, block diagram). NEVER use ASCII character drawings (`┌─┐└─┘│├┤`), plain-text boxes, or manual text-based layout diagrams. Mermaid ensures the diagrams are clean, maintainable, and consistently render across different viewers.

7b. **Mermaid node names with spaces MUST use quotes; inner quotes MUST be escaped.** When a mermaid node identifier contains spaces or special characters (e.g., Chinese characters, colons, parentheses), wrap the description text in double quotes inside square brackets. When the node text itself contains double-quote characters (e.g., Chinese quotation marks `""` around a term), those inner quotes MUST be HTML-escaped as `&quot;` within the outer quotes. Failing to quote names or escape inner quotes causes mermaid rendering errors.

```
✅ CORRECT:
    AuthCtx["AuthContext Provider"]
    Login["用户登录流程"]
    DB["Database (PostgreSQL)"]
    ModuleA["01_认证模块"]
    ExportBtn["点击&quot;导出Word&quot;按钮"]

❌ WRONG (will fail to render):
    AuthCtx[AuthContext Provider]
    Login[用户登录流程]
    DB[Database (PostgreSQL)]
    ExportBtn["点击"导出Word"按钮"]
```

This applies to ALL mermaid node shapes: `[]` (rectangle), `()` (rounded), `{}` (diamond), `[()]` (circle), `[//]` (parallelogram), etc. The quoting and escaping rules are the same for all shapes.

7c. **Subgraph headers must NOT use node shapes; define nodes with shapes inside the subgraph.** When using `subgraph` to group components, the subgraph header should only specify the ID and optional label using `subgraph ID [Label Text]` format. NEVER put node shape syntax (e.g., `[()]`, `[( )]`, `{ }`) directly in the subgraph header line. Instead, define the shaped node as a separate node inside the subgraph body, and connect related nodes with edges (`-->`).

```
✅ CORRECT:
    subgraph DB_Layer [Database Layer]
        DB[(PostgreSQL)]
        DB --> Table["journal_information表"]
    end

    subgraph Auth_Zone [Authentication]
        AuthSvc[Auth Service]
        AuthSvc --> TokenMgr["Token Manager"]
    end

❌ WRONG (shape in subgraph header — will render incorrectly):
    subgraph DB[(PostgreSQL)]
        Table["journal_information表"]
    end

    subgraph AuthSvc[Auth Service]
        TokenMgr["Token Manager"]
    end
```

This applies to ALL subgraph shapes: `subgraph ID[Label]` (rectangle), `subgraph ID(Label)` (rounded), `subgraph ID[(Label)]` (cylinder/database), `subgraph ID{Label}` (diamond), etc. Always use the clean `subgraph ID [Label]` header format and define shaped nodes inside.

8. **Modification history on every doc.** Every document under `docs/` must begin with a modification history table (see Modification History format below).

9. **Screenshot placeholders for frontend UI modules.** When documenting frontend/UI modules (pages, components, views with visual output), insert screenshot placeholders at key UI interaction points. Each placeholder MUST be followed by a markdown image link pointing to the `截图/` directory at the same directory level. If the project is a Web App, auto-capture screenshots using the `auto-capture-for-webapp:take-screenshots` skill after generating docs.

## Prerequisites

This skill depends on the **everything-claude-code** plugin:

```
/plugin marketplace add https://github.com/affaan-m/everything-claude-code
/plugin install everything-claude-code@everything-claude-code
```

## Modification History Format

Every document file under `docs/` must include a modification history table at the very beginning (after any YAML frontmatter, before the main heading). Use this exact format:

```markdown
## Modification History

| Time | Version | Description | Operator |
|------|---------|-------------|----------|
| 2026-05-20 10:30 | v1.0.0 | Initial documentation generated | ATreep |
| 2026-05-21 14:15 | v1.1.0 | Added error handling flow to data-pipeline module | ATreep |
```

Rules for the history table:
- **Time**: ISO date + time of the modification (`YYYY-MM-DD HH:MM`).
- **Version**: Semantic version (`MAJOR.MINOR.PATCH`). MAJOR for doc rewrites, MINOR for new sections/modules, PATCH for fixes and clarifications.
- **Description**: Brief one-line summary of what changed.
- **Operator**: Git username of the person who made the change. Obtain via `git config user.name` or `git log -1 --format='%an'`. If no git username is available, use the system username.
- On every delta update, append a new row to the table. Never remove or modify existing rows.

## Docs Directory Structure

The docs directory uses a numbered Chinese folder hierarchy, organized by domain category at the top level and by module at the detail level:

```
docs/
├── README.md                                       # 导航索引（不含序号前缀）
├── 01_产品需求文档/
│   └── 01_产品需求文档.md                            # PRD：系统概述、能力清单、用户故事、非功能需求
├── 02_核心业务流程/
│   └── 01_核心业务流程.md                            # 端到端业务流程与数据/控制流
├── 03_系统架构与开发/
│   ├── 01_系统架构概述.md                            # 系统结构、边界、技术栈、模块间交互
│   ├── 02_数据模型.md                                # 存储模式、实体、关系（mermaid ER 图）
│   └── 03_模块集成.md                                # 第三方 API/服务及交互契约
├── 04_运行时与部署/
│   ├── 01_运行时环境.md                              # 环境配置、启动脚本、执行模型
│   └── 02_部署与运维.md                              # 部署/运行手册、健康检查、故障/回滚路径
├── 05_实现指南/
│   ├── 01_端到端重建蓝图.md                           # 从零重建的完整实现指南
│   └── 02_模块目录.md                                # 模块目录及覆盖率矩阵
└── 06_模块详细设计/
    ├── 01_认证模块/                                   # 示例：认证模块
    │   ├── 01_模块概述.md                             # 模块职责、设计目标、已知缺陷
    │   ├── 02_登录流程.md                             # 登录子模块 doc
    │   ├── 03_令牌管理.md                             # 令牌子模块 doc
    │   ├── 04_会话处理.md                             # 会话子模块 doc
    │   └── 截图/                                      # 截图目录（与 doc 文件同级）
    │       ├── 1-login-page.png
    │       └── 2-login-error.png
    ├── 02_数据管道/
    │   ├── 01_模块概述.md
    │   ├── 02_数据摄取.md
    │   ├── 03_数据转换.md
    │   └── 04_数据导出.md
    └── 03_用户界面/
        ├── 01_模块概述.md
        ├── 02_仪表盘.md
        ├── 03_设置.md
        └── 截图/                                      # 用户界面模块的截图目录
            ├── 1-dashboard-overview.png
            └── 2-settings-page.png
```

### Folder and File Naming Rules

- **Top-level category folders**: `{序号}_{中文类别名}/` directly under `docs/`. Sequence determines the reading order. Categories should cover: product requirements, business flows, architecture, runtime/deployment, implementation guide, and module details.
- **Module folders**: `{序号}_{中文模块名}/` under `06_模块详细设计/`. Module name derived from domain responsibility (e.g., `01_认证模块`, `02_数据管道`).
- **Individual doc files**: `{序号}_{中文文件名}.md` inside their respective folders. Sequence by logical reading order.
- The root `README.md` is the only file without a numeric prefix — it serves as the navigation index.
- Each module folder MUST contain a `01_模块概述.md` (module overview, design objectives, known flaws).
- Sub-modules become additional numbered `.md` files in the same module folder.
- If a sub-module is complex enough, it can become a sub-folder (nested hierarchy uses the same `{序号}_{中文名}` convention).
- **No English names** for any file or directory under `docs/`.
- **Screenshot directory**: `截图/` placed at the same level as the `.md` files that reference them. Each module folder that has frontend UI content gets its own `截图/` subdirectory.

## Screenshot Placeholders for Frontend UI

### When to Insert Placeholders

Screenshot placeholders are required when documenting modules that have **visual user interfaces**. This includes:

- Web frontend pages (HTML, React, Vue, Angular components)
- Mobile app screens (SwiftUI, Jetpack Compose, Flutter widgets)
- Desktop app windows (Electron, Qt, WinForms)
- CLI/TUI interactive interfaces
- Any module under `06_模块详细设计/` that renders a visual output

Placeholders are NOT needed for:
- Backend API modules (no visual interface)
- Data processing pipelines (no UI)
- Library/utility modules (no UI)
- Configuration/infrastructure modules (no UI)

### Placeholder Format

Use this exact format — each placeholder MUST be followed by a markdown image link pointing to the `截图/` directory:

```
【图X：[所属功能模块] - [页面/组件名称] - [当前界面状态]，[简要描述页面整体布局，约30字]】

![图X](截图/X-descriptive-name.png)
```

**The description should be brief** — describe the overall page layout and module context at a high level. Do NOT enumerate individual UI widgets, color HEX codes, button labels, or pixel-level details. A reader should understand which page this is and its general structure, not reconstruct every pixel.

The description must answer:
1. **Which feature module** does this page belong to? (e.g., "听课记录模块", "用户设置")
2. **What page/component** is this? (e.g., "编辑页面", "登录弹窗")
3. **What state** is the interface in? (e.g., "默认状态", "空数据列表")
4. **What is the overall layout?** (e.g., "顶部导航+左侧菜单+主内容区三栏布局")

The image filename uses the pattern `X-descriptive-name.png` where:
- `X` is the sequential figure number (starting from 1, numbering within each `.md` file)
- `descriptive-name` is a short English slug describing the screenshot

Brief examples:

- `【图1：用户认证模块 - 登录页面 - 默认状态，页面居中展示Logo与登录表单，包含用户名、密码输入框及登录按钮】` followed by `![图1](截图/1-login-page.png)`

- `【图2：用户管理模块 - 设置页面 - 编辑状态，顶部标题栏+主体卡片式表单布局，包含头像、昵称、邮箱等设置项】` followed by `![图2](截图/2-settings-edit.png)`

- `【图3：问卷管理模块 - 列表页面 - 三条数据，顶部标题栏与筛选区+下方卡片列表布局，每张卡片展示标题、时间与状态】` followed by `![图3](截图/3-survey-list.png)`

### Screenshot Count Guidelines

Screenshot counts:

| UI Scope | Base Screenshot Count |
|----------|-----------------|
| Small (1-3 pages/views) | 3-6 screenshots |
| Medium (4-10 pages/views) | 6-15 screenshots |
| Large (11+ pages/views) | 15-25 screenshots |

Priority for screenshots in architecture docs:
1. Main page / entry view (logged-in and logged-out states)
2. Core feature pages (main interaction views)
3. Key interaction states (form validation, loading, empty, error states)
4. Complex dialogs or modals
5. Data visualization or result views
6. Navigation structure (menu, breadcrumb, tab switching)
7. Settings/configuration pages

### Auto-Capture with take-screenshots Skill

When the project is a **Web App** (detected by presence of `.html`, `.tsx`, `.jsx`, `.vue`, `.svelte` files, or a `package.json` with frontend framework dependencies), use the `auto-capture-for-webapp:take-screenshots` skill to automatically capture real screenshots for each placeholder.

**REQUIRED SKILL:** Use `auto-capture-for-webapp:take-screenshots` for screenshot automation. This skill handles:
- Starting the dev server and waiting for readiness
- Navigating to each page/route
- Capturing screenshots for each placeholder
- Saving screenshots to the correct `截图/` directories
- Graceful error handling and cleanup

If the `take-screenshots` skill is not available, or the project cannot be started, skip auto-capture and leave the placeholders for manual capture.

## Core Rules

- Prefer multiple focused files over a single large file.
- Cover **all code modules** in scope: `.py`, `.html`, `.js`, `.ts`, `.tsx`, `.jsx`, `.java`, `.sh`, `.go`, `.rs`, `.php`, `.rb`, `.cs`, `.kt`, `.swift`, `.sql`, `.yaml`, `.yml`.
- Documentation must be detailed enough that Claude Code can implement the full project from docs alone.
- Derive docs from source-of-truth files and code; avoid inventing behavior.
- Use `everything-claude-code:plan` to draft a plan before your actions.
- **No source code** — represent logic with mermaid diagrams (flowchart, sequence, class, state, ER) and describe behavior in natural language. Pseudocode is acceptable only for complex algorithms where natural language alone is ambiguous.

## Mode 1: Full Document Generation (no existing docs/)

Run when `docs/` does not exist or is empty.

### Workflow

1. **Inventory the codebase**
   - Identify project type(s), runtimes, entry points, module boundaries, and infra files.
   - Build a complete module index for all relevant source files in scope.
   - Map modules to a numbered sub-folder hierarchy under `docs/06_模块详细设计/`.
   - **Detect frontend/UI code**: Identify modules that contain visual user interfaces (see "When to Insert Placeholders" above).

2. **Map architecture and behavior**
   - Trace request/data/control flow across layers.
   - Capture dependencies, external services, config/env requirements, scripts/commands, and operational behavior.
   - Represent all flows as mermaid diagrams.
   - For frontend/UI modules, additionally map the visual layout: pages, components, their states, and user interaction flows.

3. **Generate `docs/` set**
   - `docs/README.md` — 导航索引
   - `docs/01_产品需求文档/01_产品需求文档.md` — 产品需求文档（详见下方 PRD 要求）
   - `docs/02_核心业务流程/01_核心业务流程.md` — 端到端业务流程与数据/控制流（mermaid 图）
   - `docs/03_系统架构与开发/01_系统架构概述.md` — 系统结构、边界、技术栈、模块间交互（mermaid 图）
   - `docs/03_系统架构与开发/02_数据模型.md` — 存储模式、实体、关系（mermaid ER 图）
   - `docs/03_系统架构与开发/03_模块集成.md` — 第三方 API/服务及交互契约
   - `docs/04_运行时与部署/01_运行时环境.md` — 环境配置、启动脚本、执行模型
   - `docs/04_运行时与部署/02_部署与运维.md` — 部署/运行手册、健康检查、故障/回滚路径
   - `docs/05_实现指南/01_端到端重建蓝图.md` — 从零重建的完整实现指南
   - `docs/05_实现指南/02_模块目录.md` — 模块目录及覆盖率矩阵
   - `docs/06_模块详细设计/` — 模块详细设计根目录
   - `docs/06_模块详细设计/{序号}_{模块名}/01_模块概述.md` — 每个模块的概述
   - `docs/06_模块详细设计/{序号}_{模块名}/{序号}_{子模块名}.md` — 每个子模块或功能一个文件

4. **Insert Screenshot Placeholders**
   - For every frontend/UI module doc file, insert screenshot placeholders at key UI interaction points (see Screenshot Placeholder Format).
   - Place placeholders after the natural language description of each page/view/component, before the next section.
   - Create the `截图/` subdirectory in each module folder that has frontend UI content.

5. **Auto-Capture Screenshots (Web App projects)**
   - If the project is a Web App, use the `auto-capture-for-webapp:take-screenshots` skill to capture real screenshots for each placeholder.
   - Invoke the skill with the project source path and the list of screenshot placeholders to capture.
   - The `take-screenshots` skill handles starting the dev server, navigating pages, capturing screenshots, and saving them to the correct `截图/` directories.
   - If the `take-screenshots` skill is not available or the project cannot be started, skip auto-capture. The placeholders remain in the docs for manual capture later.

6. **PRD Requirements**

   The `docs/01_产品需求文档/01_产品需求文档.md` must contain:

   - **System Overview**: What the system does, who uses it, and why it exists.
   - **Capabilities**: High-level list of everything the system can do, organized by module.
   - **Module Design Objectives**: For each module in `docs/06_模块详细设计/`, document:
     - What problem it solves.
     - What it was designed to achieve.
     - Known design flaws, limitations, or trade-offs.
   - **User Stories / Use Cases**: Key scenarios the system supports.
   - **Non-Functional Requirements**: Performance, security, scalability constraints.
   - **Out of Scope**: Explicitly list what the system does NOT do.

7. **Per-module documentation requirements**

   Each module sub-folder (`docs/06_模块详细设计/{序号}_{模块名}/`) must include:

   - `01_模块概述.md` with:
     - Module responsibility and scope.
     - Design objectives and known flaws.
     - Dependencies on other modules.
     - Modification history table.
   - One `.md` per sub-module or feature (e.g., `02_登录流程.md`, `03_令牌管理.md`), each containing:
     - File path(s) and responsibility.
     - Public interfaces (functions/classes/endpoints/CLI commands) — described in natural language, not code.
     - Inputs/outputs, side effects, and invariants.
     - Internal dependencies and call relationships (mermaid sequence/flowchart diagrams).
     - Error handling and edge cases.
     - Security considerations and validation boundaries.
     - Reimplementation notes (what must be preserved for parity).
     - Modification history table.
     - **For frontend/UI sub-modules**: Screenshot placeholders at key visual states.

8. **Logic Representation (mermaid)**

   Use mermaid diagrams to represent all logic, architecture, and layout structures. NEVER use ASCII character drawings (`┌─┐└─┘│├┤`) for any diagram — mermaid ensures clean, maintainable, and consistently rendered output.

   Required diagram types:

   | Logic Type | Mermaid Diagram |
   |------------|----------------|
   | Request/data flow | `flowchart TD` or `flowchart LR` |
   | Component interactions | `sequenceDiagram` |
   | Data entities & relationships | `erDiagram` |
   | State machines | `stateDiagram-v2` |
   | Class/module structure | `classDiagram` |
   | System boundaries / architecture | `flowchart` with subgraphs |
   | Page layout design / UI structure | `graph` or `flowchart` with subgraphs |

   Every module overview must include at least one flowchart showing the module's internal flow.

   **Mermaid quoting rules (CRITICAL):** When a node identifier contains spaces, Chinese characters, colons, parentheses, or any special characters, the display text MUST be wrapped in double quotes inside the brackets. When the node text itself contains double-quote characters (e.g., Chinese quotation marks `""` around a term), those inner quotes MUST be HTML-escaped as `&quot;` within the outer quotes. Failing to quote names or escape inner quotes causes mermaid rendering errors:
   ```
   ✅ CORRECT:
       AuthCtx["AuthContext Provider"]
       Login["用户登录流程"]
       DB["Database (PostgreSQL)"]
       ModuleA["01_认证模块"]
       ExportBtn["点击&quot;导出Word&quot;按钮"]

   ❌ WRONG (will cause rendering failure):
       AuthCtx[AuthContext Provider]
       Login[用户登录流程]
       DB[Database (PostgreSQL)]
       ExportBtn["点击"导出Word"按钮"]
   ```
   This rule applies to ALL node shapes: `[]` (rectangle), `()` (rounded corners), `{}` (diamond), `[()]` (circle), `[//]` (parallelogram), etc.

9. **Coverage validation**
   - Produce a coverage matrix in `docs/05_实现指南/02_模块目录.md` mapping every discovered source module to a documentation target folder.
   - Explicitly list any skipped/generated/vendor files and the reason.
   - If coverage is incomplete, continue until all in-scope modules are documented.

10. **Staleness and provenance**
    - Add generated markers and scan metadata (date, scope, files scanned).
    - Preserve manually written sections when updating existing docs.

11. **Final summary**
    - Report created/updated files in `docs`.
    - Report module coverage totals and any intentional exclusions.

## Mode 2: Delta Update (docs/ already exists)

Run when `docs/` exists with detailed documentation.

### Workflow

1. **Detect Changes (git diff, last 3 commits)**

   ```bash
   git diff HEAD~3..HEAD --name-status
   ```

   - Focus on source files only. Ignore non-source files (`.md`, `.gitignore`, lock files, config files that don't affect behavior).
   - If there are also uncommitted changes, include them: `git diff HEAD --name-status`.
   - Default depth is 3 commits. User can override by passing a different range.

   If no meaningful source changes are found, report this and stop.

2. **Map Changes to Doc Files**

   For each changed source file:
   - Look up the corresponding module folder in `docs/06_模块详细设计/{序号}_{模块名}/`.
   - Check whether the change affects other doc files (e.g., `03_系统架构与开发/`, `04_运行时与部署/`, `01_产品需求文档/`).
   - If a changed module has no corresponding docs folder yet, flag it as **new** — create the folder and `01_模块概述.md`.
   - If a docs folder exists but the source module was deleted, flag it for removal or archival.

3. **Classify Changes and Map to Screenshot Actions**

   For each changed source file, classify the change type and determine the corresponding screenshot action:

   | Change Type | Doc Action | Screenshot Impact |
   |------------|---------------|-------------------|
   | New UI page/component added | Add new sub-module doc | **新增** — Insert new screenshot placeholders |
   | Existing UI page modified (layout/visual changed) | Update doc sections + screenshots | **替换** — Update placeholder description, mark old screenshot for replacement |
   | Existing UI page modified (logic only, same visual) | Update doc text only | **保留** — Keep existing screenshot, no visual change |
   | UI page/component removed | Remove doc sections | **删除** — Remove placeholders and screenshot files |
   | New backend module (no UI) | Add new module docs | No screenshot needed |
   | Backend logic changed (no UI impact) | Update module docs | No screenshot needed |
   | UI redesign (structural) | Rewrite affected docs | **替换** — Replace ALL affected screenshots |
   | Config/deployment change | Update architecture/runtime docs | No screenshot needed |

   **CRITICAL: After classifying changes, update screenshot placeholders** (see Step 6 below) to reflect additions, replacements, or deletions.

4. **Read and Analyze Affected Docs**

   For each doc file that needs updating:
   - Read the current doc content.
   - Read the corresponding source code (current state).
   - Identify what has changed and which sections of the doc are now stale.

5. **Update Docs (delta only)**

   Apply targeted updates:
   - **Modified modules** — update relevant sections in the module's doc files under `06_模块详细设计/`.
   - **New modules** — create `docs/06_模块详细设计/{序号}_{模块名}/01_模块概述.md` and sub-module docs following per-module documentation requirements.
   - **Deleted modules** — mark archived or remove if appropriate.
   - **Cross-cutting changes** — update affected files in `01_产品需求文档/`, `02_核心业务流程/`, `03_系统架构与开发/`, `04_运行时与部署/`.
   - **Coverage matrix** — update `docs/05_实现指南/02_模块目录.md` if modules were added or removed.
   - **PRD** — update design objectives/flaws in `docs/01_产品需求文档/01_产品需求文档.md` if module behavior changed.
   - **Index** — update `docs/README.md` if new doc files/folders were added or removed.
   - **Modification history** — append a new row to the history table in every affected document. Use git username from `git config user.name`.

6. **Update Screenshot Placeholders (for frontend/UI changes)**

   When the changed source files include frontend/UI code, update screenshot placeholders based on the change classification from Step 3:

   - **New UI pages/views** (新增) → Insert new screenshot placeholders following the Screenshot Placeholder Format. Add them to the `截图/` directory at the module folder level.
   - **Modified UI** (替换) → Update existing placeholder descriptions to reflect the new UI state.
   - **Removed UI pages/views** (删除) → Remove their placeholders.
   - **Renumbering**: If ANY placeholder was added or deleted within a doc file, renumber ALL placeholders in that file from 图1 upward in ascending order. Do NOT keep gaps or skip numbers. If only descriptions were updated (same count, no additions/removals), keep the original numbering.
   - If a module that now has UI content didn't previously have a `截图/` directory, create it.

7. **Auto-Capture Updated Screenshots (Web App projects)**
   - If the project is a Web App and frontend/UI code was changed, use the `auto-capture-for-webapp:take-screenshots` skill to auto-capture updated screenshots.
   - Only capture screenshots for placeholders marked 新增 or 替换 (from Step 3). Skip 保留 placeholders and clean up 删除 files.
   - Invoke the skill with the project source path and the list of affected screenshot placeholders.
   - If the `take-screenshots` skill is not available or the project cannot be started, skip auto-capture.

8. **Final Summary**

   Report:
   - Source files detected as changed.
   - Doc files/folders updated/created/archived.
   - Modules skipped (non-source, vendor, generated) and why.
   - If no docs needed updating, say so explicitly.

## Staleness Check (both modes)

After completing either mode, compare doc freshness against the project:

1. Get the newest modification time among doc files: `find docs -name '*.md' -exec stat -f '%m' {} \; | sort -rn | head -1`
2. Get the newest modification time among source files (exclude `docs/`, `node_modules/`, `.git/`): `find . -name '*.ts' -o -name '*.py' -o -name '*.go' ... | xargs stat -f '%m' | sort -rn | head -1`
3. If most docs are significantly older than the newest source files (e.g., >7 days gap), ask the user:

   > Most docs haven't been updated in a while compared to recent source changes. Would you like me to refactor the entire docs from scratch?

   - If yes → switch to Mode 1 (full document generation).
   - If no → stop.

## Output Quality Bar

The documentation must provide enough architectural, interface, and behavioral detail for full-project reconstruction without reading the original code — whether generated fresh or updated incrementally. Logic must be expressed through mermaid diagrams and natural language descriptions, never raw source code. Architecture and layout diagrams must use mermaid — ASCII character drawings are strictly forbidden. All mermaid node names containing spaces or special characters must be wrapped in double quotes, and inner double-quote characters must be HTML-escaped as `&quot;`. Subgraph headers must use clean `subgraph ID [Label]` format without node shapes; define shaped nodes and their connections inside the subgraph body. Frontend UI documentation must include screenshot placeholders that identify the feature module, page/component name, UI state, and overall layout structure at a high level.
