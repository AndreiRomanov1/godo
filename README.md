# Анимация стрельбы из АК‑74 через kimodo.cpp → Godot

В репозитории два набора:

- `prompts/` + этот README — промпты и пошаговый путь «kimodo.cpp → GLB → Godot» для стрельбы из автомата.
- `agent-rules/` — правила для ИИ‑агентов (Godot в редакторе, структура, UI‑стиль, game feel, анимационный пайплайн с kimodo.cpp и Blender MCP, адаптеры под Grok Build / Cursor / Claude Code). Начало — `agent-rules/README.md`, ядро — `agent-rules/RULES.md`.

Набор промптов для [kimodo.cpp](https://github.com/localai-org/kimodo.cpp) (C++/GGML‑порт
NVIDIA Kimodo, модель text‑to‑motion), которые генерируют анимацию персонажа,
стреляющего из автомата. На выходе получается `.glb` со скелетом и анимацией —
он напрямую импортируется в Godot 4, а макет АК‑74 крепится к кости `RightHand`.

Важно понимать: Kimodo генерирует **только движение тела** (30 костей скелета SOMA).
Само оружие модель не рисует и не знает, чем АК‑74 отличается от М4, поэтому в
промптах пишется просто `rifle`. Автомат вы добавляете уже в Godot.

## Главный промпт

```text
A person stands in a combat stance holding a rifle with both hands, aims forward at shoulder height and fires several shots, the body recoiling slightly with each shot.
```

Почему именно так:

- Промпты только на английском и начинаются с `A person …` — так размечен обучающий
  датасет (BONES / Bones Rigplay), и модель на такую формулировку реагирует лучше всего.
- Одно‑два действия и средняя детализация. «A person shoots» слишком коротко, а
  пошаговое описание каждой конечности модель «размазывает».
- «holding a rifle with both hands … at shoulder height» задаёт двуручный хват:
  правая рука у рукоятки, левая вытянута вперёд на цевье. Именно под эту позу потом
  подгоняется макет автомата.
- «recoiling slightly with each shot» — это то движение, которое видно в анимации
  выстрела (толчок корпуса и рук).

## Все промпты (`prompts/`)

| Файл | Что получается | Кадров (30 fps) |
| --- | --- | --- |
| `prompts/ak74_fire.txt` | Стоя, прицеливание и несколько выстрелов | 120 |
| `prompts/ak74_aim_idle.txt` | Idle в прицеливании (зацикливать) | 90–120 |
| `prompts/ak74_walk_fire.txt` | Медленно идёт вперёд и стреляет | 150 |
| `prompts/ak74_crouch_fire.txt` | С колена, прицеливание и выстрелы | 120 |
| `prompts/ak74_reload.txt` | Перезарядка (смена магазина) | 120–150 |
| `prompts/sequence/01_ready.txt` → `02_raise_and_fire.txt` → `03_lower.txt` | Связка: оружие опущено → вскинул и дал очередь → опустил | 90 / 150 / 90 |

Хорошая практика — сгенерировать 3–5 вариантов с разными `SEED` и выбрать лучший.

## Генерация в kimodo.cpp

Предполагается, что kimodo.cpp собран (`cmake --preset release && cmake --build --preset release`)
и скачаны веса `soma-rp-v1.1` плюс текстовый энкодер:

```sh
scripts/download_gguf_weights.sh --output "$PWD" --model soma-rp-v1.1
# получатся models/kimodo-soma-rp-v1.1-f32.gguf и generated/llm2vec-text-bundle/
```

Формат CLI:

```text
kmd-generate MOTION.gguf TEXT_BUNDLE PROMPT.txt FRAMES STEPS SEED OUTPUT_DIR
kmd-generate MOTION.gguf TEXT_BUNDLE --sequence TRANSITION STEPS SEED OUTPUT_DIR FRAMES PROMPT.txt [FRAMES PROMPT.txt ...]
```

Одиночная анимация выстрелов (4 с, 100 шагов диффузии, seed 42):

```sh
./build/release/kmd-generate models/kimodo-soma-rp-v1.1-f32.gguf generated/llm2vec-text-bundle \
  /path/to/prompts/ak74_fire.txt 120 100 42 out/ak74_fire
python scripts/export_glb.py --motion-dir out/ak74_fire --output out/ak74_fire/ak74_fire.glb
```

Связка «опущено → очередь → опустил» одним файлом (переход 8 кадров):

