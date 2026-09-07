# animation-pipeline.md — персонаж, скелет, анимации, оружие в руке

Пайплайн: **kimodo.cpp** генерирует движение скелета по тексту → (опционально **Blender MCP** чистит, ретаргетит, привязывает реквизит, экспортирует) → **Godot** собирает `AnimationPlayer` / `AnimationTree`, крепит оружие на кость, добавляет feel. Всё, что пользователь потом крутит руками, живёт в редакторе Godot.

## 1. Кто что делает

| Инструмент | Делает | ЗАПРЕЩЕНО |
|---|---|---|
| kimodo.cpp (`tools/kimodo-cpp.md`) | Движение скелета по промпту: idle, стрельба, перезарядка, ходьба, удары, жесты. Вывод GLB (30 fps) или BVH | Ждать от него меш, оружие, пальцы, лицевую анимацию; промпты про экзотику вне обучающего набора (бейсбол, акробатика) |
| Blender MCP (`tools/blender-mcp.md`) | Модели оружия/реквизита, импорт готовых (Poly Haven / Sketchfab с проверкой лицензии), чистка ключей, ретаргет аддоном, привязка оружия к кости, bake, экспорт GLB | UI, раскладка уровня Godot, игровая логика, «рендер вместо игры» |
| Godot (редактор / MCP) | Импорт GLB, `BoneMap`‑ретаргет, `AnimationLibrary`, `AnimationPlayer` / `AnimationTree`, `BoneAttachment3D`, Input Map, feel (`modules/game-feel.md`) | Собирать анимацию кодом покадрово; хранить `.f32`/промпты в `assets/` |

## 2. Маршруты

| Маршрут | Когда | Шаги |
|---|---|---|
| **A. Прямой** | Прототип, мокап, свой персонаж ещё не готов | kimodo → `export_glb.py` → `assets/animations/<actor>/<name>.glb` → Godot import → `AnimationPlayer` |
| **B. На свой риг (в Godot)** | Есть свой персонаж‑GLB с гуманоидным скелетом | Как A, плюс в импорте обоих GLB `Skeleton3D → Retarget → Bone Map` (`SkeletonProfileHumanoid`) → анимации переносятся через `.res` / `AnimationLibrary` |
| **C. Через Blender** | Нужно править ключи, убрать дрожание, ретаргет аддоном, запечь оружие в риг | kimodo → GLB или BVH → Blender MCP (import, чистка, retarget, parent‑to‑bone, bake) → `export_scene.gltf` → Godot |

По умолчанию — A или B: меньше шагов, результат правится в редакторе. C — когда A/B не дают качества.

## 3. Скелет kimodo.cpp (SOMA, 30 костей)

```text
Hips
├─ Spine1 → Spine2 → Chest
│   ├─ Neck1 → Neck2 → Head → Jaw, LeftEye, RightEye
│   ├─ LeftShoulder → LeftArm → LeftForeArm → LeftHand → LeftHandThumbEnd, LeftHandMiddleEnd
│   └─ RightShoulder → RightArm → RightForeArm → RightHand → RightHandThumbEnd, RightHandMiddleEnd
├─ LeftLeg → LeftShin → LeftFoot → LeftToeBase
└─ RightLeg → RightShin → RightFoot → RightToeBase
```

- Единицы — метры, Y вверх, 30 fps; Godot масштабировать не нужно.
- Пальцев нет (только кончики большого и среднего) — палец на спуске не анимируется, для макета достаточно.
- Меш в GLB — служебные кубики на костях (`KimodoSkinnedMesh`); скрой `MeshInstance3D` или используй только для ретаргета.
- После `BoneMap`‑ретаргета Godot переименует кости в имена профиля: `RightArm → RightUpperArm`, `RightForeArm → RightLowerArm`, `LeftLeg → LeftUpperLeg`, `LeftShin → LeftLowerLeg`, `Spine1/Spine2/Chest → Spine/Chest/UpperChest`, `Neck1/Neck2 → Neck`. `RightHand` / `LeftHand` не меняются. `bone_name` у `BoneAttachment3D` указывай по фактическим именам в `Skeleton3D`.

## 4. Промпты для kimodo.cpp

