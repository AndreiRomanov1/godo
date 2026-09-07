# blender-mcp.md — Blender через MCP в пайплайне Godot

Blender MCP (аддон BlenderMCP в открытой сессии Blender + MCP‑сервер `blender-mcp`) даёт агенту живой Blender: инспекция сцены, выполнение Python (`bpy`), скриншот вьюпорта, загрузка ассетов Poly Haven / Sketchfab, генерация моделей Hyper3D / Hunyuan3D. В нашем пайплайне Blender — **инструмент подготовки ассетов и анимаций**, а не место, где живёт игра.

## 1. Роль в пайплайне

| Делаем в Blender | Не делаем в Blender |
|---|---|
| Модели оружия/реквизита: моделирование, импорт готовых, ретопология по необходимости, origin, масштаб | UI и HUD (это Control‑узлы Godot) |
| Материалы/текстуры для экспорта в GLB (PBR) | Раскладку уровня Godot (кроме статичной геометрии/модулей) |
| Импорт анимаций kimodo.cpp (GLB/BVH), чистка ключей, ретаргет аддоном, bake | Игровую логику, стейт‑машины анимаций (это `AnimationTree`) |
| Привязка оружия к кости для проверки хвата, экспорт | Финальную сборку сцены персонажа (это Godot) |
| Быстрые 3D‑иконки/пропсы, когда нет ассетов | Рендер «вместо игры» как результат |

## 2. Инструменты (ориентир для `ahujasid/blender-mcp`; читай схему своего сервера)

| Инструмент | Назначение |
|---|---|
| `get_scene_info`, `get_object_info` | Что в сцене, свойства объекта — вызывать **до** правок |
| `get_viewport_screenshot` | Визуальная проверка (передать в vision) |
| `execute_blender_code` | Любой `bpy`‑код. **Малыми кусками**, по шагу; каждый шаг проверять |
| `get_polyhaven_status`, `get_polyhaven_categories`, `search_polyhaven_assets`, `download_polyhaven_asset`, `set_texture` | Модели, текстуры, HDRI (CC0) |
| `get_sketchfab_status`, `search_sketchfab_models`, `download_sketchfab_model` | Готовые модели — проверить лицензию конкретной модели |
| `get_hyper3d_status`, `generate_hyper3d_model_via_text` / `_via_images`, `poll_rodin_job_status`, `import_generated_asset` | Генерация модели по тексту/фото (лимиты ключа, лицензия сервиса) |

## 3. Правила безопасности

1. Перед `execute_blender_code` — сохранить `.blend` (`bpy.ops.wm.save_as_mainfile(filepath=...)`) в `BLENDER_WORK`; рабочие `.blend` пользователя не перезаписывать без копии.
2. Один writer на сессию Blender: субагенты не трогают Blender параллельно с parent.
3. Код — маленькими шагами; после каждого — `get_scene_info`/`get_viewport_screenshot`, не «100 строк и надеемся».
4. Платные ассеты и модели без ясной лицензии — не качать без просьбы; источник и лицензию — в `assets/LICENSES.md`.
5. Не подменять Godot: если что‑то можно сделать в редакторе Godot и пользователь потом должен это крутить — делай в Godot.

## 4. Экспорт в Godot

- Единицы — метры (scene unit scale 1.0), персонаж ~1.7–1.8 м, оружие в реальном размере.
- Применить трансформы (`bpy.ops.object.transform_apply(location=False, rotation=True, scale=True)`); origin оружия — у пистолетной рукоятки, ствол вдоль одной оси по конвенции проекта.
- Имена объектов/костей/actions — те, что должны появиться в Godot (`Ak74`, `Muzzle`, `fire`).
- Экспорт:

```python
bpy.ops.export_scene.gltf(
    filepath="<GAME_ROOT>/assets/models/weapons/ak74.glb",
    export_format='GLB', use_selection=True,
    export_apply=True, export_animations=False)
```

  Для анимаций — `export_animations=True`, один armature, actions названы как анимации. glTF‑экспортёр сам переводит Z‑up в Y‑up.
- Пустые объекты `Muzzle`/`ShellEject` можно экспортировать как Empty → в Godot станут `Node3D`; либо добавить `Marker3D` уже в сцене оружия в Godot (предпочтительно — правится там).
- Прямой импорт `.blend` в Godot возможен (`BLENDER_BIN` в Editor Settings → FileSystem → Import → Blender Path), но в `assets/` кладём GLB: детерминированно и без зависимости от версии Blender.

## 5. Типовые операции (`execute_blender_code`, по шагам)

```python
# импорт анимации kimodo.cpp
bpy.ops.import_scene.gltf(filepath="<KIMODO_OUT>/ak74_fire_seed42/ak74_fire.glb")
# или BVH (экспорт из kimodo.cpp с --scale 1 → метры)
bpy.ops.import_anim.bvh(filepath="<KIMODO_OUT>/ak74_fire_seed42/animation.bvh", global_scale=1.0)

# переименовать action → имя анимации в Godot
arm = next(o for o in bpy.data.objects if o.type == 'ARMATURE')
arm.animation_data.action.name = "fire"

# оружие к кости (bone‑parent цепляет к концу кости: доправить offset объекта)
w = bpy.data.objects["Ak74"]; w.parent = arm; w.parent_type = 'BONE'; w.parent_bone = "RightHand"

# скриншот для проверки хвата → get_viewport_screenshot
```

Ретаргет на свой риг — аддоном, который есть у пользователя (Rokoko Retargeting, Auto‑Rig Pro, Expy Kit и т.п.); нет аддона → ретаргет через `BoneMap` в импорте Godot (`modules/animation-pipeline.md §6`).

## 6. Чеклист Blender‑шага

- [ ] `.blend` сохранён в `BLENDER_WORK` до правок
- [ ] Масштаб в метрах, трансформы применены, origin осмысленный
- [ ] Имена объектов/костей/actions = имена в Godot
- [ ] GLB экспортирован в `assets/models/…` или `assets/animations/…`
- [ ] Скриншот вьюпорта проверен
- [ ] Источник и лицензия ассетов записаны в `assets/LICENSES.md`
