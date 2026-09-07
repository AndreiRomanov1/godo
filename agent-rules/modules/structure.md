# structure.md — папки проекта, дерево нод, naming, git

Читать перед созданием любых файлов или узлов. Дерево сцены — это UI пользователя в редакторе; папки — его карта проекта.

## 1. Каркас папок (Godot 4, если в проекте ещё нет своей схемы)

```text
res://
  project.godot
  scenes/
    main/                # entry, game root, bootstrap
    levels/              # уровни / локации
    actors/              # player, npc, enemies (префабы)
    props/               # интерактивные объекты и декор
      weapons/           # Ak74.tscn и т.п. (меш + Muzzle/ShellEject + звук)
    ui/                  # меню, HUD, окна, диалоги (отдельные .tscn)
    systems/             # scene‑based managers (иначе autoload)
  scripts/
    actors/ ui/ systems/ components/
  resources/             # .tres: stats, items, dialogue data…
    items/
    themes/              # Theme, StyleBox
    animations/          # AnimationLibrary / сохранённые из импорта .res
  assets/
    textures/ sprites/ fonts/ shaders/
    models/
      characters/        # GLB персонажей (меш + скелет)
      weapons/           # GLB оружия
      props/
    animations/          # GLB только с анимацией (вывод kimodo.cpp / Blender)
      <actor>/           # aim_idle.glb, fire.glb, reload.glb …
    audio/
      music/ sfx/
    LICENSES.md          # источник и лицензия каждого внешнего/сгенерированного ассета
  data/                  # json/csv — если не resources
  addons/                # плагины (не трогать без нужды)
  refs/                  # НЕ игровые файлы: референсы, промпты, сырые .f32, рабочие .blend
```

Перед добавлением: `get_filesystem_tree` / листинг / CBM — **есть ли уже** `ui/`, `menus/`, `HUD`? Клади туда же, не заводи синоним (`UI` + `ui` + `Menus`). В отчёте пиши, куда положил.

| Клади сюда | Не клади сюда |
|---|---|
| UI‑сцены → `scenes/ui/` | UI вперемешку с уровнями в корне |
| Скрипт кнопки/меню → `scripts/ui/` (или co‑located, как в проекте) | все `.gd` кучей в `res://` |
| Текстуры кнопок → `assets/sprites/ui/` | дубли ассетов «на всякий» |
| Theme / StyleBox → `resources/themes/` | стили, зашитые в один узел, если тема общая |
| Уровень → `scenes/levels/level_01.tscn` | монолит `World.tscn` со всем UI внутри |
| Анимации персонажа → `assets/animations/<actor>/` | `.glb` рядом с `.tscn` в `scenes/` |
| Промпты, `.f32`, референсы → `refs/` или вне репо | внутри `assets/` и `scenes/` |

## 2. Naming‑канон

Если проект уже задал стиль — продолжай его. Иначе:

| Что | Стиль | Пример |
|---|---|---|
| Папки | `snake_case`, английский | `scenes/ui/` |
| Сцены и `class_name` | `PascalCase` | `MainMenu.tscn`, `Player.tscn` |
| Скрипты | `snake_case` | `main_menu.gd` рядом по смыслу (`scripts/ui/` или co‑located) |
| Ресурсы `.tres`/`.res` | `snake_case` | `game_ui_theme.tres` |
| Ассеты | `snake_case` | `ak74_body_albedo.png`, `fire.glb` |
| Узлы | `PascalCase`, роль + имя | `BtnPlay`, `PanelSettings`, `EnemySpawnPoints`, `WeaponSocket` |
| Анимации | `snake_case` глагол/состояние | `aim_idle`, `fire`, `fire_burst`, `reload`, `walk_aim` |
| Input actions | `snake_case` | `fire`, `aim`, `reload`, `move_left` |

Один стиль на проект. Не смешивать `main_menu` / `MainMenu` / `mainMenu`. Не плодить `MainMenu2.tscn` — вариации через theme/visibility/экспорт.

## 3. Дерево нод — общие правила

1. Корень = смысл файла: `MainMenu`, `Level01`, `Player`, `Hud`. Не `Node2D`/`Control` без имени.
2. Группы‑папки (пустые `Node`/`Node2D`/`Node3D`/`Control`): `World`, `Entities`, `Lighting`, `CameraRig`, `UI`, `Systems`.
3. Имена уникальные и читаемые в пределах сцены. ЗАПРЕЩЕНЫ `Control`, `Control2`, `Node2D3`, `@Node@42`.
4. Порядок детей = порядок смысла и draw order: фон → мир → сущности → оверлеи → UI.
5. Не смешивать 2D‑геймплей, 3D‑мир и UI в одном корне без явных веток.
6. Переиспользуемое — отдельная сцена + `add_scene_instance`, не копипаст 15 кнопок.
7. `%Name` — только при включённом `unique_name_in_owner`. Поиск через `get_child(3)` ЗАПРЕЩЁН.
8. После сборки `get_scene_tree`: дерево читается «с листа», без догадок.

### UI‑сцена (канон)

