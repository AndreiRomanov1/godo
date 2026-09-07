# adapters.md — различия клиентов ИИ

Ядро (`RULES.md`) одно для всех. Клиенты различаются: (а) как подключить правила, (б) есть ли субагенты и как они называются, (в) есть ли MCP (Godot / Blender / CBM). Перед работой определи клиент и возможности, недостающее — fallback из `RULES.md §1`.

| Возможность | Grok Build | Cursor | Claude Code | Codex / Gemini CLI |
|---|---|---|---|---|
| Файл правил | читать `RULES.md` по пути из пинга | `.cursor/rules/*.mdc`, `AGENTS.md` | `CLAUDE.md` (+ `@импорты`) | `AGENTS.md` / `GEMINI.md` |
| Субагенты | `spawn_subagent` (explore / plan / general-purpose + roles/personas) | `Task` (explore / generalPurpose / computerUse) | встроенные субагенты | обычно нет → последовательно |
| MCP Godot / Blender / CBM | локальная конфигурация MCP | `.cursor/mcp.json` (локально); в облачных агентах локальных MCP нет | конфигурация MCP клиента | по конфигурации клиента |
| Vision | vision‑модель клиента по скрину | чтение изображений | чтение изображений | по клиенту |

---

## 1. Grok Build (TUI)

Субагенты включены по умолчанию. Каталог ролей/персон — `/config-agents` (alias `/agents`, персоны `/personas`). Доки — `GROK_DOCS/16-subagents.md`, бандл — `GROK_BUNDLE`.

### 1.1 `spawn_subagent`

| Параметр | Смысл |
|---|---|
| `prompt` | Полное ТЗ ребёнку: цель, пути, что нельзя трогать, формат ответа, язык |
| `description` | Ярлык 3–5 слов |
| `subagent_type` | `explore` (read‑only, поиск/обзор), `plan` (read‑only, архитектура и шаги), `general-purpose` (полный toolset) или user/project agent из `/config-agents` |
| `background` | `true` — параллель; результат через `get_command_or_subagent_output` |
| `capability_mode` | `read-only` / `read-write` / `execute` / `all` — минимально достаточный |
| `isolation` | `none` или `worktree` (изолированные правки git) |
| `resume_from` | продолжить завершённого субагента того же типа |
| `cwd` | рабочая папка (не вместе с `worktree`) |

Ограничения: спавнит только parent (вложенность 1); не для микрозадач; не для диалога с пользователем; Godot MCP и Blender MCP координирует parent — дети не дерутся за editor state.

### 1.2 Roles и personas — как использовать смысл

Роли/персоны бандла — поведенческие слои; в `prompt` явно пиши роль и контракт («ты reviewer: только notes с severity и file:line, сцену не правишь»).

| Роль / персона | Для чего |
|---|---|
| `explore`, `quick-search`, `researcher` | обзор проекта, точечный lookup, глубокое исследование базы с цитатами путей |
| `plan` | план фичи/экрана/уровня до сборки |
| `implementer` | кусок по notes/ТЗ (файлы; MCP — только по поручению parent) |
| `reviewer` | ревью кода и структуры сцен → structured notes, без самовольных фиксов |
| `test-writer` | тесты (GUT / сценарии / asserts) + прогон |
| `security-auditor` | ввод, сейвы, читы, RPC — реальные уязвимости |
| `design-doc-writer`, `design-doc-reviewer` | design doc большой системы и его рецензия |

Актуальный список всегда в `/config-agents`; custom agents пользователя — использовать.

### 1.3 Когда обязан звать субагента

