# AGENTS.md — sre-doc 项目指南

> 本文件在每次会话启动时自动加载，为 AI Agent 提供项目上下文和操作规范。

## 项目概述

sre-doc 是一个纯静态个人技术知识库站点。核心流程：

1. 在 `MarkDown/` 目录撰写 Markdown 笔记
2. 将 Markdown 手工转换为 HTML，放入 `posts/` 目录
3. 在 `index.html` 首页添加文章卡片入口
4. 推送到 GitHub `main` 分支
5. GitHub Workflow 自动触发 Cloudflare Pages 部署

**站点地址**：通过 Cloudflare Pages 托管，项目名 `sre-note`。

---

## 目录结构

```
sre-doc/
├── MarkDown/            # Markdown 源文件（笔记原文）
│   ├── MTU.md           # 示例：网络类笔记
│   └── secret.md        # 示例：Kubernetes 类笔记
├── posts/               # 生成的 HTML 文章页面
│   ├── template.html    # 文章 HTML 模板（必须基于此模板生成）
│   ├── admission-webhook.html
│   └── configmap-secret.html
├── css/
│   └── style.css        # 全站样式
├── index.html           # 首页（文章列表 + 分类导航）
├── .github/
│   └── workflows/
│       └── static.yml   # GitHub Actions → Cloudflare Pages 部署
└── AGENTS.md            # 本文件
```

---

## 分类体系

站点使用固定分类，每篇文章必须归属到以下分类之一（或多个）：

| 分类 ID     | 显示名称      | CSS 变量         | 标签 CSS class  |
|------------|-------------|-----------------|----------------|
| kubernetes | Kubernetes  | --tag-k8s       | tag-k8s        |
| go         | Go          | --tag-go        | tag-go         |
| python     | Python      | --tag-python    | tag-python     |
| devops     | DevOps      | --tag-devops    | tag-devops     |
| linux      | Linux       | --tag-linux     | tag-linux      |
| network    | Network     | --tag-network   | tag-network    |

### 自动分类规则

根据 Markdown 内容自动判断主分类和标签：

| 关键词 / 主题                                              | 分类       |
|--------------------------------------------------------|-----------|
| kubernetes, k8s, pod, deployment, service, ingress, configmap, secret, helm, admission, webhook, etcd, coredns, crd, operator | kubernetes |
| golang, go, goroutine, channel, slice, map, interface, go mod, go test, reflect, embed | go         |
| python, flask, django, pip, venv, pytest, asyncio, decorator, type hint | python     |
| devops, ci/cd, jenkins, gitlab, argocd, terraform, ansible, docker, containerd, prometheus, grafana | devops     |
| linux, shell, bash, systemd, kernel, iptables, selinux, strace, ebpf, namespace, cgroup | linux      |
| tcp, udp, mtu, mss, bgp, ospf, dns, http, tls, vlan, vxlan, 链路层, ip包, 路由 | network    |

**判断逻辑**：
1. 扫描 Markdown 全文，统计各分类关键词命中次数
2. 命中最多的分类为主分类
3. 命中次数 ≥ 主分类 50% 的其他分类作为附加标签
4. 如果无任何命中，根据文章标题和一级标题人工判断

---

## 文章命名规则

从 Markdown 内容自动生成 HTML 文件名和显示标题：

| 来源           | 规则                                                        |
|--------------|-----------------------------------------------------------|
| 文件名          | 取 Markdown 一级标题（`# 标题`），转小写，空格替换为 `-`，中文保留       |
| HTML 文件名     | `{slug}.html`，slug 同上                                    |
| HTML `<title>` | `{显示标题} - Tech Notes`                                   |
| 显示标题         | 直接使用 Markdown 一级标题原文                                    |

示例：
- `# Admission Webhook 开发` → 文件名 `admission-webhook-开发.html`，标题 `Admission Webhook 开发 - Tech Notes`
- `# ConfigMap 与 Secret` → 文件名 `configmap-与-secret.html`，标题 `ConfigMap 与 Secret - Tech Notes`
- `# IP包结构与MTU位置` → 文件名 `ip包结构与mtu位置.html`，标题 `IP包结构与MTU位置 - Tech Notes`

---

## 新增文章完整流程

当用户提供一篇新的 Markdown 笔记时，按以下步骤执行：

### Step 1: 接收与分类

1. 读取 Markdown 全文内容
2. 提取一级标题作为文章标题
3. 按自动分类规则确定主分类和附加标签
4. 生成 HTML 文件名（slug）

### Step 2: Markdown → HTML 转换

1. 以 `posts/template.html` 为基础模板
2. 修改模板中的以下占位内容：

| 模板占位             | 替换为                                        |
|------------------|---------------------------------------------|
| `文章标题`           | Markdown 一级标题                              |
| `分类名` (breadcrumb) | 主分类显示名称（如 Kubernetes）                    |
| `../index.html#kubernetes` (breadcrumb 链接) | `../index.html#{分类ID}`           |
| `../index.html#kubernetes` (sidebar active) | 当前分类设为 `class="nav-item active"`，其余为 `class="nav-item"` |
| `<span class="tag tag-k8s">Kubernetes</span>` | 按分类添加所有标签                                |
| `2026-05-12`     | 当前日期（格式 YYYY-MM-DD）                       |
| 文章正文区域           | Markdown 内容转 HTML                          |

3. **Markdown → HTML 转换规则**：

