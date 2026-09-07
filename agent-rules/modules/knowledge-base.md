# knowledge-base.md — «не знаешь → найди»

Пути — только через `paths.md`. Порядок: проект → база курса → official docs → интернет (если клиент даёт поиск). Память модели — последний источник, и она подписывается в отчёте как «предположение».

## 1. codebase‑memory (CBM)

```text
list_projects
→ index_repository(...)          # если проекта ещё нет
→ search_graph / search_code / get_architecture / query_graph / get_code_snippet
```

- Проекты: `CBM_PROJECTS` (`godot-course` — база уроков; проект игры — код и сцены).
- После крупных добавлений — reindex / `detect_changes`.
- CBM ≠ Godot MCP: CBM читает и ищет, MCP меняет редактор.
- Tools не видны → рестарт клиента; CLI — `CBM_BIN cli <tool> ...`.
- Тяжёлый обход графа — в explore/researcher субагент (если есть).

## 2. Видеоуроки (YouTube → txt)

`COURSE_ROOT`, бриф — `COURSE_BRIEF`, пайплайн — там же.

1. Найти txt (`rg -i "<тема>" LESSONS_TXT`) или выгрузить `FETCH_LESSON` → 2. прочитать → 3. реализовать в **текущем** проекте.
4. Не копировать txt в репо игры. 5. Не качать видео/аудио без просьбы.

## 3. Книги и official docs

`BOOKS_INDEX` → `BOOKS_TXT/<slug>/full.txt`. Official docs — `BOOKS_TXT/godot-official-docs-stable/full.txt` (Theme, StyleBoxFlat, Tween, AnimationTree, BoneAttachment3D, Skeleton3D, BoneMap). Companion‑код книг — reference, не «текст книги». Чужие книги и уроки не коммитить.

## 4. Якоря по темам

| Тема | Где искать |
|---|---|
| Theme, Main Menu, стиль UI | `LESSONS_TXT/*Theme*`, `*Main_Menu*`, `*Style*UI*` |
| HUD | `*HUD*`, `*Health_HUD*` |
| Polish / juice / shake | `*Polish*`, `*Juice*`, `*Camera_Juice*`, `*Camera_Shake*`, `*Screen_Shake*`, `*Visual_Effects*`, `*Level_Up_Polish*` |
| Lighting 2D/3D | `*Lighting*`, `*Environment*` |
| Анимация 3D, скелеты, ретаргет | `*Animation*`, `*Skeleton*`, `*Retarget*`, `*AnimationTree*`; official docs: Retargeting 3D Skeletons, Using AnimationTree |
| kimodo.cpp | `KIMODO_ROOT/README.md`, `docs/` (гайд для Unreal переносится на Godot почти 1‑в‑1), `tools/kimodo-cpp.md` |
| Blender MCP | схема инструментов сервера (`list_tools`), `tools/blender-mcp.md` |
| Официальный API | `BOOKS_TXT/godot-official-docs-stable/full.txt`, `BOOKS_INDEX` |

## 5. Правило «не выдумывай»

| Не знаешь | Действие |
|---|---|
| API Godot / паттерн | official docs → книга → урок (субагент, если есть) |
| «Где меню в этой игре?» | CBM + `get_scene_tree` + листинг `scenes/ui/` |
| Сигнал / call graph | CBM `search_graph` |
| Имена MCP‑инструментов и параметры | схема сервера, не память |
| Как выглядит «нормально» | референс + vision, база стиля |
| Как двигается персонаж | kimodo.cpp‑генерация, не ручные ключи «из головы» |
| Лицензия ассета/модели | карточка модели / страница ассета; неизвестно → в релиз не брать |
