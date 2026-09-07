# report.md — единый шаблон отчёта и Definition of Done

Отчёт на русском, кратко, по этому шаблону. Пользователь — художник/дизайнер: ему важно, **где** файлы, **как** устроено дерево и **что крутить** в инспекторе.

## Шаблон

```text
## Что сделано
<1–3 предложения: результат, а не процесс>

## Файлы
- scenes/ui/MainMenu.tscn           (новая)
- scripts/ui/main_menu.gd           (новая)
- resources/themes/game_ui_theme.tres (изменена: hover/pressed StyleBox)
- assets/animations/soldier/fire.glb (из kimodo.cpp, seed 42, 120 кадров)

## Дерево (кратко)
MainMenu (Control)
├── Background
├── SafeArea → VBoxMain → Buttons → BtnPlay / BtnSettings / BtnQuit
└── PanelSettings (hidden)

## Что крутить в инспекторе
- WeaponSocket: transform (хват), CameraShaker.power / decay, Theme → Button/hover

## Откуда паттерн
- Theme/hover: урок <имя файла txt>; recoil/shake: *Camera_Shake* + official docs Tween

## Сверка с референсом
- P0–P1 совпали; P2: цвет панели темнее референса на ~10% (нет исходного HEX) | референса не было

## Инструменты и субагенты (что реально вызывалось)
- Godot MCP: add_node ×14, connect_signal ×3, save_scene ✓; get_editor_screenshot ✓
- kimodo.cpp: kmd-generate ×3 (seeds 7/42/99), export_glb ✓
- Blender MCP: не использовался
- Субагенты: explore (поиск Theme‑уроков), reviewer (3 notes, исправлены 2)

## Пометки
- PROTO: <что осталось серым и почему> | FALLBACK: <что записано на диск без редактора> | ДОПУЩЕНИЯ (автономный режим): <…>

## Не сделано / риски
- <честно: чего нет, что не проверено, что нужно от пользователя>
```

## Definition of Done (сводный чеклист)

Структура (`modules/structure.md`):
- [ ] файлы в правильных папках; корень сцены и узлы названы; группы, не плоский dump; instance вместо копий; Input Map

Визуал (`modules/ui-style.md`) — если задача про UI/картинку:
- [ ] Theme + StyleBox‑состояния + spacing + шрифт; не выглядит как пустой Godot‑проект; `PROTO` снят или помечен

Feel (`modules/game-feel.md`) — если задача про действие:
- [ ] фазы + ярус + каналы; логика на IMPACT; параметры `@export`; один FX‑слой

Анимации (`modules/animation-pipeline.md`) — если задача про персонажа:
- [ ] GLB в `assets/animations/`, имена `snake_case`, `AnimationTree` в сцене, оружие через `BoneAttachment3D`, лицензии записаны

Референс (`modules/reference-vision.md`):
- [ ] vision‑сверка выполнена (≤3 итераций) или честно «не сверено»

Общее:
- [ ] сцена сохранена; `get_editor_errors` чист; `play_scene` smoke пройден
- [ ] пользовательские узлы не тронуты без запроса
- [ ] отчёт по шаблону, все инструменты/субагенты перечислены честно
