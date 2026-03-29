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

## Recommended Configuration

This is a more practical baseline for personal use than the minimal example above.
It keeps the JSON API enabled, limits the enabled engines, and applies a few safer privacy and filtering defaults.

```yml
# Enable Bing web search explicitly.
# It may not be preserved reliably by keep_only alone.
engines:
  - disabled: false
    name: bing

# Outgoing request settings (SearXNG -> search engines)
outgoing:
  enable_http2: true          # Use HTTP/2 when available for better performance
  request_timeout: 5.0        # Give upstream engines at most 5 seconds to respond

# Search behavior
search:
  autocomplete: google        # Use Google for query autocomplete suggestions
  default_lang: zh            # Default search language: Chinese
  formats:                    # Keep both UI and API formats enabled
    - html                    # Browser UI
    - json                    # JSON API for agents and scripts
  safe_search: 1              # 0=off, 1=moderate, 2=strict

# Server settings
server:
  image_proxy: true           # Proxy remote images through SearXNG for privacy
  limiter: false              # Fine for personal use; enable for public deployments
  secret_key: YOUR_RANDOM_SECRET_KEY  # Replace this with a real random secret in production

# Frontend settings
ui:
  static_use_hash: true       # Add hashes to static assets to avoid stale caches

# Engine allowlist (core safety setting)
use_default_settings:
  engines:
    keep_only:                # Keep only the engines below and disable the rest
      - google                # Google web search
      - google images         # Google Images
      - google news           # Google News
      - google videos         # Google Videos
      - google scholar        # Google Scholar
      - bing                  # Bing web search
      - bing images           # Bing Images
      - bing news             # Bing News
      - bing videos           # Bing Videos
      - wikipedia             # Wikipedia
```


## License

MIT
