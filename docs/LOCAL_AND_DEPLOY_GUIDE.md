# 本地运行与云端部署指南

## 项目概述

本项目是基于 **Cloudflare Workers** 的 AI 资讯聚合平台，核心技术栈：

- **运行时**：Cloudflare Workers（Serverless）
- **存储**：Cloudflare KV（键值存储）
- **AI 模型**：Google Gemini / OpenAI 兼容 API
- **数据源**：Folo 订阅源、GitHub Trending 等
- **发布**：GitHub Pages（通过 mdbook 构建静态站点）
- **CLI 工具**：Wrangler（Cloudflare 官方开发部署工具）

---

## 一、本地运行

### 1. 环境准备

- **Node.js**：>= 16.x（推荐 18+）
- **npm**：随 Node.js 安装
- **Wrangler CLI**：

```bash
npm install -g wrangler
```

### 2. 克隆项目

```bash
git clone https://github.com/Fluchw/CloudFlare-AI-Insight-Daily.git
cd CloudFlare-AI-Insight-Daily
```

### 3. 登录 Cloudflare

本地开发需要连接 Cloudflare 账号（用于 KV 绑定等远程资源）：

```bash
wrangler login
```

浏览器会弹出授权页面，完成授权即可。

### 4. 创建 KV 命名空间

在 Cloudflare 控制台 → `Workers 和 Pages` → `KV` 中创建命名空间，或通过命令行：

```bash
wrangler kv namespace create "NEWS_DATA_KV"
```

记录返回的 `id`，填入 `wrangler.toml`：

```toml
kv_namespaces = [
  { binding = "NEWS_DATA_KV", id = "你的KV命名空间ID" }
]
```

### 5. 配置环境变量

编辑 `wrangler.toml` 中的 `[vars]` 部分，必填项：