```sh
./build/release/kmd-generate models/kimodo-soma-rp-v1.1-f32.gguf generated/llm2vec-text-bundle \
  --sequence 8 100 42 out/ak74_sequence \
  90  /path/to/prompts/sequence/01_ready.txt \
  150 /path/to/prompts/sequence/02_raise_and_fire.txt \
  90  /path/to/prompts/sequence/03_lower.txt
python scripts/export_glb.py --motion-dir out/ak74_sequence --output out/ak74_sequence/ak74_sequence.glb
```

На Windows то же самое через `build\Release\kmd-generate.exe` (см. `docs/how_to_setup_with_unreal_in_windows.md`
в репозитории kimodo.cpp — шаги сборки и скачивания весов идентичны, отличается только движок).

Параметры:

- `FRAMES` — длина в кадрах при 30 fps. Модель даёт максимум 10 с на промпт, веб‑демо
  ограничивает сегмент 60–150 кадрами. Для выстрелов хватает 90–150.
- `STEPS` — шаги диффузии: 50 быстрее, 100 качественнее.
- `SEED` — любое число; меняйте, чтобы получить другой вариант движения.
- `TRANSITION` — кадров на сглаживание между сегментами (5–10).

Альтернатива без CLI: `go run ./demo -addr 0.0.0.0:8094`, открыть `http://localhost:8094`,
вставить промпт, выбрать модель `SOMA RP v1.1` — готовый `animation.glb` скачивается
кнопкой Download (лежит в `demo-output/<id>/animation.glb`).

## Импорт в Godot 4 и крепление макета автомата

1. Перетащите `ak74_fire.glb` в проект. Godot импортирует его как сцену со `Skeleton3D`
   (кости названы как в SOMA: `Hips`, `Spine1`, …, `RightHand`, `LeftHand`) и `AnimationPlayer`
   с анимацией. Единицы — метры, ось Y вверх, масштабировать не нужно.
2. Инстанцируйте сцену, включите «Editable Children» (или «Make Local»), добавьте к `Skeleton3D`
   дочерний `BoneAttachment3D`, в его свойстве `Bone Name` выберите `RightHand`.
3. Дочерним узлом к `BoneAttachment3D` добавьте меш автомата. Сдвигайте и вращайте его локальный
   `Transform`, пока пистолетная рукоятка не окажется в правой ладони, а приклад — у правого плеча.
   Проверяйте на паузе в середине анимации (позе прицеливания).
4. Левая рука сама ляжет близко к цевью, потому что промпт задаёт двуручный хват. Если нужно
   идеальное попадание пальцев на цевьё — добавьте `SkeletonIK3D` для цепочки `LeftArm → LeftHand`
   с целью в точке цевья на меше автомата.
5. Для `ak74_aim_idle` в доке Import → Animation включите Loop Mode = Linear, чтобы idle
   зацикливался. Вспышку выстрела/звук вешайте через `Call Method Track` в `AnimationPlayer`
   на кадрах, где корпус дёргается от отдачи.
6. Чтобы играть анимацию на своём персонаже, а не на кубиках SOMA: в доке Import для GLB
   у `Skeleton3D` включите Retarget → Bone Map с профилем `SkeletonProfileHumanoid`
   (кнопка автомаппинга); тот же Bone Map назначьте своему ригу — тогда анимация
   переносится через `AnimationLibrary`.

Скелет SOMA (30 костей), чтобы ориентироваться в дереве:

```text
Hips
├─ Spine1 → Spine2 → Chest
│   ├─ Neck1 → Neck2 → Head → Jaw, LeftEye, RightEye
│   ├─ LeftShoulder → LeftArm → LeftForeArm → LeftHand → LeftHandThumbEnd, LeftHandMiddleEnd
│   └─ RightShoulder → RightArm → RightForeArm → RightHand → RightHandThumbEnd, RightHandMiddleEnd
├─ LeftLeg → LeftShin → LeftFoot → LeftToeBase
└─ RightLeg → RightShin → RightFoot → RightToeBase
```

## Ограничения

- Пальцев у скелета нет (только кончики большого и среднего), палец на спуске отдельно
  не анимируется — для макета этого достаточно.
- Модель обучена на «videogame combat», поэтому стрельба из винтовки — в её распределении,
  но экзотика (стрельба с бедра в прыжке и т.п.) может выходить хуже.
- Лицензии: чекпоинт SOMA RP v1.1 — NVIDIA Open Model License (коммерческое использование
  разрешено), SMPL‑X — только некоммерческие исследования. Для игры используйте SOMA.
