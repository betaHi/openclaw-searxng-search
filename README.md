# SearXNG Search Skill

[中文版](README_zh.md)

A search skill that provides web search capabilities through a locally deployed [SearXNG](https://github.com/searxng/searxng) instance. Search the web, get AI-summarized results.

Designed for [OpenClaw](https://github.com/betaHi/openclaw-searxng-search), but works with any AI coding assistant that supports skill-based workflows.

## Features

- **Web search via SearXNG** — privacy-respecting metasearch engine running locally
- **AI-powered summaries** — search results are read and summarized into concise answers
- **Auto-deployment** — SearXNG Docker container is set up automatically on first use
- **CJK support** — full support for Chinese, Japanese, Korean queries
- **Multi-language** — use `--lang` to search in any language

## Quick Start

### 1. Add the skill to your project

Copy `skills/searxng-search/SKILL.md` into your `~/.openclaw/skills` or your project's skill directory.

### 2. Use it

```
/searxng-search what is spaced repetition
/searxng-search 间隔重复是什么 --lang zh
```

That's it. If SearXNG isn't running, the skill will deploy it automatically via Docker.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed (or the skill will try to install it)
- Linux / WSL environment (Debian/Ubuntu-based for auto-install)

## How It Works

```
/searxng-search <query>
       │
       ├─ Health check SearXNG (localhost:8080)
       │     ├── Running → search directly
       │     └── Not running → auto-deploy via Docker
       │
       ├─ Search via curl → extract top 5 results
       │
       └─ AI summarizes and presents results
```

## Usage

```
/searxng-search <query>              # search with auto-detected language
/searxng-search <query> --lang zh    # search in Chinese
/searxng-search <query> --lang ja    # search in Japanese
```

### Output Format

```
**About "spaced repetition":**

Spaced repetition is a learning technique that optimizes review intervals
based on the forgetting curve to strengthen long-term memory retention.

**Results:**
1. [Spaced repetition - Wikipedia](https://en.wikipedia.org/wiki/Spaced_repetition)
   Complete introduction to spaced repetition...
   Source: google

2. [How to Remember Anything...](https://example.com/...)
   Practical guide...
   Source: bing

*Source: SearXNG (google, bing, duckduckgo)*
```

## Configuration

SearXNG runs on `localhost:8080` by default. Configuration is stored at `~/searxng/settings.yml`.

The skill handles deployment automatically, but if you want to set it up manually:

```bash
mkdir -p ~/searxng

cat > ~/searxng/settings.yml << 'EOF'
use_default_settings: true

server:
  secret_key: "change-me-to-something-random"
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

## License

MIT
