---
name: bohrium-mentor
description: "AI science tutor powered by Bohrium's adk_science_navigator agent. Submits a research question, subscribes to an SSE stream, and returns a structured Markdown answer. Use when: the user wants an AI-guided explanation of a scientific topic, a research progress summary, or a methodology comparison. NOT for: paper search (use bohrium-paper-search), literature review (use literature-review), or scholar lookup (use bohrium-scholar-search)."
---

# SKILL: AI Science Mentor (Science Navigator)

## Overview

**AI Science Mentor** calls Bohrium's `adk_science_navigator` agent to answer scientific questions with deep reasoning. The agent creates a session, streams intermediate and final response events over SSE, and returns a structured Markdown answer.

**Flow:**

```text
User scientific question
  |
  |-- Step 1: Create a session (POST sessions)
  |           -> get sessionId
  |
  |-- Step 2: Subscribe to the SSE stream (GET stream)
  |           -> receive streamed reasoning and answer fragments
  |
  |-- Step 3: Merge events and print the final answer
  |
  v
Output: reasoning fragments + structured Markdown answer
```

**Use cases:**

- Deep scientific Q&A, such as "recent CRISPR-Cas9 progress in gene therapy"
- Research method comparison and assessment
- Recent progress summaries for a scientific field
- Cross-disciplinary concept explanations

**Not for:**

- Specific paper search -> use `bohrium-paper-search`
- Systematic literature review -> use `literature-review`
- Scholar profile lookup -> use `bohrium-scholar-search`

**No CLI support** - use the Python script below to call the SSE APIs.

## Authentication

```json
"bohrium-mentor": {
  "enabled": true,
  "apiKey": "YOUR_BOHR_ACCESS_KEY",
  "env": {
    "BOHR_ACCESS_KEY": "YOUR_BOHR_ACCESS_KEY"
  }
}
```

The script reads `BOHR_ACCESS_KEY` from the environment.

## API Endpoints

This skill uses:

| # | API | Method and Path | Purpose |
|---|-----|-----------------|---------|
| 1 | Create session | `POST /v2/sigma-search/api/v4/ai_search/sessions` | Submit the question and get `sessionId` |
| 2 | SSE stream | `GET /v2/sigma-search/api/v3/sse/ai_search/v1/{sessionId}/stream` | Receive streamed reasoning and answer events |
| 3 | Session detail | `GET /v2/sigma-search/api/v4/ai_search/sessions/{sessionId}` | Recover or replay session output |

Authentication uses the `Authorization: Bearer <BOHR_ACCESS_KEY>` header.

## Inputs

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Scientific question, for example "applications of quantum computing in drug discovery" |
| `discipline` | string | No | Discipline filter: `All` (default), `Physics`, `Chemistry`, `Biology`, etc. |
| `journal_type` | string | No | `foreign` (default) or `chinese` |
| `model` | string | No | `reason` (default) |

## Output

| Field | Description |
|-------|-------------|
| Reasoning | Streamed preprocessor content showing agent's thought chain |
| Final answer | Structured Markdown answer from the main layout |
| References | Papers/web pages cited in the answer (layout=sider, tab id=used) with title, authors, journal, DOI |
| All search results | Full list of retrieved resources (layout=sider, tab id=all), including papers and web pages |
| Recommended questions | Follow-up questions suggested by the agent (layout=postprocessor) |

---

## SSE Protocol

Each SSE event has the following shape:

```text
event:data
data:{"channel":{...},"system":{...},"sessionId":"..."}
```

Important fields:

| Path | Description |
|------|-------------|
| `system.event.type` | `start`, `streaming`, `end`, or `error` |
| `channel.uiInfo.layout` | `preprocessor` for reasoning, `main` for the final answer |
| `channel.uiInfo.subType` | Content component type |
| `channel.uiInfo.actionList[].action` | `append`, `delta`, or `patch` |
| `channel.uiInfo.content` | Current content fragment |
| `channel.answerId` | Message identifier. Events with the same `answerId` belong to the same answer block |

