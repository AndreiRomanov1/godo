# kimodo-cpp.md — генерация анимаций скелета по тексту

[kimodo.cpp](https://github.com/localai-org/kimodo.cpp) — C++/GGML‑порт NVIDIA Kimodo (kinematic motion diffusion, обучен на 700 ч mocap). По текстовому промпту генерирует движение скелета: root‑позиции + локальные повороты костей, 30 fps. Работает на CPU/Vulkan без PyTorch. Порт умеет: текстовые промпты, многосегментные последовательности, экспорт GLB/BVH, веб‑демо. **Не умеет пока:** ограничения (keyframes/пути), skinned‑меш, 77‑костную SOMA.

## 1. Модели

| Ключ | Скелет | Лицензия | Использовать |
|---|---|---|---|
| `soma-rp-v1.1` | SOMA, 30 костей | NVIDIA Open Model License (коммерческое ок) | **да, по умолчанию** |
| `soma-seed-v1.1` | SOMA, 30 костей | Open Model License; обучена на меньшем наборе | сравнение/бенчмарк |
| `g1-rp-v1`, `g1-seed-v1` | Unitree G1, 34 сустава | Open Model License | роботы |
| `smplx-rp-v1` | SMPL‑X, 22 сустава | только внутренние исследования | **в игру нельзя**; в установщик не входит |

## 2. Установка (один раз)

```sh
git clone https://github.com/localai-org/kimodo.cpp.git && cd kimodo.cpp
git submodule update --init --recursive
scripts/download_gguf_weights.sh --output "$PWD" --model soma-rp-v1.1     # ~8 ГБ: models/kimodo-soma-rp-v1.1-f32.gguf + generated/llm2vec-text-bundle/
cmake --preset release && cmake --build --preset release                   # build/release/kmd-generate
```

Windows: `python scripts/download_gguf_weights.py --model soma-rp-v1.1`, сборка через VS 2022 — `docs/how_to_setup_with_unreal_in_windows.md` (шаги до экспорта GLB одинаковы для Godot). Нужны C++23‑компилятор, CMake 3.25+, Ninja, Python 3 + `huggingface_hub`; Vulkan — опционально. `KIMODO_TEXT_LAYER_CHUNK=1..32` — подстройка VRAM текстового энкодера.

## 3. CLI

```text
kmd-generate MOTION.gguf TEXT_BUNDLE PROMPT.txt FRAMES STEPS SEED OUTPUT_DIR
kmd-generate MOTION.gguf TEXT_BUNDLE --sequence TRANSITION STEPS SEED OUTPUT_DIR FRAMES PROMPT.txt [FRAMES PROMPT.txt ...]
```

| Параметр | Значение |
|---|---|
| `PROMPT.txt` | файл с промптом (UTF‑8); готовые — `MOTION_PROMPTS` |
| `FRAMES` | кадров при 30 fps; 90–150 для стрельбы/перезарядки; модель до 300 (10 с) |
| `STEPS` | шаги диффузии: 50 черновик, 100 качество |
| `SEED` | воспроизводимость; перебрать 3–5 |
| `TRANSITION` | кадров сглаживания между сегментами, 5–10 |
| `OUTPUT_DIR` | `root_positions.f32` + `local_rotations_xyzw.f32` |

Пример (одиночная + связка):

```sh
B=$KIMODO_ROOT/build/release/kmd-generate; M=$KIMODO_MOTION; T=$KIMODO_TEXT; P=$MOTION_PROMPTS
$B $M $T $P/ak74_fire.txt 120 100 42 $KIMODO_OUT/ak74_fire_seed42
$B $M $T --sequence 8 100 42 $KIMODO_OUT/ak74_seq_seed42 \
   90 $P/sequence/01_ready.txt 150 $P/sequence/02_raise_and_fire.txt 90 $P/sequence/03_lower.txt
```

## 4. Экспорт

```sh
python $KIMODO_ROOT/scripts/export_glb.py --motion-dir $KIMODO_OUT/ak74_fire_seed42 --output $KIMODO_OUT/ak74_fire_seed42/fire.glb          # Godot: скелет + анимация "KimodoMotion", 30 fps, метры
python $KIMODO_ROOT/scripts/export_bvh.py --motion-dir $KIMODO_OUT/ak74_fire_seed42 --output $KIMODO_OUT/ak74_fire_seed42/fire.bvh --scale 1  # Blender: метры (по умолчанию --scale 100 = см для Unreal)
```

В GLB — скин, кости с именами SOMA, служебный меш‑кубики, одна анимация `KimodoMotion`. Переименование — в импорте Godot или в Blender (`modules/animation-pipeline.md §6`).

## 5. Веб‑демо (альтернатива CLI)

`cd $KIMODO_ROOT && go run ./demo -addr 0.0.0.0:8094` → `http://localhost:8094`. Промпт, модель, кадры (60–150 на сегмент), steps (по умолчанию 100), seed, сегменты и transition — в сайдбаре; история сохраняется. Готовый `demo-output/<id>/animation.glb` — кнопка Download или `/api/animations/<id>/animation.glb`.

## 6. Промпты

Правила — `modules/animation-pipeline.md §4`. Кратко: английский, `A person …`, 1–2 действия, 15–30 слов, без марок оружия, описывать хват и видимое движение. Примеры:

```text
A person stands in a combat stance holding a rifle with both hands, aims forward at shoulder height and fires several shots, the body recoiling slightly with each shot.
A person stands still in a combat stance, holding a rifle with both hands and aiming it forward at shoulder height, breathing calmly.
A person holding a rifle with both hands lowers it slightly, pulls out the magazine with the left hand, pushes in a new magazine and raises the rifle back to aim forward.
```

Для последовательности каждый сегмент самодостаточен: «A person holding a rifle raises it…», не «Then he raises it».

## 7. Ограничения и честность

- Тело без пальцев (кончики большого и среднего), без лица, без оружия — оружие добавляется в Godot/Blender.
- Обучающий набор: локомоция, жесты, быт, объекты, videogame combat, танцы, стили. Экзотика — плохо; так и говорить пользователю.
- Foot‑skating возможен (пост‑обработка из Python‑версии NVIDIA в порте не заявлена) — чистить в Blender или принимать для мокапа.
- Нет ограничений (keyframes, пути) в порте — если нужна точная траектория, это делается в Godot (`root_motion`, код) или в Blender.
- Не выдумывать, что генерация прошла: в отчёт — команда, seed, кадры, куда лёг GLB.

## 8. Куда что складывать

| Что | Куда |
|---|---|
| Промпты | `MOTION_PROMPTS` (этот репозиторий) или `REFS/prompts/` |
| `.f32`, варианты seed, `fire_seed7.glb` | `KIMODO_OUT` (`refs/motion/`), не в `assets/` |
| Выбранный GLB | `GAME_ROOT/assets/animations/<actor>/<name>.glb` |
| Запись о лицензии | `GAME_ROOT/assets/LICENSES.md`: файл, модель `soma-rp-v1.1`, NVIDIA Open Model License, промпт, seed |
