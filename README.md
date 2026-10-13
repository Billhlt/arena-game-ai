# arena-game-ai（AI 闯关学习小程序）

一款基于 AI 的学习类微信小程序：输入任意想学的内容，AI 自动联网检索资料并生成一套闯关题目，用户边答边学，通关后获得一份 AI 生成的复盘报告。项目采用前后端分离架构，覆盖出题、答题、报告、用户系统、知识库（RAG）、AI 配图等完整业务闭环。

## 目录

- [项目介绍](#项目介绍)
- [核心功能](#核心功能)
- [技术栈](#技术栈)
- [整体架构](#整体架构)
- [目录结构](#目录结构)
- [快速开始](#快速开始)
- [环境变量配置](#环境变量配置)
- [API 概览](#api-概览)
- [测试](#测试)
- [部署](#部署)
- [文档索引](#文档索引)
- [许可证](#许可证)

## 项目介绍

「arena-game-ai」是一款 AI 闯关学习小程序，目标是把任意输入的学习内容，快速转化为一局趣味性的问答闯关游戏：

1. 用户在首页输入一句话、一段话或一个主题；
2. 后端通过 AI Agent 联网搜索补充最新知识，或基于用户上传的私有文档（RAG）检索资料；
3. AI 生成包含单选、多选、判断题的题库，每题附带详细讲解；
4. 用户逐题作答，提交后立即获得对错反馈与知识讲解；
5. 全部通关后，AI 生成包含掌握度、薄弱知识点、知识总结与学习建议的复盘报告。

该项目的业务骨架可复用到任何垂直领域，例如驾考宝典、面试题库、企业培训考核、职业资格考试复习等，替换知识源即可形成新产品。


## 运行截图
#### 1.初始界面

#### 2.答题界面
<img width="600" height="1226" alt="image" src="https://github.com/user-attachments/assets/2391f6a8-734c-404a-ac13-d701eb836237" />

#### 3.查看答题报告
<img width="1327" height="1345" alt="截图 2026-10-13 10-33-19" src="https://github.com/user-attachments/assets/83843292-600c-4347-8a82-5365a4280bf1" />

#### 4.个人中心

#### 5.个人知识库
<img width="1993" height="1350" alt="截图 2026-10-13 10-33-32" src="https://github.com/user-attachments/assets/37bbf9b8-f444-439b-996c-bc256070d942" />

## 核心功能

### 1. 一句话起题，AI 自动联网出题

大模型自身的知识存在时效性，因此出题环节接入 Tavily 联网搜索，使用 LangGraph 构建 ReAct Agent，让 AI 自主决定搜索什么、搜索几轮，从而保证题目内容不过时。

### 2. 闯关答题，即时判分与讲解

支持单选、多选、判断三种题型。每题提交后立即高亮正确答案、标记错误选项，并给出解析和对应知识点；答错同样提供讲解，并支持回看上一题。

### 3. AI 生成复盘报告

通关后生成结构化报告，包含掌握度评分、掌握较好/薄弱的知识点、三句话知识总结、后续学习建议以及一句可分享的学习金句，并结算本局获得的经验值（XP）。

### 4. 私有知识库出题（RAG）

用户可上传 PDF、Word、Markdown、纯文本文档。文档会被异步解析、分块、向量化后写入 Chroma 向量库（每个用户独立 collection），之后即可一键基于该文档生成闯关题，适合企业培训考核、特定资料复习等场景。检索环节使用 Agentic RAG：AI 自主决定查询知识库、联网补充或两者结合。

### 5. AI 题目配图

调用通义千问文生图模型按知识点为题目生成配图，并转存到腾讯云 COS 得到永久可访问链接。实现了每人每日生图额度、并发控制等成本管理，生图失败不会影响出题主流程。

### 6. 微信登录与历史回看

基于微信 `jscode2session` 静默登录 + JWT 鉴权。个人中心展示累计经验值、答题统计与闯关历史，点击任意记录可回看当时的复盘报告。项目采用「可选登录」设计，未登录也能完整体验出题、答题、看报告的核心流程。

### 7. 异步任务出题

出题链路支持异步任务模式（`POST /quiz/generate/async` + 轮询 `GET /quiz/task/{task_id}`），接口秒级返回任务 ID，AI 在后台生成，前端定时轮询，避免长时间阻塞请求。

## 技术栈

| 层面 | 技术选型 |
| --- | --- |
| 小程序前端 | Taro 4 · React 18 · TypeScript · Sass |
| 后端服务 | Python 3.11+ · FastAPI · Pydantic v2 · Uvicorn · asyncio |
| AI 编排 | LangChain · LangGraph（ReAct Agent）· langchain-openai |
| 大模型 | DeepSeek（`deepseek-chat`，出题 / 报告） |
| 联网搜索 | Tavily（`langchain-tavily`） |
| 向量检索 | Chroma（知识库 RAG）· 阿里云百炼 `text-embedding-v4` |
| 文生图 | 阿里云百炼 `qwen-image-2.0` |
| 对象存储 | 腾讯云 COS |
| 数据库 | MySQL 8.0（`aiomysql` 异步连接池，启动时自动建库建表） |
| 鉴权 | 微信 `jscode2session` 静默登录 + JWT（PyJWT） |
| 日志 | structlog |
| 测试 | pytest · pytest-asyncio · httpx |
| 部署 | Docker · 微信云托管 |

## 整体架构

```text
┌─────────────────────────────────────────────────────────┐
│                    微信小程序（Taro + React）             │
│   首页输入 / 闯关答题 / 复盘报告 / 个人中心 / 知识库        │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTP / JSON（统一 ApiResponse 结构）
┌──────────────────────────▼──────────────────────────────┐
│                  FastAPI 后端（/api/v1）                  │
│  routes: health · quiz · report · user · knowledge       │
│  services: 出题 / 报告 / 评分 / 登录 / 知识库 / 配图 / RAG │
└──┬─────────┬───────────────┬──────────────┬─────────────┘
   │         │               │              │
   ▼         ▼               ▼              ▼
 DeepSeek   Tavily        Chroma         MySQL / COS
 (大模型)   (联网搜索)    (向量库)      (业务数据 / 图片)
```

### 核心业务链路

```text
用户输入学习内容
   │
   ▼
敏感词校验 ──► 获取参考资料（指定 doc_id 走 RAG 知识库，否则联网搜索）
   │
   ▼
LangChain 出题链（DeepSeek 结构化输出题库 JSON）
   │
   ▼
（可选）按题为题目生成配图 ──► 转存 COS
   │
   ▼
用户逐题作答（前端本地即时判分 + 讲解）
   │
   ▼
全部完成后调用报告链 ──► 生成复盘报告 + 结算经验值
```

## 目录结构

```text
arena-game-ai/
├── backend/                    # Python FastAPI 后端
│   ├── app/
│   │   ├── api/v1/routes/      # 接口路由：health / quiz / report / user / knowledge
│   │   ├── core/               # 配置、数据库、鉴权、敏感词过滤、异常处理
│   │   ├── llm/                # LangChain 模型工厂、出题链、报告链
│   │   ├── models/             # Pydantic 数据模型
│   │   ├── prompts/            # 出题 / 报告 / 搜索 / RAG Prompt 模板
│   │   ├── repositories/       # 数据访问层（用户、题库、任务、知识库、生图日志）
│   │   ├── services/           # 业务服务层（见下）
│   │   ├── utils/              # 工具（ID 生成等）
│   │   └── main.py             # 应用入口
│   ├── scripts/init_mysql.py   # 手动初始化数据库脚本
│   ├── sql/init_mysql.sql      # 基础建表 SQL（仅 4 张表，勿用于建表，详见文档）
│   ├── tests/                  # pytest 单元 / 集成测试
│   ├── requirements.txt
│   └── Dockerfile              # 微信云托管 / Docker 部署镜像
├── frontend/                   # Taro 微信小程序前端
│   ├── src/
│   │   ├── pages/              # index(首页) / quiz(闯关) / report(报告) / profile(我的) / knowledge(知识库)
│   │   └── services/api.ts     # 统一 API 封装与类型定义
│   └── config/                 # Taro 环境配置（dev / prod，含后端地址）
├── docs/                       # 需求分析、方案设计、本地运行指南等文档
├── openspec/                   # OpenSpec 规范驱动开发文档
└── prototypes/                 # HTML 交互原型
```

主要服务模块（`backend/app/services/`）：

| 模块 | 职责 |
| --- | --- |
| `quiz_service` | 出题流程编排（同步 / 异步任务），含内容校验、资料获取、落库 |
| `report_service` | 复盘报告生成，答题记录落库、经验值结算 |
| `scoring_service` | 计算正确率、答对数等统计摘要 |
| `search_service` | Tavily 联网搜索 Agent（轻量搜索 / 深度搜索 / URL 提取） |
| `rag_service` | 知识库 Agentic RAG：知识库检索 + 可选联网补充 |
| `knowledge_service` | 知识库文档上传、解析、状态、删除 |
| `document_loader_service` | 解析 PDF / DOCX / Markdown / 文本并分块 |
| `vector_store_service` | Chroma 向量化写入与检索（按用户隔离） |
| `image_service` | AI 生图（并发、每日额度）并转存 COS |
| `cos_service` | 腾讯云 COS 上传封装 |
| `user_service` | 微信登录、用户档案、统计聚合 |
| `history_service` | 闯关历史列表与详情 |

## 快速开始

> 详细的保姆级步骤（含每个 API Key 的申请流程、环境变量逐项说明、11 个常见问题排查）请参见 [保姆级本地运行指南](./docs/保姆级本地运行指南.md)。以下是精简版。

### 前置条件

- Python >= 3.11
- Node.js >= 18（推荐 20+）
- MySQL >= 8.0
- 微信开发者工具
- DeepSeek API Key（必需，出题与报告）
- Tavily API Key（可选，联网搜索）
- 阿里云百炼 API Key（可选，知识库 RAG 与 AI 生图）
- 腾讯云 COS 存储桶（可选，存储配图）
- 微信小程序测试号（可选，微信登录）

### 1. 克隆项目

```bash
git clone https://github.com/Billhlt/arena-game-ai.git
cd arena-game-ai
```

### 2. 启动后端

```bash
cd backend

# 建议创建虚拟环境
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\Activate.ps1

# 安装依赖（国内可加 -i https://pypi.tuna.tsinghua.edu.cn/simple 加速）
pip3 install -r requirements.txt

# 配置环境变量（至少填写 DEEPSEEK_API_KEY 和 MYSQL_PASSWORD）
cp .env.example .env

# 启动服务
python3 -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

启动成功后访问 <http://localhost:8000/docs> 可看到 FastAPI 自动生成的接口文档。

> 数据库无需手动创建：`MYSQL_AUTO_INIT=true` 时启动会自动建库并创建 7 张表。也可以手动执行 `python3 scripts/init_mysql.py`。注意 `sql/init_mysql.sql` 只包含 4 张基础表，缺少 `quiz_tasks`、`kb_documents`、`image_generation_logs`，请不要用它建表。

### 3. 启动前端

**新开一个终端：**

```bash
cd frontend
npm install                       # 国内可加 --registry=https://registry.npmmirror.com
npm run dev:weapp                 # 编译到 dist/ 并进入监听模式
```

用微信开发者工具新建项目，**目录选择 `frontend/dist`**，AppID 填测试号 AppID。

> ⚠️ 本地开发必须在微信开发者工具的「详情 → 本地设置」中**勾选「不校验合法域名」**，否则请求 `localhost` 会被拦截。

生产构建（自动切换到线上后端地址）：

```bash
npm run build:weapp
```

### 4. 验证

在微信开发者工具模拟器中，输入任意想学的内容（例如「RAG 和传统搜索有什么区别」），点击生成题目即可体验完整闭环。出题过程涉及联网搜索与模型推理，通常需要数十秒，前端会定时轮询任务状态，属于正常现象。

## 环境变量配置

复制 `backend/.env.example` 为 `backend/.env` 后按需填写，关键项如下：

| 变量 | 必需 | 说明 |
| --- | --- | --- |
| `DEEPSEEK_API_KEY` | ✅ | AI 出题与报告生成 |
| `MYSQL_PASSWORD` | ✅ | 本机 MySQL 密码 |
| `TAVILY_API_KEY` | 🔧 | 联网搜索，不配置则自动降级为大模型自身知识 |
| `DASHSCOPE_API_KEY` | 🔧 | 知识库 Embedding 与 AI 生图 |
| `DASHSCOPE_IMAGE_API_KEY` | 🔧 | 生图专用 Key，留空复用 `DASHSCOPE_API_KEY` |
| `COS_*` | 🔧 | 腾讯云 COS（存储桶需为「公有读私有写」），配图功能需要 |
| `WECHAT_APP_ID` / `WECHAT_APP_SECRET` | 🔧 | 微信登录 |
| `JWT_SECRET` | 🔧 | 登录令牌签名密钥，**上线必须改为随机长字符串** |

> 完整变量清单见 `.env.example`。核心功能仅需 `DEEPSEEK_API_KEY` 与 MySQL；其余功能不配置只会对应不可用，不影响项目启动。

## API 概览

所有接口统一返回 `{ "code": 0, "message": "ok", "data": ... }` 结构（`code = 0` 表示成功）。

### 出题

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/v1/quiz/generate` | 同步生成题库 |
| POST | `/api/v1/quiz/generate/async` | 异步创建出题任务，返回 `task_id`（可选传 `doc_id` 基于知识库出题、`generate_images` 生成配图） |
| GET | `/api/v1/quiz/task/{task_id}` | 轮询任务状态（pending / running / completed / failed） |

### 报告与用户

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/v1/report/generate` | 生成复盘报告（提交题目与答题记录） |
| POST | `/api/v1/user/login` | 微信静默登录，换取 JWT |
| GET / PUT | `/api/v1/user/profile` | 获取 / 更新用户档案 |
| GET | `/api/v1/user/quizzes` | 闯关历史（分页） |
| GET | `/api/v1/user/quizzes/{quiz_id}` | 闯关详情（含题目、答题记录、报告） |

### 知识库

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/v1/knowledge/documents` | 上传文档（PDF / Word / Markdown / 文本） |
| GET | `/api/v1/knowledge/documents` | 文档列表 |
| GET | `/api/v1/knowledge/documents/{doc_id}` | 文档处理状态 |
| DELETE | `/api/v1/knowledge/documents/{doc_id}` | 删除文档（级联清理向量与文件） |

### 其他

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/v1/health` | 健康检查 |

## 测试

```bash
cd backend
pytest
```

测试通过 mock 隔离了大模型和第三方接口，不消耗 API 额度，也不依赖真实数据库。覆盖出题、报告、评分、搜索、向量检索、知识库、用户、鉴权、安全过滤等模块。

## 部署

`backend/Dockerfile` 提供了适配 **微信云托管** 的多阶段构建镜像（基于 Python 3.11 slim，默认监听 80 端口，含健康检查）。部署步骤：

1. 将后端镜像部署到微信云托管，容器端口与云托管控制台配置保持一致；
2. 密钥类环境变量（数据库密码、各类 API Key、微信 AppSecret、`JWT_SECRET` 等）通过平台环境变量注入，**不要提交到代码仓库**；
3. 前端执行 `npm run build:weapp` 后，将 `dist/` 上传为小程序体验版 / 正式版。

正式上线还需要完成：小程序服务类目申请、ICP 备案、服务器域名白名单配置、代码审核等流程。

## 文档索引

| 文档 | 说明 |
| --- | --- |
| [保姆级本地运行指南](./docs/保姆级本地运行指南.md) | 从零到一跑通项目的完整指南 |
| [需求分析文档](./docs/需求分析文档.md) | 产品需求与功能清单 |
| [方案设计文档](./docs/方案设计文档.md) | 技术选型、架构与接口设计 |
| [用户系统需求分析文档](./docs/用户系统需求分析文档.md) | 用户系统专项设计 |
| [用户系统方案设计文档](./docs/用户系统方案设计文档.md) | 用户系统专项方案 |
| [OpenSpec 使用教程](./docs/OpenSpec使用教程.md) | 规范驱动开发流程说明 |

## 许可证

本项目基于 [MIT License](./LICENSE) 开源。