Merge rules:

1. Maintain accumulated state by `answerId` and `layout`.
2. For `action=append` or `action=delta`, append `content[key]` to the accumulated text.
3. For `action=patch`, apply JSON Patch (RFC 6902) from `uiInfo.patches` array to build resourceMap and sider tabs incrementally.
4. On `system.event.type=end`, the `main.text` state contains the final answer if the stream completed cleanly.

### Resource and Citation Data (JSON Patch transport)

Reference metadata and citation lists are transmitted incrementally via `action=patch`:

| Data | Source | Example patch path |
|------|--------|--------------------|
| Resource metadata | `layout=main, subType=@bohrium-chat/snp/cards-data` | `/data/resourceMap/<resourceId>` |
| References list | `layout=sider, subType=@bohrium-chat/snp/sider-tab/v2` | `/tabs/0/items/-` (References tab) |
| All search results | `layout=sider, subType=@bohrium-chat/snp/sider-tab/v2` | `/tabs/1/items/-` (All tab) |

**Resource types in resourceMap:**

- `@bohrium/card-type/paper`: Academic paper with title, authors, journal, doi, citationNums, impactFactor, jcrZone, url, abstract
- `@bohrium/card-type/web`: Web page with title, url, source, snippet, date

**Sider tabs structure:**

```json
{
  "tabs": [
    {"id": "used", "title": "References", "items": [{"resourceId": "@bohrium:doi:xxx", "@bohrium-type": "..."}]},
    {"id": "all", "title": "All search results", "items": [...]}
  ]
}
```

---

## Complete Script

Dependencies: `pip install requests jsonpatch`

