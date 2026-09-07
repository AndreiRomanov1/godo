# paths.md — пути и ресурсы конкретной машины

Единственное место, где живут пути. Правила ссылаются на имена из левой колонки.
Пути с кириллицей и пробелами ломают shell‑экранирование — сделай симлинки без них
(`ln -s ~/"Рабочий стол"/Курс/godot-уроки ~/godot-course`) и указывай симлинк.

## Игровой проект

| Имя | Значение | Комментарий |
|---|---|---|
| `GAME_ROOT` | `<путь к проекту Godot>` | папка с `project.godot` |
| `REFS` | `<GAME_ROOT>/refs/` или вне репо | референсы, промпты, сырые выводы генераторов |
| `GODOT_VERSION` | `4.4+` | минимум 4.3 (`TileMapLayer`) |

## База знаний (курс Godot)

| Имя | Значение | Комментарий |
|---|---|---|
| `COURSE_ROOT` | `~/godot-course` | симлинк на `~/Рабочий стол/Курс/godot-уроки/` |
| `COURSE_BRIEF` | `<COURSE_ROOT>/AGENT.md` | бриф курса, читать перед крупной работой по урокам |
| `LESSONS_TXT` | `<COURSE_ROOT>/txt/` | выгрузки видеоуроков |
| `BOOKS_PDF` | `<COURSE_ROOT>/книги/pdf/` | |
| `BOOKS_TXT` | `<COURSE_ROOT>/txt/books/` | текст книг и official docs (`godot-official-docs-stable/full.txt`) |
| `BOOKS_INDEX` | `<COURSE_ROOT>/книги/INDEX.md` | каталог книг |
| `FETCH_LESSON` | `<COURSE_ROOT>/bin/fetch_lesson.sh`, `fetch_batch.sh` | YouTube → txt; видео/аудио не качать без просьбы |

## codebase‑memory (CBM)

| Имя | Значение |
|---|---|
| `CBM_MCP` | MCP‑сервер `codebase-memory-mcp` |
| `CBM_BIN` | `~/.local/bin/codebase-memory-mcp` (CLI: `codebase-memory-mcp cli <tool> ...`) |
| `CBM_ALLOWED_ROOT` | `/home/<user>` |
| `CBM_PROJECTS` | `godot-course`, `<имя проекта игры>` |

## Godot MCP Pro

| Имя | Значение |
|---|---|
| `GODOT_MCP` | MCP‑сервер `godot-mcp-pro` (плагин редактора через WebSocket) |
| `GODOT_SKILL_LEVEL_EDITING` | `godot-mcp-direct-level-editing` (если установлен) |

## Blender

| Имя | Значение | Комментарий |
|---|---|---|
| `BLENDER_MCP` | MCP‑сервер Blender (`blender-mcp`, аддон BlenderMCP в Blender) | сессия Blender должна быть открыта и подключена |
| `BLENDER_BIN` | `<путь к blender>` | для прямого импорта `.blend` в Godot: Editor Settings → FileSystem → Import → Blender Path |
| `BLENDER_WORK` | `<REFS>/blender/` | рабочие `.blend`, не в `assets/` |

## kimodo.cpp (text‑to‑motion)

| Имя | Значение | Комментарий |
|---|---|---|
| `KIMODO_ROOT` | `<путь к клону kimodo.cpp>` | |
| `KIMODO_BIN` | `<KIMODO_ROOT>/build/release/kmd-generate` (Linux) / `build\Release\kmd-generate.exe` (Windows) | |
| `KIMODO_MOTION` | `<KIMODO_ROOT>/models/kimodo-soma-rp-v1.1-f32.gguf` | SOMA RP v1.1 — коммерческое ок |
| `KIMODO_TEXT` | `<KIMODO_ROOT>/generated/llm2vec-text-bundle` | текстовый энкодер |
| `KIMODO_EXPORT_GLB` | `<KIMODO_ROOT>/scripts/export_glb.py` | `.f32` → GLB (30 fps) |
| `KIMODO_EXPORT_BVH` | `<KIMODO_ROOT>/scripts/export_bvh.py` | `.f32` → BVH (для Blender) |
| `KIMODO_DEMO` | `go run ./demo -addr 0.0.0.0:8094` → `http://localhost:8094` | вывод `demo-output/<id>/animation.glb` |
| `KIMODO_OUT` | `<REFS>/motion/` | сырые выводы, не в `assets/` |
| `MOTION_PROMPTS` | `<этот репозиторий>/prompts/` | готовые промпты (АК‑74 и др.) |

## Клиент ИИ

| Имя | Значение |
|---|---|
| `AGENT_RULES_ROOT` | `<путь к папке agent-rules/>` |
| `GROK_DOCS` | `~/.grok/docs/user-guide/` (субагенты — `16-subagents.md`) |
| `GROK_BUNDLE` | `~/.grok/bundled/agents/`, `roles/`, `personas/` |
