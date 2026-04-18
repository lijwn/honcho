# Honcho 部署指南

本指南帮助开发者快速部署 Honcho 服务。

## 目录

- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [环境变量详解](#环境变量详解)
- [常见部署方案](#常见部署方案)
- [故障排查](#故障排查)

---

## 快速开始

### 前置要求

- Docker
- Docker Compose

### 步骤 1：克隆仓库

```bash
git clone https://github.com/lijwn/honcho.git
cd honcho
```

### 步骤 2：配置环境变量

```bash
# 复制模板文件
cp .env.template .env

# 编辑配置文件，填入你的配置
vim .env
```

### 步骤 3：启动服务

```bash
# 首次构建并启动
docker compose up -d --build

# 查看运行状态
docker compose ps
```

### 步骤 4：验证部署

```bash
# 检查 API 是否正常运行
curl http://127.0.0.1:8000/docs

# 查看日志
docker compose logs -f
```

服务启动后，访问 http://127.0.0.1:8000/docs 查看 API 文档。

---

## 配置说明

### 最低配置（必须修改）

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `DB_CONNECTION_URI` | 数据库连接字符串 | `postgresql+psycopg://postgres:postgres@database:5432/postgres` |
| `LLM_OPENAI_COMPATIBLE_BASE_URL` | LLM API 端点 | `https://api.openai.com/v1` |
| `LLM_OPENAI_COMPATIBLE_API_KEY` | LLM API Key | `sk-xxxx...` |
| `DERIVER_MODEL` | Deriver 模型名称 | `gpt-4o` |
| `DIALECTIC_LEVELS__minimal__MODEL` | 最低级别对话模型 | `gpt-4o-mini` |

### 推荐配置（根据需求修改）

#### 使用独立 Embedding 服务

如果你的 Embedding 服务与 LLM 服务不同（例如使用 SiliconFlow、BAAI 等）：

```bash
# Embedding 配置
LLM_EMBEDDING_PROVIDER=custom
LLM_EMBEDDING_BASE_URL=https://api.siliconflow.cn/v1
LLM_EMBEDDING_API_KEY=sk-xxxx...
LLM_EMBEDDING_MODEL=BAAI/bge-m3
VECTOR_STORE_DIMENSIONS=1024
```

#### 使用相同的 LLM 服务作为 Embedding

```bash
LLM_EMBEDDING_PROVIDER=openrouter
LLM_EMBEDDING_MODEL=openai/text-embedding-3-small
VECTOR_STORE_DIMENSIONS=1536
```

---

## 环境变量详解

### 数据库配置

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `DB_CONNECTION_URI` | (必填) | PostgreSQL 连接字符串 |
| `DB_SCHEMA` | `public` | 数据库 Schema |
| `DB_POOL_SIZE` | `10` | 连接池大小 |

### LLM 配置

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `LLM_OPENAI_COMPATIBLE_BASE_URL` | (必填) | OpenAI 兼容 API 端点 |
| `LLM_OPENAI_COMPATIBLE_API_KEY` | (必填) | API Key |
| `LLM_DEFAULT_MAX_TOKENS` | `2500` | 默认最大 Token 数 |

### Embedding 配置

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `LLM_EMBEDDING_PROVIDER` | `openai` | Embedding 提供商：`openai`, `gemini`, `openrouter`, `custom` |
| `LLM_EMBEDDING_BASE_URL` | - | 自定义 Embedding 端点（仅当 PROVIDER=custom 时） |
| `LLM_EMBEDDING_API_KEY` | - | 自定义 Embedding API Key |
| `LLM_EMBEDDING_MODEL` | - | Embedding 模型名称 |
| `VECTOR_STORE_DIMENSIONS` | `1536` | 向量维度（根据模型调整：OpenAI=1536, BGE-M3=1024） |

### 各功能模块模型配置

| 模块 | 变量 | 说明 |
|------|------|------|
| Deriver | `DERIVER_PROVIDER`, `DERIVER_MODEL` | 记忆提取 |
| Dialectic | `DIALECTIC_LEVELS__*__MODEL` | 对话推理（支持 minimal/low/medium/high/max 级别） |
| Summary | `SUMMARY_PROVIDER`, `SUMMARY_MODEL` | 会话总结 |
| Dream | `DREAM_PROVIDER`, `DREAM_MODEL` | 记忆整合 |

---

## 常见部署方案

### 方案 1：本地开发部署

```bash
# 启动所有服务
docker compose up -d

# 查看日志
docker compose logs -f api

# 停止服务
docker compose down
```

### 方案 2：重置数据库

如果需要重新初始化数据库（注意：这会删除所有数据）：

```bash
# 停止服务并删除数据卷
docker compose down -v

# 重新启动（会自动创建新数据库）
docker compose up -d --build
```

### 方案 3：使用外部数据库

修改 `.env` 中的数据库连接：

```bash
DB_CONNECTION_URI=postgresql+psycopg://user:password@your-host:5432/your-db
```

### 方案 4：自定义向量维度

如果使用不同的 Embedding 模型，需要设置对应的维度：

```bash
# BAAI/bge-m3
VECTOR_STORE_DIMENSIONS=1024

# OpenAI text-embedding-3-small
VECTOR_STORE_DIMENSIONS=1536

# 其他模型请查阅文档
```

---

## 故障排查

### 服务无法启动

```bash
# 查看详细日志
docker compose logs api
docker compose logs deriver

# 检查容器状态
docker compose ps
```

### 数据库连接失败

1. 确认数据库服务正常运行：
   ```bash
   docker compose ps database
   ```

2. 检查连接配置：
   ```bash
   docker exec -it honcho-api-1 sh -c "echo \$DB_CONNECTION_URI"
   ```

### Embedding 服务连接失败

1. 检查 API Key 和端点配置
2. 测试网络连通性：
   ```bash
   docker exec -it honcho-api-1 sh -c "curl -s \$LLM_EMBEDDING_BASE_URL/models"
   ```

### 向量维度不匹配

如果遇到向量维度相关错误：

1. 确认 `VECTOR_STORE_DIMENSIONS` 与 Embedding 模型输出维度一致
2. 如需更改维度，需要重建数据库：
   ```bash
   docker compose down -v
   docker compose up -d --build
   ```

---

## API 文档

服务启动后，访问以下地址查看完整 API 文档：

- Swagger UI: http://127.0.0.1:8000/docs
- ReDoc: http://127.0.0.1:8000/redoc

---

## 更多信息

- 官方文档：https://docs.honcho.dev
- GitHub 仓库：https://github.com/lijwn/honcho
- 问题反馈：https://github.com/lijwn/honcho/issues