| Ситуация | Кого |
|---|---|
| «Сделай экран/уровень/систему» с нуля | `plan` → parent MCP (+ `general-purpose` на куски) |
| Не знаешь структуру проекта | `explore` / `quick-search` с CBM в prompt |
| Нужен урок/книга/паттерн | `explore` или `general-purpose` read‑only, пути из `paths.md` |
| Несколько независимых кусков (меню + player + save) | несколько `background` субагентов, разные файлы, merge у parent |
| После крупного кода | `reviewer` |
| Геймплейная логика готова | `test-writer` |
| Сейвы, читы, сеть | `security-auditor` |
| Сложный референс | `general-purpose`: «UI‑иерархия + diff‑чеклист»; сборка MCP — parent |
| Пачка анимаций через kimodo.cpp | `general-purpose` с `execute`: генерация + экспорт GLB по `tools/kimodo-cpp.md`; импорт и `AnimationTree` — parent |

### 1.4 Конвейеры

```text
A. UI‑экран по фото:   parent контекст → explore (theme/шрифты/похожие меню) → gp (разбор референса → иерархия Control + spacing/colors) → parent MCP + save + скрин → vision loop (≤3) → reviewer (опц.)
B. Геймплейная фича:   plan → parent/implementer (скрипты + сигналы; узлы в сцене) → test-writer → reviewer → parent play_scene smoke
C. Не знаю API:        параллельно explore (CBM + код) и gp (txt/books) → parent синтез → MCP/код
D. Персонаж + оружие:  plan → gp/execute: kimodo.cpp генерация + export_glb (+ Blender MCP чистка/ретаргет при необходимости) → parent: импорт в Godot, BoneAttachment3D, AnimationTree, save → скрин/видео feel
```

### 1.5 Минимум в промпте субагенту

1. Цель одной фразой. 2. Пути проекта/сцен/референсов (из `paths.md`). 3. «Читай `RULES.md`: сцена‑first, структура папок/нод, no runtime level‑spawn». 4. Что можно / нельзя (сцену X не трогать, ничего не качать, read‑only…). 5. Формат ответа (пути, дерево нод, diff, шаги). 6. Язык — русский.

### 1.6 Запреты

Не игнорировать субагентов на больших задачах; не спамить 10 детей на пустяк; не отдавать двум writer’ам одну сцену/`.blend`; не считать работу ребёнка сделанной в редакторе, пока parent не сохранил через MCP; не выдумывать «субагент уже отревьюил», если spawn не было.

---

## 2. Cursor

- **Правила:** положи `RULES.md` как `AGENTS.md` в корень игрового репо **или** `.cursor/rules/godot-editor-first.mdc` с заголовком:

```text
---
description: Godot 4 — игра в редакторе, структура, UI-стиль, game feel, анимации
globs: ["**/*.gd", "**/*.tscn", "**/*.tres", "**/project.godot"]
alwaysApply: true
---
```

  Модули (`modules/*.md`, `tools/*.md`) оставь рядом — агент читает их по ссылкам из ядра.
- **Субагенты:** `Task` с `subagent_type`: `explore` (поиск/обзор), `generalPurpose` (реализация, ревью, тесты), `computerUse` (ручной тест GUI, скрины игры). Роли reviewer/test‑writer задаются контрактом в промпте.
- **MCP:** Godot MCP Pro и Blender MCP подключаются через `.cursor/mcp.json` локально. **Облачный агент Cursor не видит локальные MCP и редактор** → это всегда `FALLBACK` (файлы на диске), о чём агент обязан сказать.
- Vision: агент читает изображения напрямую (скрины из `get_editor_screenshot` / `get_viewport_screenshot` сохранять в файл и читать).

## 3. Claude Code / Codex / Gemini CLI

- Правила: `CLAUDE.md` (с `@agent-rules/modules/...` импортами по необходимости) / `AGENTS.md` / `GEMINI.md`. Содержимое — `RULES.md` без изменений.
- Субагенты: если есть — по тем же триггерам (`RULES.md §9`); нет — работать последовательно и явно написать это в отчёте.
- MCP: по конфигурации клиента; нет Godot MCP → fallback на диске с предупреждением о конфликте с открытым редактором (`RULES.md §10`).
