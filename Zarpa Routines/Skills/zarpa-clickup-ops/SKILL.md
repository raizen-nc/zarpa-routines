---
name: zarpa-clickup-ops
description: >
  Operational skill for the Zarpa Copywriter agent in Claude Routines. Use FIRST
  on every task — explains how to receive a ClickUp webhook payload, parse the brief,
  identify content type from tags, and save results back without erasing the original brief.
  Contains the full ClickUp workspace map (IDs, tag meanings, status workflow).
---

# zarpa-clickup-ops — Operational Guide for the Zarpa Copywriter

Read this skill every time before starting work on a content task.

---

## 1. Webhook Trigger — What You Receive

When ClickUp Automation fires (tag `write now` added), you receive:
- `task_id` — ClickUp task ID (e.g. `869d5rfj5`)
- optionally: `task_name`, `list_id`, `tags`

First action: **fetch the full task**
```
clickup_get_task(task_id="<id>", detail_level="detailed")
```
If too large, retry with `detail_level="summary"`.

---

## 2. Workspace Map

### Content Lists
| List name | List ID | Content type |
|-----------|---------|--------------|
| zarpa io | `901211514850` | Blog articles + service pages |
| Контент | `901213337774` | Social media posts |

### Reference Lists
| List name | List ID | Use for |
|-----------|---------|---------|
| Stuff | `901211025182` | Specialist names/credentials (E-E-A-T) |
| SEO AEO ZARPA | `901215378522` | Keyword strategy materials |
| Google search console | `901215702849` | Organic traffic data |

---

## 3. Tag System

| Tag | Meaning |
|-----|---------|
| `write now` | Trigger — generate content |
| `blog` | Blog article → use zarpa-blog-writer |
| `lending` | Service/landing page → use zarpa-page-writer |
| `marketing` | Marketing page → use zarpa-page-writer (marketing mode) |
| `img` | Generate cover image — skip if absent |
| `send to discord` | Handled by n8n separately — ignore |
| `draft` | Sparse brief — use judgment to fill gaps |

**Decision logic:**
```
blog tag       → zarpa-blog-writer
lending tag    → zarpa-page-writer
marketing tag  → zarpa-page-writer (marketing mode)
Контент list   → zarpa-social-writer
no type tag    → infer from task name prefix (📝 [БЛОГ] etc.) or default to blog
```

---

## 4. Parsing the Task Description (Quill Delta)

ClickUp stores rich text as Quill Delta JSON: `{"ops": [...]}`.

Each op is either:
- `{"insert": "text"}` → plain text, include it
- `{"insert": "text", "attributes": {...}}` → include text, ignore attributes
- `{"insert": {"divider": true}}` → skip
- `{"insert": {"table-embed": {...}}}` → parse as table (see below)

**Python extraction:**
```python
import json

def quill_to_text(quill_json_string):
    data = json.loads(quill_json_string)
    result = []
    for op in data.get("ops", []):
        insert = op.get("insert", "")
        if isinstance(insert, str):
            result.append(insert)
        elif isinstance(insert, dict) and "table-embed" in insert:
            result.append(parse_table(insert["table-embed"]))
    return "".join(result)

def parse_table(table_data):
    cells = table_data.get("cells", {})
    rows = table_data.get("rows", [])
    cols = table_data.get("columns", [])
    lines = []
    for r_idx in range(1, len(rows)+1):
        row_cells = []
        for c_idx in range(1, len(cols)+1):
            key = f"{r_idx}:{c_idx}"
            cell = cells.get(key, {})
            cell_text = "".join(
                op.get("insert","") for op in cell.get("content",[])
                if isinstance(op.get("insert",""), str)
            ).strip()
            row_cells.append(cell_text)
        lines.append(" | ".join(row_cells))
    return "\n".join(lines) + "\n"
```

**Key sections to extract from the brief:**

| Section heading | Contains |
|----------------|----------|
| Мета | Goal of the piece |
| URL | Target slug (e.g. zarpa.io/blog/shrot-terapiya) |
| Ключові слова / Семантичне ядро | Keywords, search intent, priority |
| Конкуренти | Competitor gaps to exploit |
| Обсяг та структура | Word count + section outline |
| Автор | Author name for E-E-A-T |

---

## 5. Saving Content Back

**Always save as a comment — never overwrite the task description.**

```
clickup_create_task_comment(task_id="<id>", comment_text="<content>")
clickup_update_task(task_id="<id>", status="in progress")
```

**Comment format:**
```markdown
## ✍️ Згенерований контент — [тип]

> Автор: [ім'я або "Команда Zarpa"]
> Тип: [blog / lending / marketing / social]
> Дата: [YYYY-MM-DD]

---

[повний текст]

---

**SEO Title:** [до 60 символів]
**Meta Description:** [до 155 символів]
```

---

## 6. Checklist Before Writing

- [ ] Task fetched via clickup_get_task
- [ ] Quill Delta parsed, brief extracted
- [ ] Content type identified from tags or list
- [ ] Target URL found
- [ ] Keywords extracted
- [ ] Section structure identified
- [ ] Author identified (or defaulted to "Команда Zarpa")
- [ ] Correct writing skill loaded

---

## 7. Error Handling

| Situation | Action |
|-----------|--------|
| No type tag (blog/lending/marketing) | Infer from task name prefix or list |
| Empty description | Write from task name alone |
| Author not found in Stuff list | Use "Команда Zarpa", add note |
| Task in Контент, no format specified | Default to Instagram post |
| Quill Delta table unreadable | Extract keywords from H2 headings instead |
