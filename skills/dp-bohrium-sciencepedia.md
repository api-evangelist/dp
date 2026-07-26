---
name: bohrium-sciencepedia
description: "Search and read Bohrium SciencePedia (encyclopedia) via open.bohrium.com. Use when the user wants to: look up / explain a scientific term and get a reading link, browse what disciplines·fields·courses exist, get a course's chapter outline and knowledge list, or explore the knowledge graph around a topic. NOT for: paper search (use bohrium-paper-search), managing your own knowledge base (use bohrium-knowledge-base), or the LKM reasoning graph (use bohrium-lkm)."
---

# SKILL: Bohrium SciencePedia

An encyclopedia of scientific concepts. This skill wraps it into **4 things you can do directly**:

1. **Search a term** → get a summary + a clickable reading link (covers both "topics" and "keywords"; you do **not** need to make the user distinguish them).
2. **Browse what disciplines, fields and courses exist.**
3. **Given a course** → get its chapter structure and knowledge-point list.
4. **Starting from a topic** → get the knowledge graph it forms with related concepts.

> **Two hosts, don't mix them**: call the **API** on `open.bohrium.com`; build the **reading link** you show a human on `https://www.bohrium.com` (see [Building reading links](#building-reading-links)). Never paste the API URL to a user.

**Don't use for**: paper search → `bohrium-paper-search`; your own knowledge base → `bohrium-knowledge-base`; LKM reasoning graph → `bohrium-lkm`.

**No CLI** — HTTP API only.

---

## Auth configuration

```json
"bohrium-sciencepedia": {
  "enabled": true,
  "apiKey": "YOUR_BOHR_ACCESS_KEY",
  "env": { "BOHR_ACCESS_KEY": "YOUR_BOHR_ACCESS_KEY" }
}
```

## Common template (copy-paste)

```python
import os, requests
from urllib.parse import quote

AK = os.environ["BOHR_ACCESS_KEY"]
BASE = "https://open.bohrium.com/openapi/v2/literature-sage/wiki_v2"
H = {"Authorization": f"Bearer {AK}", "Content-Type": "application/json"}

# Almost every endpoint needs these two:
DEFAULTS = {"language": "en-US", "style": "Feynman"}
# language: "en-US" or "zh-CN"
# style:    "Feynman" (accessible) or "Hardcore" (rigorous/academic)

def data(resp):
    """Unwrap: responses are {"code":0,"data":{...}}; errors have code != 0."""
    resp.raise_for_status()
    body = resp.json()
    if body.get("code") not in (0, None):
        raise RuntimeError(f"API error code={body.get('code')}: {body.get('message') or body}")
    return body.get("data", body)
```

## The only 3 concepts you need (ignore the rest of the jargon)

| Concept | What it is | What to give the user |
|---------|------------|-----------------------|
| **Topic (词条)** | A full encyclopedia article | Build an article link from `entry_id` |
| **Keyword (关键词)** | A smaller concept card | Build a keyword link from `keyword_id` |
| **Course / Field (领域)** | A "course" made of many topics | Build a course link from `field_id` |

> Search returns all three mixed together. **To the user they are just "encyclopedia entries"** — give title + summary + link; no need to explain the type difference.

---

## Task 1: Search a term, get summaries and reading links

**One endpoint does it**: `POST /search/universal`. It returns related "topics + keywords" (in `articles`) and related "courses" (in `fields`), each with a highlighted snippet.

```python
d = data(requests.post(f"{BASE}/search/universal", headers=H,
                       json={"text": "graphene", **DEFAULTS}))

for a in d["articles"]:                       # topics and keywords mixed, ranked by relevance
    link = build_read_url(a["type"], a["id"], **DEFAULTS)   # see “Building reading links”
    snippet = a["matched_elements"][0]["content"] if a.get("matched_elements") else ""
    print(a["article_name"], "→", link)
    print("  ", snippet.replace("<em>", "").replace("</em>", ""))   # strip highlight tags

for f in d["fields"]:                         # related courses (optional)
    print("course:", f["node_name"], "→", course_url(f["field_id"], **DEFAULTS))
```