```text
MainMenu (Control)                     # fullscreen, anchors full rect
├── Background (TextureRect / ColorRect)
├── SafeArea (MarginContainer)
│   └── VBoxMain (VBoxContainer)
│       ├── Logo (TextureRect)
│       ├── Buttons (VBoxContainer)
│       │   ├── BtnPlay (Button)
│       │   ├── BtnSettings (Button)
│       │   └── BtnQuit (Button)
│       └── VersionLabel (Label)
├── PanelSettings (PanelContainer)     # скрыт по умолчанию или отдельная сцена
└── System (Node)                      # только невизуальная обвязка
```

Контейнеры (`Margin`/`VBox`/`HBox`/`Center`/`Grid`) для layout; `set_anchor_preset` для fullscreen и safe edges; слои Background → Content → Overlays → Modals; сложный popup — своя сцена.

### 2D level

```text
Level01 (Node2D)
├── World (Node2D)
│   ├── TileMapLayer / Terrain
│   ├── Props (Node2D)
│   └── Zones (Node2D)                 # Area2D triggers
├── Entities (Node2D)
│   ├── Player (instance)
│   └── Enemies (Node2D)
├── CameraRig (Node2D)
│   └── Camera2D
├── SpawnPoints (Node2D)
└── UI (CanvasLayer)
    └── Hud (instance → scenes/ui/Hud.tscn)
```

### 3D level

```text
Level01 (Node3D)
├── Environment (Node3D)               # WorldEnvironment, sun, probes
├── Geometry (Node3D)
├── Lighting (Node3D)
├── Entities (Node3D)
│   └── Player (instance)
├── CameraRig (Node3D)
├── Navigation (Node3D)                # при необходимости
└── UI (CanvasLayer)
    └── Hud (instance)
```

### Actor 2D

```text
Player (CharacterBody2D)
├── CollisionShape2D
├── Visual (Node2D)                    # спрайт/анимация отделены от логики
│   └── AnimatedSprite2D
├── Hitbox / Hurtbox (Area2D)
└── Components (Node)
```

### Actor 3D с скелетом и оружием

```text
Player (CharacterBody3D)               # скрипт логики — здесь
├── CollisionShape3D
├── Visual (Node3D)
│   └── Model (instance assets/models/characters/soldier.glb)
│       └── Skeleton3D
│           └── RightHandAttachment (BoneAttachment3D, bone_name = RightHand)
│               └── WeaponSocket (Node3D)          # ручной offset под хват
│                   └── Ak74 (instance scenes/props/weapons/Ak74.tscn)
│                       ├── Mesh (MeshInstance3D)
│                       ├── Muzzle (Marker3D)      # точка вспышки/рейкаста
│                       └── ShellEject (Marker3D)
├── AnimationPlayer                    # библиотека: aim_idle, fire, reload, walk_aim
├── AnimationTree                      # state machine / blend; правится в редакторе
├── CameraRig (Node3D) → Camera3D      # kick/shake через @export параметры
└── Components (Node)                  # Health, WeaponController, Feel
```

Скрипт — на корне актора; визуал и оружие правятся без поломки коллизий и логики.

## 4. Сцены, инстансы, ownership

| Что | Как |
|---|---|
| Экран меню, HUD, диалог | отдельный `.tscn` в `scenes/ui/` |
| Player, enemy, chest, оружие | префаб в `scenes/actors/` или `scenes/props/` |
| Уровень | `scenes/levels/…`, инстансит player/props/ui, не содержит уникальных «навсегда» кнопок |
| Повторяющийся виджет (HP‑бар, слот) | своя сцена + instance |
| Одноразовый декор уровня | local nodes под `Props` |

Не раздувать одну сцену до «вся игра». Autoload — только Game, Save, Audio, EventBus; UI‑дерево туда не тащить.

## 5. Input Map

Действия ввода — в Project Settings → Input Map (пользователь переназначает в редакторе). В коде только `Input.is_action_just_pressed("fire")`. Хардкод `KEY_SPACE` / `MOUSE_BUTTON_LEFT` в скриптах ЗАПРЕЩЁН.

## 6. Git‑гигиена

- `.gitignore`: `.godot/`, `*.tmp`, `refs/motion/**/*.f32`, рабочие `.blend1`. Файлы `*.import` рядом с ассетами **коммитятся**.
- Большие бинарники (>10–20 МБ) — Git LFS или вне репо.
- Чужие уроки, книги, PDF, платные ассеты — не коммитить. Сгенерированные ассеты — с записью в `assets/LICENSES.md`.
- Один логический шаг = один коммит с понятным сообщением.

## 7. Чеклист структуры (перед «готово»)

- [ ] Новые файлы в логичных папках, не в корне без причины
- [ ] Корень сцены именован по смыслу; группы‑папки, не плоский dump
- [ ] Нет `Control2` / `Node2D3`; `%Name` с включённым unique‑флагом
- [ ] UI отделён (`CanvasLayer` / `scenes/ui`), геймплей — в своих ветках
- [ ] Повторяемое — instance, не копия
- [ ] Скрипты рядом со смыслом, стиль имён единый
- [ ] Ввод — через Input Map
- [ ] Внешние/сгенерированные ассеты записаны в `assets/LICENSES.md`
- [ ] Сцена сохранена; в отчёте — пути и краткое дерево
