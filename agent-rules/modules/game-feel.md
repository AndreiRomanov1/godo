# game-feel.md — AAA game feel на любое действие

Речь не про арт. Речь про то, как ощущается любое геймплейное действие: удар, выстрел, прыжок, урон, лут, диалог, level‑up, столкновение. Плохо: `target.hp -= dmg` + мгновенный `position = target`. Нормально: фазы + 2–5 каналов feedback + easing + амплитуда, масштабируемая силой события.

## 1. Что говорит база (читать, не выдумывать)

| Источник (`paths.md: LESSONS_TXT`, `BOOKS_TXT`) | Суть |
|---|---|
| Game Juice (диалог) | Juice = SFX + VFX + мелочи, усиливающие feel. Функция уже работает — juice делает её телесной |
| Camera Juice / FPS | Жёсткая камера = «нет тела». Камера реагирует: tilt, bob, fall impact, damage shake, recoil. Мало — пусто; много — тошнит |
| Camera Shake / Polish | Impact = shake + white flash (+ red tint только при реальной потере HP). Вес удара — комбинация каналов |
| VFX Factory | Эффекты не копипастить в каждый entity: `VfxFactory.spawn(id, pos)` из одной точки |
| Official docs (first games) | После «работает» — juice pass; анимация не linear: timing/spacing, ease in/out, overshoot |
| Level‑up polish | Событие невозможно не заметить: animation + SFX + particles/UI punch |

## 2. Формула на любое действие

```text
1. ANTICIPATION   — подготовка: wind‑up, scale up, lift, charge, пауза 50–150 мс
2. ACTION         — основной жест с EASE (не linear)
3. IMPACT         — контакт: hit‑stop / punch scale / flash / shake / SFX / particles
4. FOLLOW‑THROUGH — остаток энергии: overshoot, settle, debris, fade, return home
5. STATE          — HP / score / turn применяются на IMPACT (или синхронно), не раньше картинки
```

## 3. Каналы feedback

| Канал | Примеры | Когда |
|---|---|---|
| Motion | tween path, arc, overshoot, settle | почти всегда |
| Scale / squash | punch 1.0 → 1.15 → 0.95 → 1.0 | impact, pickup |
| Rotation | tilt, spin, recoil | удары, оружие |
| Z‑order | поднять атакующий объект над полем | «в воздухе» |
| Flash / modulate | white hit flash, red damage tint | урон |
| Camera | trauma shake, kick, FOV punch | сильные удары, выстрел |
| Particles / VFX‑сцена | burst, dust, sparks, muzzle flash, гильзы | impact, land, fire |
| Audio | whoosh + hit + bass layer | обязательно на impact |
| Time | hit‑stop 1–4 кадра / slow‑mo | тяжёлые удары |
| Haptics / UI | number popup, bar punch | по жанру |

## 4. Ярусы интенсивности (обязательно калибровать)

| Ярус | Пример события | Каналы |
|---|---|---|
| **light** | наведение/клик по кнопке, шаг, подбор монеты | 1–2: motion + короткий SFX; никакого shake |
| **medium** | выстрел из автомата, обычный удар, прыжок‑приземление, открыть дверь | 3–4: recoil‑анимация + camera kick 1–2° + muzzle flash/particles + SFX (+ гильза) |
| **heavy** | крит, взрыв, смерть босса, level‑up | 4–6: всё выше + shake + hit‑stop/slow‑mo + flash + UI punch |

Очередь из автомата: каждый выстрел — medium с уменьшенной амплитудой (иначе тошнит), накопление trauma с decay, а не сумма shake’ов.

## 5. Motion design

1. Не linear: `TRANS_BACK` / `TRANS_CUBIC` / `TRANS_EXPO` + `EASE_OUT` на прилёт; wind‑up часто `EASE_IN`.
2. Timing и spacing — неравномерные ключи, контраст быстро/медленно.
3. Arc — живое движение почти никогда не идёт по прямой.
4. Anticipation в противоположную сторону, потом в цель.
5. Overshoot → settle.
6. Readability first: игрок за 0.2 с понимает кто → кого → урон применён.

## 6. Архитектура и «правится глазами»

```text
Gameplay code  →  событие (AttackResolved, Landed, Damaged, ShotFired)
       ↓
Feel / FX layer →  оркестрирует фазы и каналы
       ↓
  ├─ AnimationPlayer / AnimationTree на акторе   (авторская анимация — в редакторе)
  ├─ VfxFactory.spawn("muzzle_flash", muzzle.global_transform)
  ├─ CameraShaker.add_trauma(amount)              (@export power, decay)
  ├─ AudioBus.play("ak_fire")
  └─ optional HitStop.freeze(ms)
```

- **Авторское движение** (recoil оружия, punch UI, появление меню) — треки `AnimationPlayer`: пользователь правит кривые в таймлайне. **Процедурное** (trauma shake, попапы чисел) — Tween/код с `@export`‑параметрами (power, duration, decay), чтобы крутить в инспекторе.
- Один FX‑слой: сигналы / EventBus / FX service — одна точка правды. Не размазывать shake/particles по 15 скриптам.
- VFX как сцены в `scenes/props/vfx/` (или `scenes/fx/`), спавн через фабрику.

## 7. Процесс агента

1. Назови событие одной фразой («resolve attack», «fire one round», «land from jump»).
2. Разложи на 4 фазы (§2) в плане.
3. Выбери ярус (§4) и 2–5 каналов (§3).
4. Дёрни базу: juice, shake, polish, tween, VFX factory — не изобретай linear lerp.
5. Реализуй: анимации в `AnimationPlayer`, последовательность фаз через `AnimationTree`/`create_tween` chains/`await`, логику урона на IMPACT.
6. Playtest + скрин/видео: пусто — добавь канал; тошнит — урежь амплитуду.
7. В отчёте: фазы, каналы, ярус, откуда паттерн.

**ЗАПРЕЩЕНО сдавать:** мгновенный state change без anticipation/impact; linear‑only на важном beat; урон без хотя бы одного канала (flash / shake / SFX); heavy‑набор на каждый клик.

## 8. Якоря в базе (`paths.md: LESSONS_TXT`)

```text
*Game_Juice*  *Juice*  *Polish*  *Camera_Shake*  *Camera_Juice*  *Screen_Shake*
*Visual_Effects*  *Level_Up_Polish*
BOOKS_TXT/godot-official-docs-stable/full.txt → Tween, AnimationPlayer, easing, "juice"
CBM project: godot-course
```