- Английский, начало **`A person …`**, одно‑два действия, средняя детализация (15–30 слов). Не «A person shoots» и не пошаговое описание каждой конечности.
- Модель не знает оружия по названиям — пиши `rifle` / `pistol` / `two‑handed weapon`, описывай хват и позу: `holding a rifle with both hands at shoulder height`.
- Опиши видимое движение события: `recoiling slightly with each shot`, `pulls out the magazine with the left hand`.
- Обучающий набор: локомоция, жесты, бытовые действия, взаимодействие с объектами, **videogame combat**, танцы, стили (tired / angry / drunk / injured / stealthy / old). Вне набора качество падает.
- Длинное действие — последовательность сегментов (`--sequence`), каждый сегмент самодостаточен по контексту («A person holding a rifle raises it…», а не «Then he raises it»).
- Готовые промпты под АК‑74: `paths.md: MOTION_PROMPTS` (`ak74_fire`, `ak74_aim_idle`, `ak74_walk_fire`, `ak74_crouch_fire`, `ak74_reload`, `sequence/01–03`).

## 5. Генерация (сводно, детали в `tools/kimodo-cpp.md`)

- Модель: `soma-rp-v1.1` (коммерческое ок). `KIMODO_MOTION` / `KIMODO_TEXT` из `paths.md`.
- Кадры: 30 fps; 90–150 на сегмент для стрельбы/перезарядки; модель до 300 (10 с), веб‑демо ограничивает 60–150.
- Steps 100 (50 — быстрее для черновиков), seed — перебрать 3–5 вариантов, выбрать лучший, зафиксировать seed в отчёте.
- Вывод — в `KIMODO_OUT` (`refs/motion/<name>_seed<N>/`), в `assets/animations/<actor>/` кладётся только выбранный GLB.

## 6. Импорт в Godot

1. `assets/animations/<actor>/fire.glb` (имя файла = будущее имя анимации). Godot импортирует сцену: `Skeleton3D` с костями SOMA + `AnimationPlayer` с анимацией **`KimodoMotion`** (имя из экспортёра).
2. Import dock → **Advanced…**: выбери анимацию → `Loop Mode` (Linear для `aim_idle`, `walk_aim`; None для `fire`, `reload`) → **Save to File** → `resources/animations/<actor>/fire.res`. Альтернатива имени — `Slices` (слайс `fire` на весь диапазон) или переименовать Action в Blender до экспорта.
3. Свой риг: для GLB персонажа и для GLB анимаций в Advanced Import у `Skeleton3D` → `Retarget → Bone Map` → новый `BoneMap` с `SkeletonProfileHumanoid` → автомаппинг → проверить руки/ноги/позвоночник глазами.
4. В акторе (`modules/structure.md`, «Actor 3D»): `AnimationPlayer` → Manage Animations → библиотека → Load `fire.res` под ключом `fire`; повторить для остальных.
5. **`AnimationTree`** (правится в редакторе): `AnimationNodeStateMachine` со состояниями `aim_idle` ↔ `walk_aim` (переход по параметру скорости), `reload` (переход туда/обратно), выстрел — `AnimationNodeOneShot` поверх состояния. Код только переключает параметры: `tree.set("parameters/fire/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE)`.
6. Локомоция из kimodo содержит перемещение `Hips` — для `walk_aim` включи `root_motion_track` (`Skeleton3D:Hips`) и двигай `CharacterBody3D` через `get_root_motion_position()`; для стрельбы на месте не нужно.
7. Сохранить сцену; `get_editor_errors`; `play_scene` — проверить переходы.

## 7. Оружие в руке

```text
Skeleton3D
└── RightHandAttachment (BoneAttachment3D, bone_name = "RightHand")
    └── WeaponSocket (Node3D)      # offset/rotation под хват — крутится в инспекторе
        └── Ak74 (instance scenes/props/weapons/Ak74.tscn)
            ├── Mesh (MeshInstance3D)
            ├── Muzzle (Marker3D)
            └── ShellEject (Marker3D)
```