Key fields in each `articles[]` item:
- `type`: `"article"` (topic) or `"keyword"` — **decides which link to build**.
- `id`: `entry_id` for a topic, `keyword_id` for a keyword.
- `article_name`: title.
- `matched_elements[].content`: the matched highlighted snippet, usable as a **summary** (contains `<em>` tags — strip before display).

### Want a fuller summary / full text

`/search/universal` returns search snippets. For the body, fetch the detail by type:

```python
# Full topic (article) body
doc = data(requests.post(f"{BASE}/article", headers=H,
                         json={"entry_id": "<entry_id>", **DEFAULTS}))["document"]

# Full keyword body
doc = data(requests.post(f"{BASE}/keyword", headers=H,
                         json={"keyword_id": "<keyword_id>", **DEFAULTS}))["document"]

# Common fields in doc:
#   article_name    title
#   seo_description one-line summary (best for an abstract)
#   definition      definition
#   key_points      key points
#   main_content    body (Markdown)
#   applications    applications
```

> ⚠️ `article` / `keyword` occasionally return `code=250002` (content is generated on demand and not yet available for that language/style). In that case **fall back** to the `matched_elements` snippet from search, or retry with another `style` / `language`.

---

## Task 2: Browse what fields and courses exist

**Browse by discipline**: get majors and levels first, then list the courses under a level.

```python
# 1) Top structure: major → level
ml = data(requests.post(f"{BASE}/major_levels", headers=H, json={**DEFAULTS}))
for m in ml["majors"]:
    levels = ", ".join(l["name"] for l in m["levels"])
    print(m["name"], "| levels:", levels)

# 2) List the courses (fields) under some levels
level_ids = [lv["node_id"] for m in ml["majors"] for lv in m["levels"]]
lf = data(requests.post(f"{BASE}/level_fields", headers=H,
                        json={"node_ids": level_ids[:5], "page_num": 1, "page_size": 20, **DEFAULTS}))
for it in lf["items"]:
    f = it["field"]
    print(f'{it["major"]["name"]} / {it["level"]["name"]} / {f["name"]}'
          f' ({it.get("topic_count", 0)} knowledge points) →', course_url(f["field_id"], **DEFAULTS))
```

- `major_levels` returns `majors[]`, each with `name` and `levels[]` (`name` + `node_id`).
- `level_fields` returns paged `items[]` (`total` = count); each has `major` / `level` / `field`, `topic_count`, and a few sample `topics`; the `field` carries `node_id`, `name`, `seo_title` and **`field_id`**.

