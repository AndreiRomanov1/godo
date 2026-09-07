# ui-style.md — современный UI, а не «серый Godot 2016»

Модель по умолчанию тянется к устаревшему виду: серые `Button`, системный шрифт, нулевой polish, учебный layout. Это ЗАПРЕЩЕНО как финальный результат. Стиль не угадывается из памяти модели — он достаётся из референсов пользователя, базы курса (`paths.md: LESSONS_TXT`, `BOOKS_TXT`) и official docs.

## 1. Честная граница

| Можешь и должен | Не обещай без ресурсов |
|---|---|
| Современный UI‑system: Theme, StyleBox, состояния, spacing, type scale | AAA‑арт с нуля без спрайтов/моделей |
| Иерархия, safe area, visual hierarchy, hover/press/focus | Уникальный character art без ассетов |
| 2D light / 3D Environment, exposure, glow, fog, shadows по жанру | Фотореализм «как Unreal demo» |
| Juice: tween, punch scale, shake, SFX hooks, hit‑stop (уместно) | Полный арт‑пайплайн за один промпт |
| 1‑в‑1 к референсу + vision | Игнор базы: «серая кнопка — и так сойдёт» |

Нет ассета → скажи и предложи: (а) Theme + StyleBoxFlat как сильный stylized UI; (б) placeholder‑слоты под арт; (в) где взять pack (платное без просьбы не качать). Через Blender MCP можно быстро сделать простые 3D‑иконки/реквизит — UI‑виджеты в Blender не моделируются.

## 2. ЗАПРЕЩЕНО сдавать как «готово»

- Дефолтный Godot UI: серые кнопки, стандартный шрифт, без Theme.
- Кнопки без hover / pressed / focus (и disabled, если состояние есть).
- Случайные цвета без палитры; нулевые или хаотичные отступы; всё в один ряд.
- UI вперемешку с world‑нодами без `CanvasLayer`.
- 3D: серые меши + одно `DirectionalLight3D` без environment/ambient/теней, если задача визуальная.
- «Button и Label» без панели, фонового слоя и ритма отступов.
- Слепое копирование устаревшего beginner‑layout из старых уроков, когда в базе есть Theme/polish‑уроки новее.

Временный серый прототип допустим на время сборки дерева — пометь `PROTO` в отчёте, polish‑pass до сдачи.

## 3. Минимум для нормального UI

1. **`Theme`** в `resources/themes/<project>_ui_theme.tres` — не 40 одиночных override «на глаз».
2. **StyleBox** на интерактив: `StyleBoxFlat` (radius, border, shadow, anti‑aliasing, content margins) или `StyleBoxTexture`/atlas, если есть UI‑атлас.
3. Состояния normal → hover → pressed → focus (+ disabled).
4. Шрифт проекта (не дефолт) и иерархия размеров title / body / caption.
5. Spacing system 4/8/12/16/24 через `MarginContainer` и `separation` контейнеров — не «подвинь на 3px».
6. Цвет: 1 accent + neutrals + semantic (ok / warn / danger); читаемый контраст.
7. Фон / panel / scrim под меню — не голые кнопки на пустом viewport.
8. Фокус видим для геймпада и клавиатуры.
9. Анимации появления и hover — `AnimationPlayer` или Tween, 100–200 мс, без цирка.

## 4. 2D / 3D — не плоско

| 2D | 3D |
|---|---|
| Слои, parallax по задаче | `WorldEnvironment` + sky/ambient |
| `Light2D` / modulate осмысленно | Directional + fill; тени, если стиль позволяет |
| Y‑sort / z‑index порядок | Материалы не «серый StandardMaterial» на всё |
| VFX через `GPUParticles2D` / анимации | Post: glow / SSR / SSAO по жанру, не всё сразу |
| Camera juice уместно | Экспозиция, fog distance — читаемость сцены |

Современность = читаемость + иерархия + coherency + feedback, а не «максимум bloom».

## 5. Порядок работы (anti‑stale‑brain)

```text
1) Референс пользователя → vision‑разбор (modules/reference-vision.md)
2) CBM / поиск по проекту → какой Theme и ассеты уже есть
3) База курса: Theme, menu, polish, juice, lighting (modules/knowledge-base.md)
4) Только потом сборка в редакторе
5) Скрин → vision vs референс / чеклист §6
```

Нет референса: интерактивно — предложи 2 направления по жанру и спроси; автономно — выбери одно, зафиксируй допущение в отчёте. «Современно» ≠ один стиль на всех; главный арбитр — референсы пользователя.

## 6. Чеклист сдачи UI

- [ ] Не выглядит как пустой Godot‑проект
- [ ] Есть Theme (или явный style kit сцены) в `resources/themes/`
- [ ] Hover / press / focus на всём кликабельном
- [ ] Отступы ритмичные, выравнивание контейнерами
- [ ] Шрифт и контраст в порядке
- [ ] UI в `scenes/ui/` с деревом по `modules/structure.md`
- [ ] Если был референс — vision‑diff закрыт или gap честно описан
- [ ] В отчёте — откуда взят паттерн (урок / docs / референс)
