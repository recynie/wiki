# AGENTS.md

This is a Quartz 4 + Obsidian knowledge base. Agents operate on the `wiki/` and `blogs/` content folders to write, revise, and extend content.

---

## 1. Build & Deploy

This is a **content repository** — there are no build, lint, or test commands.

- **Quartz config**: `/home/aroma/Documents/sites/quartz/quartz.config.ts`
- **Preview locally**: Install Quartz 4 per [Quartz Docs](https://quartz.jzhao.xyz/) and run `quartz build --serve`
- **Deploy**: Push to GitHub; CI/CD handles the Quartz build

### Ignored Directories
The following directories are **ignored** during publishing (defined in quartz config):

| Directory | Purpose |
|-----------|---------|
| `private` | Private/hidden content |
| `templates` | Content templates |
| `.obsidian` | Obsidian vault configuration |
| `assets` | Static assets (handled separately) |

> **Note**: Files in these directories will not appear in the published site.

### Directory & File Naming
- Use **kebab-case** for directory names: `breadth-first-search/`, `monotonous-stack/`
- One topic per directory; files within represent subtopics or related content

---

## 3. Frontmatter Conventions

Every markdown file **must** include frontmatter:

```yaml
---
title: <Descriptive Title>
date: <YYYY-MM-DD>
description: <Brief description>
tags:
  - <tag1>
  - <tag2>
draft: <true|false>
permalink:
---
```

### Rules
- `title`: Basically, Chinese preferred for Chinese content; English for English content. Do not use bilingual title. Keep title short and clear. English terms are ok if their translations are not commonly used.
- `date`: Date of creation or major revision
- `description`: One-line summary
- `tags`: Lowercase kebab-case, 1-3 most relevant tags
- `draft`: `true` = work-in-progress; `false` = finalized
- `permalink`: Leave empty (`permalink:`) — Quartz generates it from file path

### Example

```yaml
---
title: BFS(Breadth-First Search)
date: 2026-03-13
description: 广度优先搜索（Breadth-First Search, BFS）
tags:
  - bfs
  - graph
draft: false
permalink:
---
```

---

## 4. Wiki-Link Conventions

Use Obsidian-style wiki links for internal navigation:

| Pattern    | Meaning                             |                                       |
| ---------- | ----------------------------------- | ------------------------------------- |
| `[[path    | Display Text]]`                     | Link to file with custom display text |
| `[[path]]` | Link using filename as display text |                                       |

### Path Resolution
- Relative to current file's directory
- Use `..` to go up: `[[../algorithms/breadth-first-search|BFS]]`
- Keep link paths short; avoid absolute paths

### Examples from existing content

```markdown
BFS和[[depth-first-search|DFS]]是图搜索中最基础的算法。

[[../data-structure/monotonous-stack|单调栈]]

[[../CCF-CSP/CSP202506B|机器人复建指南]]
```

---

## 5. Code Block Conventions

### Language Identifier
Always specify the language after the opening backticks:

```cpp
```cpp
#include<bits/stdc++.h>
using namespace std;
...
```

### Supported Languages
- `cpp` — C++ (most common in this repo)
- `python`
- `bash`
- `sql`
- etc.

### C++ Style Guidelines
- Include what you need; commonly: `#include <bits/stdc++.h>`
- Use `ios::sync_with_stdio(false); cin.tie(nullptr);` for faster I/O
- Prefer `vector`, `queue`, `stack`, `pair` from STL
- Use `int64_t` or `long long` for large integers

```cpp
int main(){
    ios::sync_with_stdio(false);cin.tie(nullptr);
    int n, m;
    cin >> n >> m;
    vector<vector<int>> grid(n, vector<int>(m, 0));
    // ...
}
```

---

## 6. Mathematical Notation

Use LaTeX syntax inside `$...$` (inline) or `$$...$$` (block):

```markdown
设当前遍历到数组的第 $i$ 个元素 $A[i]$，栈为 $S$。

$$
h(t_{i+1}) = h(t_i) + \Delta t \cdot f(h(t_i), t_i)
$$
```

---

## 7. Reference Notation

Use superscript footnotes for references:

```markdown
BFS和DFS是图搜索中最基础的算法。[^1]
```

**脚注放置位置**：脚注内容应放在相应段落的**正后方**（而非文件末尾的单独章节）。这样在阅读 md 源文件时可以快速看到脚注内容，在网页渲染时脚注会自动聚集到页面底部。[^1]

[^1]: 本段来自[BFS（图论） - OI Wiki](https://oi-wiki.org/graph/bfs)

---

## 8. Content Style

### Language
- **Chinese content**: Use simplified Chinese, Chinese punctuation (，。、：；？！""『』)
- **English content**: Use English throughout, English punctuation

### Headings
- Use `##` for main sections, `###` for subsections
- Avoid excessive heading levels (max 3-4 levels)

### Voice and Tone
- Technical, precise, educational
- Explain "why" not just "what"
- Include practical examples and applications

### Content Structure (Recommended)
1. **Definition/Introduction** — What is this?
2. **Core Properties/Principles** — How does it work?
3. **Algorithm/Implementation** — Code template or detailed steps
4. **Applications/Examples** — Real problems, related topics
5. **References** — Sources, further reading

---

## 9. Draft vs. Finalized Content

| State | Meaning |
|-------|---------|
| `draft: true` | Incomplete, may have errors, subject to change |
| `draft: false` | Reviewed, accurate, ready for publication |

When revising content:
- Update `draft: false` only after thorough review
- Add `draft: true` to mark work-in-progress content

---

## 10. Tag Conventions

Use lowercase kebab-case. Common tags in this repo:

| Tag | Meaning |
|-----|---------|
| `bfs` | Breadth-First Search |
| `dfs` | Depth-First Search |
| `dynamic-planning` | Dynamic Programming |
| `monotonous-stack` | Monotonic Stack |
| `graph` | Graph algorithms |
| `simulation` | Simulation problems |

---

## 11. Related Agent Instructions

See also:
- `prompts4agent/code of coding.zh.md` — General coding guidelines
- `prompts4agent/code of coding.en.md` — English version
- `prompts4agent/researcher.md` — Research paper analysis guidelines

---

## 12. Scaffolds

When creating new content, use templates from:

- `blogs/_scaffolds/quartz.md` — General page template
- `blogs/_scaffolds/quartz-csp.md` — CCF-CSP problem template

---

*Last updated: 2026-03-19*