| Markdown 元素                    | HTML 对应                                                    |
|--------------------------------|-------------------------------------------------------------|
| `## 标题`                       | `<h2>标题</h2>`（自动编号：在 h2 前加序号，如 `1. 标题`）            |
| `### 标题`                      | `<h3>标题</h3>`                                              |
| `**粗体**`                      | `<strong>粗体</strong>`                                      |
| `` `行内代码` ``                 | `<code>行内代码</code>`                                       |
| 代码块 ` ```lang `               | `<pre><code><span class="code-label">语言</span>代码</code></pre>` |
| 表格                             | `<table><thead>...<tbody>...`                               |
| 提示/注意框                       | `<div class="highlight-box"><p>...</p></div>`               |
| 信息卡片                         | `<div class="card">...</div>`                                |
| 流程图/ASCII 图                  | `<div class="flow-diagram">...</div>`                        |
| 步骤流                           | `<div class="flow-steps"><div class="flow-step">...</div></div>` |
| 普通段落                         | `<p>...</p>`                                                |
| 无序列表                          | `<ul><li>...</li></ul>`                                     |
| 有序列表                          | `<ol><li>...</li></ol>`                                     |
| `---` 分隔线                    | 忽略或转为 `<hr>`                                              |

4. sidebar 导航中当前分类的文章计数 +1
5. TOC（目录）由模板内 JS 自动生成，无需手动编写

### Step 3: 更新首页

编辑 `index.html`，在对应分类的 `<div class="post-cards" data-category="{分类ID}">` 内添加文章卡片：

```html
<a href="posts/{slug}.html" class="post-card" data-title="{文章标题}" data-date="{YYYY-MM-DD}" data-tags="{主分类id},{附加标签id}">
  <div class="post-card-title">{文章标题}</div>
  <div class="post-card-desc">{从文章内容提取的简短描述，≤80字}</div>
  <div class="post-card-meta">
    <span>{标签 HTML}</span>
    <span>{日期}</span>
  </div>
</a>
```

同时更新 sidebar 中对应分类的 `<span class="count">` 计数 +1。

如果分类下原来是"暂无笔记"的空状态（`<div class="empty-state">`），则替换为文章卡片。

### Step 4: 更新搜索索引

编辑 `search-index.js`，在 `window.SEARCH_INDEX` 数组末尾追加一条记录：

```js
{
  title: "{文章标题}",
  desc: "{≤80字的描述，与首页卡片 desc 一致}",
  tags: ["{主分类id}", "{附加标签id}"],
  category: "{主分类id}",
  date: "{YYYY-MM-DD}",
  url: "posts/{slug}.html"
}
```

### Step 5: 保存 Markdown 源文件

将原始 Markdown 文件保存到 `MarkDown/` 目录，文件名与标题对应。

### Step 6: Git 推送

```bash
git add MarkDown/{slug}.md posts/{slug}.html index.html search-index.js
git commit -m "docs: add {文章标题} article"
git push origin main
```

推送后 GitHub Actions 自动触发 Cloudflare Pages 部署，无需手动操作。

---

## HTML 模板关键约束

1. **必须基于 `posts/template.html`** 生成新文章，不要从头编写 HTML 结构
2. sidebar 和 TOC 的 JS 脚本必须完整保留，不要修改
3. 所有 CSS class 名称必须与 `css/style.css` 中定义的一致
4. 文章内的代码块必须使用 `<span class="code-label">语言</span>` 标注语言
5. 链接路径：文章页引用首页用 `../index.html`，引用 CSS 用 `../css/style.css`
6. sidebar 中当前分类必须设为 `active`，其他分类不设

---

## 部署架构

```
git push → main 分支
    ↓
GitHub Actions (.github/workflows/static.yml)
    ↓
cloudflare/wrangler-action@v3
    ↓
Cloudflare Pages (project: sre-note, branch: main)
    ↓
线上站点更新
```

**关键配置**：
- 部署触发：push 到 `main` 或手动 `workflow_dispatch`
- Cloudflare secrets：`CLOUDFLARE_API_TOKEN`、`CLOUDFLARE_ACCOUNT_ID`
- 部署目录：项目根目录（`--project-name sre-note`）

---

## Git Commit 规范

| 前缀         | 用途              | 示例                                       |
|------------|-----------------|--------------------------------------------|
| `docs:`    | 新增/更新文章       | `docs: add IP包结构与MTU位置 article`         |
| `feat:`    | 新功能            | `feat: add search functionality`            |
| `fix:`     | 修复             | `fix: correct sidebar count on Go category` |
| `ci:`      | CI/CD 变更       | `ci: update Cloudflare deployment config`   |
| `refactor:`| 重构（非功能变更）    | `refactor: redesign blog card layout`       |
| `style:`   | 样式调整           | `style: adjust card hover effect`           |

---

## 禁止事项

- 不要修改 `.github/workflows/static.yml` 除非明确要求
- 不要删除 `MarkDown/` 中的源文件
- 不要修改 `css/style.css` 除非明确要求
- 不要在 HTML 中使用内联样式（除 template 中已有的布局辅助样式外）
- 不要跳过首页文章卡片的添加——每篇文章必须在首页有入口
- 不要修改 template.html 的 JS 脚本部分
- 提交时不要 force push，不要使用 `--no-verify`

---

## 快速参考

### 描述卡片（post-card-desc）提取规则

取文章第一段正文内容，截断至 ≤80 个中文字符，末尾加 `…`。不要使用标题作为描述。

### 多分类标签

一篇文章可以有多个分类标签，例如 Admission Webhook 同时涉及 Kubernetes + Python + DevOps，在标签区全部显示：

```html
<span class="tag tag-k8s">Kubernetes</span><span class="tag tag-python">Python</span>
```

主分类决定文章在首页的展示位置（category-group），附加标签只在文章卡片和文章页的 meta 区显示。
