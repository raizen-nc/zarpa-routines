---
name: zarpa-clickup-ops
description: >
  Operational guide for Zarpa content agents (Research Routine + Writing Routine).
  Load FIRST on every task — before any other skill.
  CRITICAL GOTCHAS: write_now = custom field checkbox (NOT a tag);
  save content by appending to description (NOT comments, NOT overwrite);
  final status = "check and correction" (NOT "in progress").
  Contains: workspace map with all 4 list IDs, all custom field IDs,
  two-agent workflow, content type routing, Quill Delta parser, status workflow.
---

# zarpa-clickup-ops — Операційний гід для агентів Zarpa

Читай цей Skill ПЕРШИМ перед будь-якою іншою дією.

---

## ⚠️ GOTCHAS — Прочитай перед усім іншим

Ці факти не очевидні і викликають збої якщо їх пропустити:

1. **`write now` — це CUSTOM FIELD (чекбокс), НЕ тег.**
   ID: `3eb51e48-1787-4146-b84f-eac2fffb65b2`
   Встановлюється через `clickup_update_task` з параметром `custom_fields`.
   Теги не використовуються для тригерування агентів у цьому pipeline.

2. **Весь контент (дослідження + текст) APPENDING до description таску.**
   Ніколи не перезаписуй description. Ніколи не пиши в коментарі (comments).
   Читай поточний опис → додай новий блок в кінці → збережи повний опис.

3. **Статус після написання тексту = `check and correction`**, не "in progress", не "review".

4. **Сторінки "Ми лікуємо" (list `901218584743`) → zarpa-page-writer We Treat Mode.**
   Це НЕ landing page і НЕ marketing page. Окрема структура для хвороб/станів.

5. **Research Routine і Writing Routine — два окремих агенти з різними тригерами.**
   Research Routine закінчує роботу, встановлюючи `write now = true` → це тригер для Writing Routine.

6. **ClickUp зберігає опис як Quill Delta JSON** — перетворюй у plain text перед читанням.

---

## 1. Two-Agent Workflow

```
RESEARCH ROUTINE (Claude Sonnet)
─────────────────────────────────
Тригер: статус таску змінюється на "seo research"
        АБО custom field research_started = true
Читає:  zarpa-seo-researcher (цей Skill завантажує Research Routine)
Робить: 15-25 запитів WebSearch + Apify, семантичне ядро, аналіз конкурентів
Зберігає: додає "## 🔍 SEO Research" блок до description таску
Завершує: статус → "writing text"
          custom field write_now (ID: 3eb51e48-1787-4146-b84f-eac2fffb65b2) = true

           ↓ ClickUp Automation фіксує зміну write_now → Webhook

WRITING ROUTINE (Claude Opus)
─────────────────────────────
Тригер: custom field write_now = true (через ClickUp Automation Webhook)
Читає:  zarpa-brand-voice + zarpa-seo-aeo-framework + zarpa-page-writer (або zarpa-blog-writer)
        Читає task description — там є "## 🔍 SEO Research" блок від Research Routine
Робить: Пише повний SEO/AEO текст на основі дослідження
Зберігає: додає "## ✍️ Текст сторінки" блок до description таску
           заповнює custom field img_prompt (ID: 07c498a0-9985-44c3-bcdd-5ba0a40b16be)
Завершує: статус → "check and correction"
          write_now → false (скидання для можливого повторного запуску)
```

---

## 2. Отримання Таску

При будь-якому тригері — перша дія завжди:
```
clickup_get_task(task_id="<id>", detail_level="detailed")
```
Якщо відповідь занадто велика — спробуй `detail_level="summary"`.

З відповіді витягни:
- `description` — Quill Delta JSON, розпарсити (секція 5)
- `status.status` — поточний статус
- `list.id` — визначає тип контенту (секція 4)
- `custom_fields` — поточні значення полів

---

## 3. Workspace Map — Усі Lists

### Content Lists
| Назва списку | List ID | Тип контенту | Writing Skill |
|-------------|---------|--------------|---------------|
| Статичні маркетингові сторінки | `901211514850` | Service/landing pages | zarpa-page-writer (Landing Mode або Marketing Mode) |
| Сторінки "Ми лікуємо" | `901218584743` | Сторінки хвороб/станів | zarpa-page-writer **(We Treat Mode)** |
| Сторінки блогу | `901218632544` | Блог-статті | zarpa-blog-writer |
| SEO AEO перевірка ефективності | `901215378522` | Моніторинг (read-only) | — |

### Reference Lists
| Назва списку | List ID | Призначення |
|-------------|---------|-------------|
| Stuff | `901211025182` | Імена спеціалістів, кваліфікація (для E-E-A-T) |
| Google search console | `901215702849` | Дані органічного трафіку |

**Workspace ID:** `9012154420`

---

## 4. Routing: Якого Writing Skill завантажувати

```
List ID 901218584743  →  zarpa-page-writer We Treat Mode   ← "Ми лікуємо"
List ID 901218632544  →  zarpa-blog-writer                 ← Блог
List ID 901211514850:
  тег lending         →  zarpa-page-writer Landing Mode
  тег marketing       →  zarpa-page-writer Marketing Mode
  немає тегу          →  інферуй з назви: [БЛОГ] → blog, [СТОРІНКА] → landing
```

