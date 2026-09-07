# agent-rules — правила для ИИ‑агентов (Godot 4.4+, игра в редакторе)

Переработанная версия единого `rules.md`: ядро отделено от справочников, инструменты — от клиентов, пути машины — в один файл. Ядро читается всегда, остальное — по задаче.

```text
agent-rules/
  RULES.md                    # ядро: MUST / SHOULD / NEVER, ~200 строк — это и есть «правила»
  paths.md                    # все пути машины (курс, CBM, kimodo.cpp, Blender, проект)
  adapters.md                 # клиенты: Grok Build (субагенты), Cursor, Claude Code, Codex/Gemini
  modules/
    structure.md              # папки, дерево нод, naming, Input Map, git
    ui-style.md               # современный UI, anti‑proto, чеклист сдачи
    game-feel.md              # 4 фазы, каналы, ярусы интенсивности, FX‑архитектура
    animation-pipeline.md     # kimodo.cpp → (Blender) → Godot: скелет, ретаргет, AnimationTree, оружие
    reference-vision.md       # референсы, приоритеты P0–P3, стоп‑условия
    knowledge-base.md         # CBM, уроки, книги, «не выдумывай»
    report.md                 # шаблон отчёта + Definition of Done
  tools/
    godot-mcp-pro.md          # редактор Godot через MCP, fallback на диск
    blender-mcp.md            # Blender через MCP: ассеты, анимации, экспорт GLB
    kimodo-cpp.md             # text‑to‑motion: установка, CLI, экспорт, промпты
```

## Как подключить

1. Скопируй папку `agent-rules/` туда, откуда агент её прочитает (в игровой репозиторий или рядом), и заполни `paths.md`.
2. Подключи ядро по правилам клиента (`adapters.md`):
   - **Grok Build** — в первом сообщении сессии пинг из `RULES.md §14` с путём к папке;
   - **Cursor** — `RULES.md` → `AGENTS.md` в корне репо или `.cursor/rules/godot-editor-first.mdc` (frontmatter в `adapters.md §2`);
   - **Claude Code / Codex / Gemini CLI** — `CLAUDE.md` / `AGENTS.md` / `GEMINI.md`.
3. Модули и `tools/` оставь рядом: ядро ссылается на них относительными путями, агент читает по задаче.

## Что изменилось относительно исходного `rules.md`

- Убраны повторы (`save_scene`, «не выдумывай», субагенты, «не серый UI» встречались по 3–5 раз) — каждое правило теперь в одном месте.
- Grok‑специфика (`spawn_subagent`, roles/personas) вынесена в `adapters.md`; ядро работает в любом клиенте.
- Пути машины — только в `paths.md`; рекомендован симлинк без кириллицы и пробелов.
- Добавлено: приоритет при конфликте инструкций, интерактивный/автономный режим, стоп‑условия vision‑цикла (3 итерации), ярусы интенсивности feel, Input Map, git‑гигиена, защита правок пользователя, предупреждение о disk‑fallback при открытой сцене, `unique_name_in_owner`, единый шаблон отчёта.
- Новые разделы: анимационный пайплайн (kimodo.cpp → Blender → Godot, оружие на `BoneAttachment3D`, `AnimationTree`, ретаргет `BoneMap`), Blender MCP, kimodo.cpp, лицензии сгенерированных ассетов.
- Устранены противоречия: naming‑канон зафиксирован (сцены/классы PascalCase, остальное snake_case), «объекты в (0,0)» уточнено, Tween‑в‑коде vs «правится глазами» разведено (авторское — `AnimationPlayer`, процедурное — Tween с `@export`), убрана висячая ссылка на Whisper, версия Godot одна (4.4+).