- Подгонка: пауза анимации в позе прицеливания → двигать `WeaponSocket`, пока рукоятка в правой ладони, приклад у правого плеча, цевьё под `LeftHand`.
- Левая рука на цевьё: обычно хватает offset (промпт задал двуручный хват). Точное попадание — `SkeletonIK3D` на цепочке `LeftArm → LeftHand` с целью `Marker3D` на цевье (класс устаревший, но рабочий в 4.x; в новых версиях — семейство `SkeletonModifier3D`).
- Оружие — отдельная сцена в `scenes/props/weapons/`: меш из `assets/models/weapons/ak74.glb`, origin у рукоятки, `+Z`/`-Z` вдоль ствола по конвенции проекта (зафиксировать в `README` проекта), `Muzzle` и `ShellEject` как точки для VFX.
- Смена оружия — замена instance под `WeaponSocket`, скелет и анимации не трогаются.

## 8. Выстрел как событие (medium, `modules/game-feel.md §4`)

`fire` (recoil из kimodo, при необходимости усилить трек `Chest`/`RightArm` в редакторе) + `Call Method Track` в анимации на кадре выстрела → `WeaponController.shoot()` → `VfxFactory.spawn("muzzle_flash", %Muzzle.global_transform)` + `CameraShaker.add_trauma(0.15)` (kick 1–2°, decay) + `AudioBus.play("ak_fire")` + гильза из `ShellEject`. Урон/рейкаст — на этом же кадре (IMPACT), не при нажатии клавиши. Очередь — накопление trauma с decay, не сумма shake’ов.

## 9. Blender‑шаг (маршрут C, через `execute_blender_code` малыми кусками)

```python
import bpy
bpy.ops.import_scene.gltf(filepath="<KIMODO_OUT>/ak74_fire_seed42/ak74_fire.glb")   # или import_anim.bvh(filepath=..., global_scale=1.0)
arm = next(o for o in bpy.data.objects if o.type == 'ARMATURE')
arm.animation_data.action.name = "fire"                                              # имя анимации в GLB
# чистка: Graph Editor smooth / decimate по необходимости; ретаргет — аддоном проекта
weapon = bpy.data.objects["Ak74"]
weapon.parent = arm; weapon.parent_type = 'BONE'; weapon.parent_bone = "RightHand"   # bone‑parent цепляет к концу кости — доправить offset
bpy.ops.export_scene.gltf(filepath="<GAME_ROOT>/assets/animations/soldier/fire.glb",
                          export_format='GLB', export_animations=True, export_apply=True)
```

- Перед `execute_blender_code` — сохранить `.blend` в `BLENDER_WORK`. Проверять результат `get_viewport_screenshot`.
- Godot может импортировать `.blend` напрямую (Editor Settings → FileSystem → Import → Blender Path = `BLENDER_BIN`), но в `assets/` предпочтительнее GLB: детерминированный импорт и без зависимости от установленного Blender.

## 10. Лицензии и что коммитить

| Источник | Условия | В релиз |
|---|---|---|
| Kimodo SOMA RP / SEED v1.1, G1 | NVIDIA Open Model License | да |
| Kimodo SMPL‑X RP v1 | только внутренние исследования, деривативы не распространять | **нет** |
| Текстовый энкодер (Llama 3) | своя лицензия Meta; используется только при генерации, в игру не попадает | — |
| Poly Haven | CC0 | да |
| Sketchfab / Hyper3D / Hunyuan3D | по лицензии конкретной модели / сервиса — проверить | по лицензии |

Коммитятся: финальные `.glb`, `.res`, `.tscn`, `assets/LICENSES.md` (файл → источник → лицензия → seed/промпт для воспроизводимости). Не коммитятся: `.f32`, промпты, рабочие `.blend`, варианты seed — `refs/` или вне репо.

## 11. Чеклист анимационной задачи

- [ ] Промпт по §4, seed и параметры записаны
- [ ] Выбранный GLB в `assets/animations/<actor>/`, сырое — в `refs/`
- [ ] Анимации названы `snake_case`, loop mode выставлен, сохранены в `resources/animations/`
- [ ] `AnimationTree` в сцене, переходы видны в редакторе, код только дёргает параметры
- [ ] Оружие через `BoneAttachment3D → WeaponSocket → instance`, offset подогнан в позе прицеливания
- [ ] Выстрел прошёл через feel‑ярус medium (`modules/game-feel.md`)
- [ ] Лицензии записаны в `assets/LICENSES.md`, SMPL‑X не используется
- [ ] Сцена сохранена, `play_scene` прошёл, в отчёте — дерево и что крутить