---

## 5. Custom Fields — Повна Таблиця

| Field | ID | Тип | Хто встановлює | Значення |
|-------|-----|-----|----------------|---------|
| `write now` | `3eb51e48-1787-4146-b84f-eac2fffb65b2` | Checkbox | Research Routine (true) / Writing Routine (reset false) | Тригер Writing Routine |
| `img_prompt` | `07c498a0-9985-44c3-bcdd-5ba0a40b16be` | Text | Writing Routine | Інструкції Google Flow для Hero + секцій |
| `image_name` | — | Text | Юра вручну | Назви файлів зображень у репо |
| `creating_a_page` | — | Text | Агент при зміні статусу | Передача даних агенту генерації сторінки |
| `start_research` | — | DateTime | Юра вручну | Час початку дослідження |
| `research_started` | — | Checkbox | ClickUp Automation | Тригер Research Routine |

**Як встановити custom field через API:**
```
clickup_update_task(
  task_id="<id>",
  custom_fields=[{"id": "3eb51e48-1787-4146-b84f-eac2fffb65b2", "value": true}]
)
```

---

## 6. Парсинг Description (Quill Delta)

ClickUp зберігає rich text як Quill Delta JSON: `{"ops": [...]}`.

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
            rows = insert["table-embed"].get("rows", [])
            cols = insert["table-embed"].get("columns", [])
            cells = insert["table-embed"].get("cells", {})
            for r in range(1, len(rows)+1):
                row = []
                for c in range(1, len(cols)+1):
                    cell = cells.get(f"{r}:{c}", {})
                    text = "".join(
                        op.get("insert","") for op in cell.get("content",[])
                        if isinstance(op.get("insert",""), str)
                    ).strip()
                    row.append(text)
                result.append(" | ".join(row))
    return "\n".join(result) if isinstance(result[0], str) else "\n".join(
        [r if isinstance(r, str) else r for r in result]
    )
```

Якщо description вже plain text (не JSON) — використовуй безпосередньо.

**Секції, які шукати в description:**

| Розділ | Що містить |
|--------|-----------|
| `## 🔍 SEO Research` | Дослідження від Research Routine — Writing Routine читає це |
| Ключові слова / Семантичне ядро | Ключові слова, пошуковий інтент |
| Конкуренти | Аналіз конкурентів для контентних пробілів |
| Обсяг та структура | Кількість слів + структура секцій |
| Автор | Ім'я для E-E-A-T |
| URL / slug | Цільовий URL сторінки |

---

## 7. Збереження Контенту — Append до Description

**Правило:** append новий блок, зберігай весь попередній вміст.

```python
# 1. Отримай поточний опис
task = clickup_get_task(task_id="<id>")
current_desc = parse_description(task)  # перетвори Quill Delta у plain text

# 2. Додай новий блок
new_content = current_desc.rstrip() + "\n\n---\n\n## [NEW BLOCK]\n\n[content]"

# 3. Збережи
clickup_update_task(task_id="<id>", description=new_content)
```

**Формат блоку від Research Routine:**
```markdown
## 🔍 SEO Research

**Дата:** YYYY-MM-DD
**Тема:** [назва сторінки]

### Семантичне ядро
[таблиця ключових слів з частотністю та інтентом]

### Аналіз конкурентів
[структури сторінок конкурентів, контентні пробіли]

### Рекомендована структура сторінки
[H1, H2 секції, FAQ теми]

### Додаткові інсайти
[What People Ask, пов'язані теми, питання пацієнтів]
```

**Формат блоку від Writing Routine:**
```markdown
## ✍️ Текст сторінки

> Тип: [we-treat / blog / landing / marketing]
> Автор: [ім'я або "Команда Zarpa"]
> Дата: YYYY-MM-DD

---

[повний текст сторінки]

---

**SEO Title:** [до 60 символів, ключове слово + "Львів"]
**Meta Description:** [до 155 символів]

### img_prompt (Google Flow)
Hero: [опис 3D персонажу у стилі Zarpa для Hero секції]
[Додаткові секції якщо потрібно]
```

---

## 8. Статус Workflow

| Статус | Хто встановлює | Що відбувається |
|--------|---------------|----------------|
| `seo research` | Юра або ClickUp Automation | Research Routine запускається |
| `writing text` | Research Routine | Writing Routine чекає тригер write_now |
| `check and correction` | Writing Routine | Юра перевіряє контент, вносить правки |
| `creating a page` | Юра | Агент генерації Next.js сторінки |
| `correction` | Юра | Ручне виправлення дизайну |
| `ai checks` | Автоматично | SEO/AEO верифікація |
| `ready to publish` | Після ai checks | Схвалено до публікації |
| `published` | Після merge + deploy | Живе на zarpa.io |

---

## 9. Error Handling

| Ситуація | Дія |
|----------|-----|
| Description — Quill Delta JSON | Парсити через quill_to_text() |
| Description — plain text | Використовувати напряму |
| Тип контенту не визначається з тегів | Визначати за List ID (секція 4) |
| Автор не знайдений у Stuff list | Використати "Команда Zarpa" |
| Запис custom field не вдається | Повторити 1 раз; якщо знову помилка — залишити нотатку в description |
| Description занадто великий | Спочатку Research блок → потім Writing блок окремим оновленням |
