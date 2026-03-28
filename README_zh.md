# SearXNG 搜索技能

[English](README.md)

一个通过本地部署的 [SearXNG](https://github.com/searxng/searxng) 实例提供网页搜索能力的技能。搜索网页，获取 AI 总结的结果。

为 [OpenClaw](https://github.com/betaHi/openclaw-searxng-search) 设计，但可以在任何支持技能工作流的 AI 编程助手中使用。

## 特性

- **通过 SearXNG 搜索网页** — 本地运行的隐私友好元搜索引擎
- **AI 智能总结** — 搜索结果会被阅读并总结为简洁的回答
- **自动部署** — 首次使用时自动通过 Docker 部署 SearXNG 容器
- **CJK 支持** — 完整支持中文、日文、韩文查询
- **多语言搜索** — 使用 `--lang` 指定搜索语言

## 快速开始

### 1. 添加技能到你的项目

将 `skills/searxng-search/SKILL.md` 复制到 `~/.openclaw/skills` 或你项目的技能目录中。

### 2. 使用

```
/searxng-search 间隔重复是什么
/searxng-search what is spaced repetition --lang en
```

就这样。如果 SearXNG 没有运行，技能会自动通过 Docker 部署它。

## 前置要求

- 已安装 [Docker](https://docs.docker.com/get-docker/)（或者让技能自动安装）
- Linux / WSL 环境（自动安装需要 Debian/Ubuntu 系）

## 工作原理

```
/searxng-search <查询>
       │
       ├─ 健康检查 SearXNG (localhost:8080)
       │     ├── 运行中 → 直接搜索
       │     └── 未运行 → 通过 Docker 自动部署
       │
       ├─ 通过 curl 搜索 → 提取前 5 条结果
       │
       └─ AI 总结并展示结果
```

## 使用方式

```
/searxng-search <查询>              # 自动检测语言
/searxng-search <查询> --lang zh    # 指定中文搜索
/searxng-search <查询> --lang en    # 指定英文搜索
```

### 输出示例

```
**关于"间隔重复"：**

间隔重复是一种利用记忆遗忘曲线来优化学习效率的方法，
通过在逐渐增加的时间间隔后复习材料来加强长期记忆。

**相关结果：**
1. [间隔重复 - 维基百科](https://zh.wikipedia.org/wiki/间隔重复)
   间隔重复的完整介绍...
   来源: google

2. [如何高效记忆...](https://example.com/...)
   实践指南...
   来源: bing

*来源: SearXNG (google, bing, duckduckgo)*
```

## 配置

SearXNG 默认运行在 `localhost:8080`。配置文件存储在 `~/searxng/settings.yml`。

技能会自动处理部署，如果你想手动配置：

```bash
mkdir -p ~/searxng

cat > ~/searxng/settings.yml << 'EOF'
use_default_settings: true

server:
  secret_key: "改成你自己的随机字符串"
  limiter: false

search:
  formats:
    - html
    - json
EOF

sudo docker run -d \
  --restart unless-stopped \
  -p 8080:8080 \
  -v ~/searxng:/etc/searxng \
  --name searxng \
  searxng/searxng
```

## 许可证

MIT