```python
import os
import sys
import json
import uuid
import requests
import jsonpatch


AK = os.environ.get("BOHR_ACCESS_KEY", "")
if not AK:
    print("ERROR: BOHR_ACCESS_KEY is not configured.")
    sys.exit(1)

BASE = "https://open.bohrium.com/openapi/v2/sigma-search"

QUERY = sys.argv[1] if len(sys.argv) > 1 else "Recent CRISPR-Cas9 progress in gene therapy"
DISCIPLINE = sys.argv[2] if len(sys.argv) > 2 else "All"
JOURNAL_TYPE = sys.argv[3] if len(sys.argv) > 3 else "foreign"
MODEL = "reason"

print("=" * 60)
print("  AI Science Mentor (Science Navigator)")
print(f"  Question: {QUERY}")
print(f"  Discipline: {DISCIPLINE}")
print(f"  Journal type: {JOURNAL_TYPE}")
print(f"  Model: {MODEL}")
print("=" * 60)


def create_session(query, discipline, journal_type, model):
    """Create an AI search session and return sessionId."""
    print("\n[Step 1/3] Creating session...")

    message_id = str(uuid.uuid4())
    answer_id = str(uuid.uuid4())

    payload = {
        "query": query,
        "model": model,
        "discipline": discipline,
        "scene": "adk_science_navigator",
        "journal_type": journal_type,
        "snp_version": "1.0.0",
        "resource_id_list": [],
        "SNPReq": {
            "sessionId": "",
            "channel": {
                "schema": "fe",
                "version": "v1",
                "role": "user",
                "auth": 1,
                "messageId": message_id,
                "answerId": answer_id,
                "uiInfo": {
                    "layout": "main",
                    "type": "ui",
                    "subType": "@bohrium-chat/common/markdown",
                    "content": {"text": query},
                    "actionList": [{"key": "text", "action": "append"}],
                },
                "entities": [],
                "state": {},
                "meta": {},
            },
            "system": {
                "payload": {
                    "model": model,
                    "agentId": "science_navigator",
                    "sessionId": "",
                    "scene": "adk_science_navigator",
                    "streaming": True,
                    "biz": {
                        "uploadList": [],
                        "sn": {
                            "discipline": discipline,
                            "journal_type": journal_type,
                            "model": model,
                        },
                    },
                },
            },
        },
    }

    url = f"{BASE}/api/v4/ai_search/sessions"
    try:
        response = requests.post(url, headers={"Authorization": f"Bearer {AK}"}, json=payload, timeout=30)
        response.raise_for_status()
    except requests.exceptions.Timeout:
        print("  Error: create session timeout")
        return None
    except requests.exceptions.RequestException as exc:
        print(f"  Error: request failed - {exc}")
        return None

    data = response.json()
    if data.get("code") != 0:
        print(f"  Error: {data.get('message', 'unknown error')} (code={data.get('code')})")
        return None

    session_id = data["data"]["sessionId"]
    print(f"  Session created: {session_id}")
    return session_id


def subscribe_sse(session_id):
    """Subscribe to SSE stream, merge events, return text state, resourceMap, sider tabs, and recommendations."""
    print("\n[Step 2/3] Subscribing to SSE stream...")

    url = f"{BASE}/api/v3/sse/ai_search/v1/{session_id}/stream"

    text_state = {}
    cards_state = {"data": {"interaction": {"summary": {}}, "resourceMap": {}}}
    sider_state = {}
    recommended_questions = []

    try:
        with requests.get(url, headers={"Authorization": f"Bearer {AK}"}, stream=True, timeout=300) as response:
            response.raise_for_status()
            buf = b""
            event_count = 0
            started = False

            for chunk in response.iter_content(chunk_size=4096):
                if not chunk:
                    continue
                buf += chunk

                while b"\n\n" in buf:
                    raw_bytes, buf = buf.split(b"\n\n", 1)
                    raw = raw_bytes.decode("utf-8", errors="replace")
                    lines = [line for line in raw.splitlines() if line.startswith("data:")]
                    if not lines:
                        continue

                    try:
                        evt = json.loads(lines[0][5:])
                    except json.JSONDecodeError:
                        continue

                    channel = evt.get("channel", {})
                    system = evt.get("system", {})
                    event_type = system.get("event", {}).get("type", "")

                    if event_type == "start":
                        if not started:
                            print("  Received start event.")
                            started = True
                        continue

                    if event_type == "error":
                        err_msg = system.get("event", {}).get("message", "unknown error")
                        print(f"  Error: {err_msg}")
                        break

                    if event_type == "end":
                        print("  Received end event.")
                        break

                    ui_info = channel.get("uiInfo") or {}
                    answer_id = channel.get("answerId", "default")
                    layout = ui_info.get("layout", "unknown")
                    subtype = ui_info.get("subType", "")
                    action_list = ui_info.get("actionList") or []
                    content = ui_info.get("content") or {}
                    patches = ui_info.get("patches") or []

                    # ── Text content merge (append/delta) ──
                    if any(a.get("action") in ("append", "delta") for a in action_list):
                        if layout in ("preprocessor", "main") and subtype == "@bohrium-chat/common/markdown":
                            slot = text_state.setdefault(answer_id, {}).setdefault(layout, {})
                            for act in action_list:
                                action_type = act.get("action", "")
                                key = act.get("key", "")
                                if action_type in ("append", "delta") and key:
                                    val = content.get(key, "")
                                    if isinstance(val, str):
                                        slot[key] = slot.get(key, "") + val

                        # Sider initialization (first append carries full tabs structure)
                        if layout == "sider" and isinstance(content, dict) and "tabs" in content:
                            sider_state = content

                        # Cards-data initialization
                        if subtype == "@bohrium-chat/snp/cards-data" and isinstance(content, dict):
                            if "data" in content and isinstance(content["data"], dict):
                                rm = content["data"].get("resourceMap", {})
                                if rm:
                                    cards_state["data"]["resourceMap"].update(rm)

                    # ── JSON Patch merge (resourceMap and sider tabs) ──
                    if patches:
                        if subtype == "@bohrium-chat/snp/cards-data":
                            try:
                                patch = jsonpatch.JsonPatch(patches)
                                cards_state = patch.apply(cards_state)
                            except Exception:
                                pass
                        elif layout == "sider":
                            try:
                                patch = jsonpatch.JsonPatch(patches)
                                sider_state = patch.apply(sider_state)
                            except Exception:
                                pass

                    # ── Recommended questions ──
                    if layout == "postprocessor" and subtype == "@bohrium-chat/common/relevant-search":
                        if isinstance(content, dict) and "data" in content:
                            data_list = content["data"]
                            if isinstance(data_list, list):
                                recommended_questions = data_list

                    event_count += 1
                    if event_count % 50 == 0:
                        print(f"  Received {event_count} events...")
                else:
                    continue
                break

    except requests.exceptions.Timeout:
        print("  Warning: SSE stream timed out (300s); returning received content")
    except requests.exceptions.RequestException as exc:
        print(f"  Error: SSE connection failed - {exc}")

    resource_map = cards_state.get("data", {}).get("resourceMap", {})
    return text_state, resource_map, sider_state, recommended_questions


def format_output(text_state, resource_map, sider_state, recommended_questions):
    """Format the full output including answer, references, and search results."""
    reasoning_parts = []
    main_parts = []

    for layouts in text_state.values():
        for layout, content in layouts.items():
            text = content.get("text", "")
            if not text:
                continue
            if layout == "preprocessor":
                reasoning_parts.append(text)
            elif layout == "main":
                main_parts.append(text)

    output_lines = []

    # ── Reasoning ──
    if reasoning_parts:
        output_lines.append("\n## Reasoning\n")
        output_lines.extend(part.strip() for part in reasoning_parts)
        output_lines.append("")

    # ── Final Answer ──
    if main_parts:
        output_lines.append("\n## Final Answer\n")
        output_lines.extend(part.strip() for part in main_parts)
    elif not reasoning_parts:
        output_lines.append("\n(No valid response content received.)")

    # ── References ──
    tabs = sider_state.get("tabs", [])
    ref_tab = next((t for t in tabs if t.get("id") == "used"), None)
    if ref_tab and ref_tab.get("items"):
        output_lines.append("\n\n## References\n")
        for idx, item in enumerate(ref_tab["items"], 1):
            rid = item.get("resourceId", "")
            res = resource_map.get(rid, {})
            btype = res.get("@bohrium-type", item.get("@bohrium-type", ""))
            if btype == "@bohrium/card-type/paper":
                title = res.get("title", res.get("titleEn", rid))
                authors = res.get("authors", [])
                author_str = ", ".join(authors[:3])
                if len(authors) > 3:
                    author_str += " et al."
                journal = res.get("journal", "")
                doi = res.get("doi", "")
                citation = res.get("citationNums", "")
                jcr = res.get("jcrZone", "")
                impact = res.get("impactFactor", "")
                url = res.get("url", "")
                line = f"{idx}. **{title}**"
                if author_str:
                    line += f"\n   - Authors: {author_str}"
                if journal:
                    meta_parts = [journal]
                    if jcr:
                        meta_parts.append(f"JCR {jcr}")
                    if impact:
                        meta_parts.append(f"IF {impact}")
                    line += f"\n   - Journal: {' | '.join(meta_parts)}"
                if doi:
                    line += f"\n   - DOI: {doi}"
                if citation:
                    line += f"\n   - Citations: {citation}"
                if url:
                    line += f"\n   - URL: {url}"
                output_lines.append(line)
            else:
                title = res.get("title", rid)
                url = res.get("url", "")
                source = res.get("source", "")
                line = f"{idx}. **{title}**"
                if url:
                    line += f"\n   - URL: {url}"
                if source:
                    line += f"\n   - Source: {source}"
                output_lines.append(line)

    # ── All Search Results ──
    all_tab = next((t for t in tabs if t.get("id") == "all"), None)
    if all_tab and all_tab.get("items"):
        papers = []
        webs = []
        for item in all_tab["items"]:
            rid = item.get("resourceId", "")
            res = resource_map.get(rid, {})
            btype = res.get("@bohrium-type", item.get("@bohrium-type", ""))
            if btype == "@bohrium/card-type/paper":
                papers.append((rid, res))
            else:
                webs.append((rid, res))

        output_lines.append("\n\n## All Search Results\n")
        output_lines.append(f"Total {len(all_tab['items'])} results (papers: {len(papers)}, web: {len(webs)})\n")

        if papers:
            output_lines.append("### Papers\n")
            for idx, (rid, res) in enumerate(papers, 1):
                title = res.get("title", res.get("titleEn", rid))
                authors = res.get("authors", [])
                author_str = ", ".join(authors[:3])
                if len(authors) > 3:
                    author_str += " et al."
                journal = res.get("journal", "")
                doi = res.get("doi", "")
                line = f"{idx}. **{title}**"
                if author_str:
                    line += f" — {author_str}"
                if journal:
                    line += f" | {journal}"
                if doi:
                    line += f" | DOI: {doi}"
                output_lines.append(line)

        if webs:
            output_lines.append("\n### Web Resources\n")
            for idx, (rid, res) in enumerate(webs, 1):
                title = res.get("title", rid)
                url = res.get("url", "")
                line = f"{idx}. {title}"
                if url:
                    line += f" — {url}"
                output_lines.append(line)

    # ── Recommended Questions ──
    if recommended_questions:
        output_lines.append("\n\n## Recommended Questions\n")
        for idx, q in enumerate(recommended_questions, 1):
            output_lines.append(f"{idx}. {q}")

    return "\n".join(output_lines)


if __name__ == "__main__":
    session_id = create_session(QUERY, DISCIPLINE, JOURNAL_TYPE, MODEL)
    if not session_id:
        print("\nFailed to create session.")
        sys.exit(1)

    text_state, resource_map, sider_state, recommended_questions = subscribe_sse(session_id)

    print(f"\n  Resource stats: resourceMap {len(resource_map)} entries", end="")
    tabs = sider_state.get("tabs", [])
    for tab in tabs:
        print(f", {tab.get('title', tab.get('id'))} {len(tab.get('items', []))} items", end="")
    print()

    result = format_output(text_state, resource_map, sider_state, recommended_questions)
    print("\n" + "=" * 60)
    print(result)
    print("=" * 60)
```

---

## Examples

```bash
export BOHR_ACCESS_KEY="your_key"

python3 science_navigator.py "Recent CRISPR-Cas9 progress in gene therapy"
python3 science_navigator.py "Latest strategies for suppressing lithium dendrites in solid-state batteries" "Chemistry" "foreign"
python3 science_navigator.py "Deep learning applications in protein structure prediction" "Biology" "chinese"
```

## Notes

1. **Dependency**: Requires `jsonpatch` library (`pip install jsonpatch`) for merging incremental patch events.
2. **First-token latency**: Deep reasoning can take 30-60 seconds; SSE timeout is set to 300s.
3. **References**: Delivered via JSON Patch on `layout=sider`, `tabs[0]` (id=used) contains cited references, `tabs[1]` (id=all) contains all search results.
4. **Resource metadata**: Patches on `layout=main, subType=@bohrium-chat/snp/cards-data` carry full metadata (title, authors, DOI, journal, etc.) for each resource.
5. **Recovery**: If SSE disconnects, call `GET sessions/{sessionId}` for basic session info (note: full message history is not available via this endpoint).
6. **Rate limiting**: Use single calls or exponential backoff to avoid 429 errors.
