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

## 推荐配置

下面这份配置比上面的最小示例更适合作为个人使用的起点。
它保留了 JSON API，限制了可用搜索引擎，并增加了一些更稳妥的隐私和内容过滤默认值。

```yml
# 单独启用 Bing 通用搜索（因为 keep_only 不一定能稳定保留它，建议手动开启）
engines:
  - disabled: false
    name: bing

# 出站请求配置（SearXNG -> 搜索引擎）
outgoing:
  enable_http2: true          # 尽量使用 HTTP/2，通常比 HTTP/1.1 更快
  request_timeout: 5.0        # 最多等待搜索引擎响应 5 秒，超时就放弃

# 搜索行为配置
search:
  autocomplete: google        # 输入框自动补全建议来自 Google
  default_lang: zh            # 默认搜索语言：中文
  formats:                    # 同时保留网页界面和 JSON API
    - html                    # 浏览器访问使用
    - json                    # Agent / 脚本调用必须保留
  safe_search: 1              # 安全搜索：0=关闭，1=适度过滤，2=严格

# 服务器配置
server:
  image_proxy: true           # 图片通过 SearXNG 代理加载，减少隐私泄露
  limiter: false              # 自用可关闭；公开部署建议开启
  secret_key: YOUR_RANDOM_SECRET_KEY  # 生产环境务必替换成随机字符串

# 前端界面配置
ui:
  static_use_hash: true       # 静态资源加 hash，避免缓存旧文件

# 引擎白名单（核心安全配置）
use_default_settings:
  engines:
    keep_only:                # 只保留以下引擎，其余全部禁用
      - google                # Google 网页搜索
      - google images         # Google 图片
      - google news           # Google 新闻
      - google videos         # Google 视频
      - google scholar        # Google 学术
      - bing                  # Bing 网页搜索
      - bing images           # Bing 图片
      - bing news             # Bing 新闻
      - bing videos           # Bing 视频
      - wikipedia             # 维基百科
```

## 许可证

MIT
