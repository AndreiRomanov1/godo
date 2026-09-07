# godot-mcp-pro.md — работа с редактором Godot через MCP

Архитектура: AI ← MCP → сервер `godot-mcp-pro` ← WebSocket → плагин редактора Godot 4.4+. Редактор **один**; всё, что делает MCP, пользователь видит в дереве сцены и инспекторе.

## 1. Перед правками

1. `get_project_info` — версия, пути, autoload.
2. `get_filesystem_tree` — где уже лежат `ui/`, `levels/`, `themes/` (продолжать схему, `modules/structure.md`).
3. `open_scene` нужной сцены → `get_scene_tree` — что уже есть; не плодить дубли кнопок/меню.
4. Имена инструментов ниже — ориентир. **Читай схему сервера** (список tools) перед первым вызовом; параметры не угадывать.

## 2. Создание и расстановка

| Задача | Инструменты (ориентир) |
|---|---|
| Узлы / префабы | `add_node`, `add_scene_instance`, `batch_add_nodes` |
| 3D | `add_mesh_instance`, `setup_lighting`, `setup_environment`, `setup_camera_3d` |
| 2D тайлы | `tilemap_set_cell`, `tilemap_fill_rect`, … |
| UI | Control‑узлы, контейнеры, `set_anchor_preset`, theme/style через свойства |
| Свойства | `update_property` / `batch_set_property` сразу после add (включая `unique_name_in_owner`, `bone_name`, `anchors`) |
| Скрипты | `create_script` / `edit_script` / `attach_script` / `validate_script` |
| Сигналы | `connect_signal` (кнопка → метод в `scripts/ui/…`) |
| Анимации | `AnimationPlayer` / `AnimationTree` как узлы + свойства; сложные треки — импортом GLB (`modules/animation-pipeline.md`), не покадрово кодом |

Типы в MCP часто передаются строками: `"Vector2(100, 200)"`, `"Vector3(1, 2, 3)"`, `"Color(1, 0, 0)"`, `"#ff0000"`, `NodePath("Skeleton3D:Hips")`.

## 3. UI — особый фокус пользователя

1. Иерархия строго по `modules/structure.md` (группы, имена, контейнеры); сцена в `scenes/ui/`.
2. Контейнеры и якоря — resize окна не ломает layout.
3. Тема и стили — `resources/themes/` + свойства узлов, чтобы правились в инспекторе.
4. Логика отдельно: `pressed` → метод в `scripts/ui/…`.
5. После UI‑правок — `get_editor_screenshot` + сверка (`modules/reference-vision.md`).
6. Повторяемые виджеты — отдельная сцена + `add_scene_instance`.

## 4. Сохранение и проверка

- После правок → **`save_scene`**. Без него задача не сделана.
- Редактор: `get_editor_errors`, `get_editor_screenshot`, `get_scene_tree`.
- Runtime: `play_scene` → runtime‑инструменты / `get_game_screenshot` → `stop_scene`. Runtime‑tools только после `play_scene`.
- Не править `.tscn` руками, если есть MCP‑команда.

## 5. Fallback без MCP (или MCP упал)

- Скажи прямо: «MCP недоступен, собираю на диске». Отчёт с пометкой `FALLBACK`.
- **Опасность:** сцена, открытая в редакторе с несохранёнными правками, при сохранении перезапишет твой файл; редактор может не подхватить изменения до перезагрузки сцены. Попроси пользователя сохранить и закрыть сцену (или перезагрузить её после записи).
- Пиши валидный `.tscn` формат Godot 4 (`[gd_scene format=3]`, `ext_resource`, `sub_resource`, `[node name=... type=... parent=...]`), не изобретай поля; после записи — попроси открыть сцену и снять скрин.

## 6. `@tool` — только для editor‑автоматизации

Сотни однотипных объектов — `@tool` + `@export` скрипт, который раскладывает их **в редакторе** и оставляет как обычные узлы. Runtime‑спавнер дизайн уровня/UI не заменяет.

## 7. Skill

Если доступен `GODOT_SKILL_LEVEL_EDITING` (`godot-mcp-direct-level-editing`) — прочитать перед крупной сборкой сцены/уровня.

## 8. MCP и субагенты

- Parent (ты) — единственный, кто дёргает Godot MCP на открытой сцене, делает финальный `save_scene` и vision‑QA.
- Субагенты — исследование, план, черновики `.gd` в файлах, тесты, ревью. Их файлы parent подключает через MCP (`attach_script`, `add_scene_instance`).
- Два writer’а на одну сцену — ЗАПРЕЩЕНО. Разные сцены/файлы — можно параллельно, merge у parent.

## 9. Частые ошибки

- `%BtnPlay` без `unique_name_in_owner = true` → null в рантайме.
- Кнопка добавлена, сигнал не подключён → «ничего не происходит»; всегда `connect_signal` + `validate_script`.
- Control без `set_anchor_preset` → UI уезжает при смене разрешения.
- `save_scene` забыт → пользователь открывает пустую сцену.
- Правка узлов, которые расставил пользователь, без запроса.