| 变量                   | 说明                                                                         |
| ---------------------- | ---------------------------------------------------------------------------- |
| `USE_MODEL_PLATFORM`   | AI 平台选择：`GEMINI` 或 `OPEN`                                              |
| `GEMINI_API_KEY`       | Gemini API 密钥（[获取地址](https://ai.google.dev/gemini-api/docs/api-key)） |
| `GEMINI_API_URL`       | Gemini API 地址（默认`https://generativelanguage.googleapis.com`）           |
| `DEFAULT_GEMINI_MODEL` | Gemini 模型名称                                                              |
| `GITHUB_TOKEN`         | GitHub Personal Access Token（需`repo` 权限）                                |
| `GITHUB_REPO_OWNER`    | GitHub 用户名                                                                |
| `GITHUB_REPO_NAME`     | GitHub 仓库名                                                                |
| `GITHUB_BRANCH`        | 目标分支                                                                     |
| `LOGIN_USERNAME`       | 后台登录用户名                                                               |
| `LOGIN_PASSWORD`       | 后台登录密码                                                                 |

数据源相关配置（按需）：


| 变量                      | 说明                       |
| ------------------------- | -------------------------- |
| `FOLO_COOKIE_KV_KEY`      | Folo Cookie 在 KV 中的键名 |
| `FOLO_DATA_API`           | Folo 数据 API 地址         |
| `NEWS_AGGREGATOR_LIST_ID` | 新闻聚合列表 ID            |
| `TWITTER_LIST_ID`         | Twitter 列表 ID            |
| `REDDIT_LIST_ID`          | Reddit 列表 ID             |
| `HGPAPERS_LIST_ID`        | HuggingFace 论文列表 ID    |

### 6. 启动本地开发服务

```bash
wrangler dev
```

默认启动在 `http://localhost:8787`，访问后台页面：

```
http://localhost:8787/getContentHtml
```

使用 `wrangler.toml` 中配置的用户名密码登录。

### 7. 主要路由说明


| 路由                  | 方法     | 说明                     |
| --------------------- | -------- | ------------------------ |
| `/login`              | GET/POST | 登录页面                 |
| `/getContentHtml`     | GET      | 后台管理页面（需登录）   |
| `/getContent`         | GET      | 获取原始数据（无需登录） |
| `/writeData`          | POST     | 写入数据到 KV            |
| `/genAIContent`       | POST     | AI 生成内容摘要          |
| `/genAIPodcastScript` | POST     | AI 生成播客脚本          |
| `/genAIDailyAnalysis` | POST     | AI 生成日报分析          |
| `/genAIDailyPage`     | GET      | 生成日报页面             |
| `/commitToGitHub`     | POST     | 提交内容到 GitHub        |
| `/rss`                | GET      | RSS 订阅源               |

---

## 二、部署到 Cloudflare Workers

> **重要说明**：部署到 Workers 后是一个 **HTTP API 服务**（而非定时任务）。Worker 只有 `fetch` 处理器，所有功能通过 HTTP 请求触发。你可以随时手动调用任意 API 端点，也可以通过外部调度器（GitHub Actions cron / Docker cron）定时调用来实现自动化。

### 1. 确认配置

确保 `wrangler.toml` 中所有必填变量已正确配置，KV 命名空间 ID 已填写。

### 2. 一键部署

```bash
wrangler deploy
```

部署成功后会返回一个 `*.workers.dev` 域名，即为线上服务地址。所有路由均可通过该域名直接调用，例如：

```
https://ai-daily.你的子域.workers.dev/getContentHtml
```

### 3. 绑定自定义域名（可选）

在 Cloudflare 控制台 → `Workers 和 Pages` → 你的 Worker → `设置` → `触发器` 中添加自定义域名。

---

## 三、自动化日报站点部署

### 方案一：GitHub Actions（推荐，零成本）

利用 GitHub Actions 定时触发，自动拉取当日数据并构建 mdbook 静态站点发布到 GitHub Pages。

**步骤：**

1. 在仓库 `Settings` → `Pages` 中，将 Source 设为 `GitHub Actions`
2. 编辑 `.github/workflows/build-daily-book.yml`：
   - 修改 `ref: 'book'` 为你的目标分支
   - 调整 `cron` 表达式设定执行时间（默认 UTC 23:00 = 北京时间 07:00）
3. 在仓库 `Settings` → `Secrets and variables` → `Actions` → `Variables` 中配置：
   - `WRITE_RSS_URL`：后端写入 RSS 数据的 URL
   - `RSS_FEED_URL`：RSS 订阅源 URL
   - `IMAGE_PROXY_URL`（可选）：图片代理地址
4. 编辑 `book.toml`，修改 `title` 和 `git-repository-url`
5. 手动触发一次 workflow 验证

访问地址：`https://<用户名>.github.io/<仓库名>/today/book/`

### 方案二：Docker 自建部署

适合有自己服务器的用户，通过 Docker 容器运行 cron 定时任务 + mdbook 静态站点。

**步骤：**

1. 编辑 `cron-docker/Dockerfile`，修改环境变量：

   ```dockerfile
   ENV OWNER="你的GitHub用户名"
   ENV REPO_NAME="你的仓库名"
   ENV GITHUB_TOKEN="你的GitHub Token"
   ENV IMG_PROXY_URL="图片代理地址"
   ```
2. 编辑 `cron-docker/scripts/build.sh` 和 `cron-docker/scripts/work/github.sh`，修改分支名
3. 编辑 `cron-docker/scripts/work/book.toml`，修改标题和仓库地址
4. 构建并运行：

   ```bash
   cd cron-docker
   docker build -t ai-daily-cron-job .
   docker run -d --name ai-daily-cron -p 4399:4399 --restart always ai-daily-cron-job
   ```
5. 访问 `http://localhost:4399` 验证

容器启动后会立即执行一次构建，之后每天 UTC 08:00（北京时间 16:00）自动执行。

---

## 四、架构总览

```
用户浏览器
    │
    ▼
Cloudflare Workers (src/index.js)
    │
    ├── 数据采集 (src/dataSources/)
    │     ├── Folo 订阅源 (newsAggregator, twitter, reddit, papers)
    │     └── GitHub Trending
    │
    ├── AI 处理 (src/chatapi.js)
    │     ├── Google Gemini
    │     └── OpenAI 兼容 API
    │
    ├── 存储 (Cloudflare KV)
    │     └── 缓存数据、配置、日报内容
    │
    └── 发布 (src/handlers/commitToGitHub.js)
          └── GitHub API → GitHub Pages / mdbook 站点
```

---

## 五、常见问题

**Q: `wrangler dev` 报错缺少环境变量？**
A: 确保 `wrangler.toml` 中所有必填的 `[vars]` 和 `kv_namespaces` 已正确配置。

**Q: 本地开发时 KV 数据如何处理？**
A: `wrangler dev` 默认连接远程 KV。如需本地持久化，可使用 `wrangler dev --persist` 参数。

**Q: 图片无法显示？**
A: 配置 `IMG_PROXY` 环境变量为图片代理地址，或在 GitHub Actions 中设置 `IMAGE_PROXY_URL` 变量。

**Q: 如何更换 AI 模型？**
A: 修改 `USE_MODEL_PLATFORM` 为 `OPEN`，然后配置 `OPENAI_API_KEY`、`OPENAI_API_URL`、`DEFAULT_OPEN_MODEL`，支持 DeepSeek 等 OpenAI 兼容 API。

**Q: 如何添加新的数据源？**
A: 参考 [项目拓展性指南](EXTENDING.md)，在 `src/dataSources/` 下创建数据源文件，然后在 `src/dataFetchers.js` 中注册。