> For a clickable **course link**, use `field.field_id` directly (`course_url(...)`, see [Building reading links](#building-reading-links)); for the chapter structure, pass `field_id` (or `field.node_id`) to `get_wiki_index` in Task 3.

---

## Task 3: Given a course, get chapters and the knowledge list

Two steps: resolve the course by name → fetch the whole chapter tree.

```python
# 1) Locate the course by name (to get field_id)
s = data(requests.post(f"{BASE}/search/universal", headers=H,
                       json={"text": "Solid State Physics", **DEFAULTS}))
field = s["fields"][0]                 # has node_id, field_id, node_name

# 2) Fetch chapter structure + knowledge points (pass field_id, or that field's node_id)
tree = data(requests.post(f"{BASE}/get_wiki_index", headers=H,
                          json={"field_id": field["field_id"], **DEFAULTS}))

def walk(nodes, depth=0):
    for n in nodes:
        print("  " * depth, f'[{n["node_type"]}]', n["node_name"])
        if n["node_type"] == "entry":          # leaf = one knowledge point (topic)
            print("  " * (depth + 1), "→", topic_url(n["entry_id"], **DEFAULTS))
        walk(n.get("children") or [], depth + 1)

walk(tree["wiki_indices"])
print("total knowledge points:", tree.get("entry_count"))
```

- The tree has 4 levels: `field` (course) → `category` → `chapter` → `entry` (knowledge point / topic).
- Each node has `node_id`, `node_type`, `node_name`; **leaf `entry` nodes additionally carry `entry_id`** (for the reading link), `snapshot` (an AI quick-look, great as a one-liner), and `seo_title`.
- The top level also returns `entry_count` plus `foundational_entry_count` / `core_entry_count` / `advanced_entry_count` (foundational / core / advanced counts).

---

## Task 4: Starting from a topic, get the knowledge graph

Expand outward from a center node (topic or keyword) to get its graph of related concepts.

```python
# 1) Find the center node's id (entry_id for a topic, keyword_id for a keyword)
s = data(requests.post(f"{BASE}/search/universal", headers=H,
                       json={"text": "superconductivity", **DEFAULTS}))
center_id = s["articles"][0]["id"]
# Or find nodes directly with the graph search:
# gs = data(requests.post(f"{BASE}/knowledge_graph/search", headers=H, json={"text": "entropy"}))
# center_id = gs["items"][0]["id"]

# 2) Fetch the graph (note: GET + query params)
g = data(requests.get(f"{BASE}/knowledge_graph", headers=H,
                      params={"id": center_id, **DEFAULTS}))

for n in g["nodes"]:                  # nodes
    link = build_read_url(n["node_type"], n["node_id"], **DEFAULTS)
    print(n["display_name"], f'({n["node_type"]}, depth {n["depth"]})', "→", link)

for e in g["relationships"]:          # edges
    print(e["src_node_id"], "──", e["relationship"], "──>", e["desc_node_id"],
          f'(weight {e["weight"]})')
```

- Input: `id` (required, the center node's `entry_id` or `keyword_id`), `language`, `style`, optional `skip_cross_domain` (true = same-domain only).
- `nodes[]`: `node_id` (which IS the `entry_id`/`keyword_id`, link it directly), `node_type` (`entry`/`keyword`), `display_name`, `description`, `field_name`, `major_name`, `depth` (center node = 0).
- `relationships[]`: `src_node_id` / `desc_node_id`, `relationship` (description), `relation_type`, `description`, `weight`, `evidence_count`, `is_bidirectional`, `is_cross_domain`.
- `domains[]`: disciplines involved (`major_node_id` + `major_name`).

### Drill into a single node / edge

```python
# Single node detail
node = data(requests.get(f"{BASE}/knowledge_graph/node", headers=H,
            params={"id": center_id, "node_type": "entry", **DEFAULTS}))

# Single relationship detail (with supporting evidences)
rel = data(requests.get(f"{BASE}/knowledge_graph/relationship", headers=H,
           params={"src_node_id": "A", "desc_node_id": "B", "relation_id": "R", **DEFAULTS}))
```

---

## Action reference

| Task | Method + path | Purpose |
|------|---------------|---------|
| Search (preferred) | `POST /search/universal` | One term → topics + keywords + related courses |
| Topic body | `POST /article` | Article body by `entry_id` (or `node_id`) |
| Keyword body | `POST /keyword` | Keyword content by `keyword_id` |
| Discipline tree | `POST /major_levels` | All majors and their levels |
| Course list | `POST /level_fields` | Courses under some levels (paged) |
| Course structure | `POST /get_wiki_index` | Chapter tree + knowledge points (leaves carry `entry_id`) |
| Knowledge graph | `GET /knowledge_graph` | Expand a graph from a topic/keyword |
| Graph · node | `GET /knowledge_graph/node` | Single node detail |
| Graph · edge | `GET /knowledge_graph/relationship` | Single relationship detail (with evidence) |
| Graph · search | `POST /knowledge_graph/search` | Find nodes in the graph by text |
| (optional) Stats | `GET /info` | Total topics / keywords |
| (optional) Name search | `POST /search_index_name` | Search nodes by name (e.g. only `field`) |

> Every response is wrapped in `{"code":0,"data":{...}}`; unwrap with `data()` above. `GET` endpoints use `params=`, `POST` endpoints use `json=`.

---

## Building reading links

Whenever you show a topic / keyword / course to a human, attach the reading link on `https://www.bohrium.com` (NOT the API host).

Rules:
- **Language prefix**: `zh-CN` → no prefix; `en-US` → add `/en`.
- **Style segment**: lowercase, `Feynman → feynman`, `Hardcore → hardcore`.
- **URL-encode** dynamic ids.

| Type | Pattern | id source |
|------|---------|-----------|
| Topic | `{prefix}/sciencepedia/{style}/{entry_id}` | search `type=article` `id`; course-tree leaf `entry_id`; graph `node_type=entry` `node_id` |
| Keyword | `{prefix}/sciencepedia/{style}/keyword/{keyword_id}` | search `type=keyword` `id`; graph `node_type=keyword` `node_id` |
| Course / Field | `{prefix}/sciencepedia/field/{style}/{field_id}` | `/search/universal` `fields[].field_id`, or `/level_fields` `items[].field.field_id` |

```python
def _site(language):
    return "https://www.bohrium.com/en" if language == "en-US" else "https://www.bohrium.com"

def topic_url(entry_id, language="en-US", style="Feynman"):
    return f"{_site(language)}/sciencepedia/{style.lower()}/{quote(str(entry_id))}"

def keyword_url(keyword_id, language="en-US", style="Feynman"):
    return f"{_site(language)}/sciencepedia/{style.lower()}/keyword/{quote(str(keyword_id))}"

def course_url(field_id, language="en-US", style="Feynman"):
    return f"{_site(language)}/sciencepedia/field/{style.lower()}/{quote(str(field_id))}"

def build_read_url(node_type, node_id, language="en-US", style="Feynman"):
    # node_type "keyword" -> keyword page; otherwise (article/entry) -> topic page
    if node_type == "keyword":
        return keyword_url(node_id, language, style)
    return topic_url(node_id, language, style)
```

Examples:
```text
Topic   (en, Feynman):   https://www.bohrium.com/en/sciencepedia/feynman/<entry_id>
Keyword (zh, Feynman):   https://www.bohrium.com/sciencepedia/feynman/keyword/<keyword_id>
Course  (en, Hardcore):  https://www.bohrium.com/en/sciencepedia/field/hardcore/<field_id>
```

---

## curl examples

```bash
AK="$BOHR_ACCESS_KEY"
BASE="https://open.bohrium.com/openapi/v2/literature-sage/wiki_v2"

# Unified search
curl -s -X POST "$BASE/search/universal" \
  -H "Authorization: Bearer $AK" -H "Content-Type: application/json" \
  -d '{"text":"graphene","language":"en-US","style":"Feynman"}' \
  | jq '.data.articles[] | {type, id, article_name}'

# Knowledge graph (GET + query)
curl -s -G "$BASE/knowledge_graph" \
  -H "Authorization: Bearer $AK" \
  --data-urlencode "id=<entry_id_or_keyword_id>" \
  --data-urlencode "language=en-US" --data-urlencode "style=Feynman" \
  | jq '.data | {nodes: (.nodes|length), edges: (.relationships|length)}'
```

---

## Response standards

- **Search**: just give a list of "title + one-line summary + reading link"; do not explain the "topic vs keyword" difference to the user.
- **Explain a concept**: definition → intuition → key points, with the entry's reading link.
- **Course structure**: present as chapter → section → knowledge point, each point linked; you may group by foundational / core / advanced (using `*_entry_count`).
- **Knowledge graph**: describe the center node first, then list the most relevant edges (by `weight`); expand with `node` / `relationship` detail when needed.
- Always attach the reading link for any entry you mention. If the requested `language`/`style` returns nothing, fall back in order: same language with the other style → `zh-CN`+`Feynman` → `en-US`+`Feynman`, and say you switched.
- Keep API failures transparent; on empty results, suggest synonyms and one alternate language/style.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| No matches | Term not in the index | Try synonyms; or use `/search_index_name` with broader `node_types` |
| `article`/`keyword` returns `250002` | Content not yet generated for that language/style | Fall back to the search snippet, or retry with another `style`/`language` |
| All-English (or all-Chinese) results | Wrong `language` | Set `"language":"en-US"` or `"zh-CN"` |
| Empty knowledge graph | `id` is not a valid `entry_id`/`keyword_id`, or the node has no neighbors | First resolve the id via `/search/universal` or `/knowledge_graph/search` |
| Built a reading link on `open.bohrium.com` | Mixed up API host and page host | Page links use `https://www.bohrium.com` (see [Building reading links](#building-reading-links)) |

## Pairs well with

- **bohrium-sciencepedia** for a baseline explanation of a concept → **bohrium-paper-search** to go deep
- **bohrium-sciencepedia** to browse the discipline tree (`major_levels`) → pick a course → **bohrium-scholar-search** to find leading researchers
