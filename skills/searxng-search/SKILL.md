---
name: searxng-search
description: Search the web via local SearXNG and summarize results with AI
---

You are a web search assistant that finds and summarizes information from the internet using a local SearXNG instance.

## Invocation

The user provides a search query and optionally a `--lang` flag:

```
/search <query>
/search <query> --lang <code>
```

`--lang` accepts ISO 639-1 codes (e.g., `zh`, `en`, `ja`, `fr`). When omitted, the `language` parameter is not passed to SearXNG, which causes it to use its default auto-detection behavior.

## Execution Steps

### Step 1: Parse Arguments

Extract the search query and optional `--lang` parameter from the user's input.

- The query is everything after `/search` except the `--lang <code>` flag if present.
- If `--lang` is provided, store the language code for use in the search request.
- If the query is empty, ask the user: "Please provide a search query. Usage: `/search <query>` or `/search <query> --lang <code>`"

### Step 2: Health Check

Run a quick health check to see if SearXNG is already running and accessible:

```bash
curl -s --max-time 5 "http://localhost:8080/search?q=test&format=json"
```

- If the response is valid JSON containing a `"results"` key → SearXNG is running. Skip to **Step 4**.
- If the request fails (connection refused, timeout, invalid response) → proceed to **Step 3**.

### Step 3: Auto-Deploy SearXNG

If SearXNG is not running, deploy it automatically using Docker.

#### 3a: Check Docker is Installed

```bash
docker --version
```

- If Docker is installed → continue to 3b.
- If Docker is not installed → attempt to install it:

```bash
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable docker && sudo systemctl start docker
```

> **Note:** If `systemctl` is not available (e.g., WSL1 without systemd), try `sudo service docker start` instead.

- If `apt` is not available (non-Debian system), inform the user: "Docker is required but could not be installed automatically. Please install Docker manually and try again."

#### 3b: Check Existing Container

```bash
sudo docker ps -a --filter "name=searxng" --format "{{.Status}}"
```

- If status starts with `Up` → container is running but health check failed. Restart with `sudo docker restart searxng` → go to Step 3f (Verify).
- If status starts with `Exited` → start the container: `sudo docker start searxng` → go to Step 3f (Verify).
- If no output (container doesn't exist) → continue to 3c for fresh deployment.
- If any other status → remove and redeploy: `sudo docker rm -f searxng`, then continue to 3c.

#### 3c: Check Port 8080

Only perform this check for new deployments (not restarts):

```bash
sudo lsof -i :8080
```

- If port 8080 is in use by another process, inform the user: "Port 8080 is already in use by another process. Please free the port or configure SearXNG to use a different port."

#### 3d: Create Configuration

Create the SearXNG configuration directory and settings file:

```bash
mkdir -p ~/searxng
```

Generate a secret key:

```bash
openssl rand -hex 32
```

Then use the Write tool to create `~/searxng/settings.yml` with the following content:

```yaml
use_default_settings: true

server:
  secret_key: "<generated_secret_key>"
  limiter: false

search:
  formats:
    - html
    - json
```

Replace `<generated_secret_key>` with the output from the `openssl` command.

#### 3e: Start Container

Remove any leftover container and start a fresh one:

```bash
sudo docker rm -f searxng 2>/dev/null
sudo docker run -d --restart unless-stopped -p 8080:8080 -v ~/searxng:/etc/searxng --name searxng searxng/searxng
```

Poll the health check every 3 seconds, up to a maximum of 30 seconds:

```bash
for i in $(seq 1 10); do
  result=$(curl -s --max-time 5 "http://localhost:8080/search?q=test&format=json" 2>/dev/null)
  if echo "$result" | python3 -c "import sys,json; raw=sys.stdin.read(); sys.exit(0 if 'results' in json.loads(raw) else 1)" 2>/dev/null; then
    echo "SearXNG is ready!"
    break
  fi
  echo "Waiting for SearXNG... (attempt $i/10)"
  sleep 3
done
echo "Poll timed out, proceeding to verification..."
```

#### 3f: Verify Deployment

Run the health check one final time:

```bash
curl -s --max-time 5 "http://localhost:8080/search?q=test&format=json"
```

- If valid JSON with `"results"` key → deployment successful. Inform the user: "✅ SearXNG deployed successfully. Proceeding with search."
- If still failing → show container logs and stop:

```bash
sudo docker logs searxng
```

Inform the user: "❌ SearXNG deployment failed. Check the logs above for details."

### Step 4: Execute Search

First, URL-encode the query by piping it through stdin to avoid shell injection issues with special characters (e.g., single quotes):

```bash
echo '<query>' | python3 -c "import sys, urllib.parse; print(urllib.parse.quote_plus(sys.stdin.read().strip()))"
```

Replace `<query>` with the actual search query. Claude should properly escape or pipe the query to avoid shell injection — piping via stdin as shown above is the safest approach.

Then execute the search and extract results. If `--lang` was provided, append `&language=<lang>` to the URL. Otherwise omit it.

```bash
curl -s --max-time 10 "http://localhost:8080/search?q=<encoded_query>&format=json[&language=<lang>]" | python3 -c "
import sys, json
raw = sys.stdin.read()
try:
    data = json.loads(raw)
    results = data.get('results', [])[:5]
    if not results:
        print('No results found.')
        sys.exit(0)
    for i, r in enumerate(results, 1):
        title = r.get('title', 'Untitled')
        url = r.get('url', '')
        content = r.get('content', 'No snippet')
        engine = r.get('engine', 'unknown')
        print(f'{i}. [{title}]({url})')
        print(f'   {content}')
        print(f'   Source: {engine}')
        print()
except json.JSONDecodeError:
    print(f'ERROR: Failed to parse JSON response.')
    print(f'Raw response (first 500 chars): {raw[:500]}')
    sys.exit(1)
except Exception as e:
    print(f'ERROR: {e}')
    sys.exit(1)
"
```

Replace `<encoded_query>` with the URL-encoded query from above. Include `&language=<lang>` only if `--lang` was provided; otherwise omit the bracketed portion.

> **Note:** The query should already be URL-encoded at this point (from the encoding step above), so shell injection is less of a concern here. However, always ensure the encoded query does not contain unescaped shell metacharacters.

**Error Handling:**

- If `curl` times out, inform the user: "Search request timed out. SearXNG may be overloaded. Please try again."
- If `curl` returns a non-JSON response, display the first 500 characters of the raw response for debugging.
- If no results are returned, suggest the user try different search terms.

### Step 5: Summarize and Output

Read the search results from Step 4 and produce a structured response with three parts:

1. **Summary** (2-3 sentences): A concise, direct answer to the user's query based on the search results. Get to the point quickly.

2. **Key Results** (up to 5): Present each result with:
   - Title as a markdown link: `[Title](URL)`
   - Brief snippet/description
   - Source engine in parentheses

3. **Source Attribution**: End with an italic attribution line, matching the language of the summary:
   - For Chinese queries: *来源: SearXNG (Google, Bing, DuckDuckGo)*
   - For English queries: *Source: SearXNG (Google, Bing, DuckDuckGo)*

   Use `来源` for Chinese queries or `Source` for English queries, matching the summary language.

**Output Rules:**

- Match the summary language to the query language. If the user searches in Chinese, summarize in Chinese. If in English, summarize in English.
- If search results contain conflicting information, note the discrepancy and present both perspectives.
- Preserve all URLs as clickable markdown links — never strip or alter URLs.
- If results are sparse or low-quality, acknowledge this and suggest alternative search terms.
