# Обновление веток: инструкция

Репозиторий — учебный курс GPGPU 2026. Схема remote'ов:

| remote     | URL                                              | роль |
|------------|--------------------------------------------------|------|
| `origin`   | https://github.com/alexserafim/GPGPUTasks2026    | **мой форк** (сюда пушу своё, отсюда делаю PR) |
| `upstream` | https://github.com/GPGPUCourse/GPGPUTasks2026    | **оригинал курса** (преподаватели; только читаю) |

Ветки в upstream:

- `main` — общая ветка: слайды, лидерборд, README верхнего уровня.
- `task00` … `task05` — по одной ветке на задание. Преподаватели периодически
  докоммичивают в них правки (например `task01`: «ncu: add --import-source yes»).
- `profiles` — **служебная ветка бота** (коммиты «Profiles for …/pull/466 at …»).
  Хранилище отчётов профилирования. Не мёржить, не брать за основу своей работы.
  Зеркалировать в форк не обязательно.

Правило для PR (из `.github/pull_request_example.md`): ветка в форке и целевая
ветка в upstream называются **одинаково** (`task0N`), заголовок PR:
`Task0N <Имя> <Фамилия> <Аффиляция>` (имя и фамилия по-русски).

## Состояние на 2026-10-18 (после первичного зеркалирования)

- В форке `origin` есть `main` и `task00..task05`, совпадающие с upstream
  (`profiles` не зеркалировалась намеренно).
- Локальные `main` → `origin/main`, `task01` → `origin/task01`. Обычный
  `git push` идёт в форк.
- `.codesearch.db/` — индекс инструмента поиска, неотслеживаемый. Добавить в
  `.git/info/exclude`, чтобы не попал в коммиты.

## 1. Первичное зеркалирование всех веток в форк

Пушим **ссылки upstream напрямую в origin**, без локальных checkout'ов:

```bash
git fetch upstream --prune

for b in main task00 task01 task02 task03 task04 task05; do
  git push origin "refs/remotes/upstream/$b:refs/heads/$b"
done
```

PowerShell:

```powershell
git fetch upstream --prune
foreach ($b in 'main','task00','task01','task02','task03','task04','task05') {
  git push origin "refs/remotes/upstream/${b}:refs/heads/$b"
}
```

Заметки:

- Это обычный (не force) push: если ветка в форке уже содержит мои коммиты и
  разошлась с upstream, git откажет — и это хорошо, см. раздел 3.
- Origin — HTTPS. Ветки содержат `.github/workflows/cmake.yml`; пуш изменений в
  workflow-файлы требует токен/учётку с правом `workflow`, иначе GitHub отклонит
  пуш с ошибкой про `workflow` scope. Решение: обновить токен (или Git Credential
  Manager заново залогинить) либо перейти на SSH (`~/.ssh/id_ed25519` уже есть):
  `git remote set-url origin git@github.com:alexserafim/GPGPUTasks2026.git`
- `profiles` при желании: добавить в список. Она большая по числу коммитов, но
  безвредная.
- Проверка: `git ls-remote --heads origin` — должны быть все 6 task-веток и main.

## 2. Регулярное обновление (когда в upstream появились новые коммиты)

### 2.1. Посмотреть, что изменилось

```bash
git fetch upstream --prune
git log --oneline --left-right main...upstream/main        # расхождение main
git log --oneline task01..upstream/task01                   # что нового в task01
git diff --stat task01 upstream/task01                      # файлы
```

Быстро по всем веткам сразу:

```bash
for b in main task00 task01 task02 task03 task04 task05; do
  echo "== $b: +$(git rev-list --count origin/$b..upstream/$b 2>/dev/null) новых в upstream"
done
```

### 2.2. Ветки, где я **не** делал свою работу (чистое зеркало)

Просто перенести состояние upstream в форк (fast-forward, без force):

```bash
git fetch upstream
git push origin refs/remotes/upstream/main:refs/heads/main
git push origin refs/remotes/upstream/task00:refs/heads/task00   # и т.д.
```

Если нужен и локальный `main`:

```bash
git switch main
git merge --ff-only upstream/main      # или: git pull --ff-only upstream main
git push origin main
```

`--ff-only` гарантирует, что локальная `main` не обрастёт «лишними» merge-коммитами.
Если ff невозможен — значит, в `main` попали мои коммиты, а этого быть не должно;
остановиться и разобраться (`git log main..upstream/main`, `git log upstream/main..main`).

### 2.3. Ветка, где есть **моя работа** (например `task01` с решением)

Опасно: upstream может править те же файлы (README, `src/main_aplusb_matrix.cpp`,
скрипты). Порядок:

1. Убедиться, что рабочее дерево чистое: `git status`. Иначе — коммит или
   `git stash`.
2. Сделать страховочную ветку: `git branch backup/task01-$(date +%Y%m%d) task01`
3. Подтянуть изменения одним из способов:

   **Merge (безопаснее, история не переписывается — рекомендуется, если ветка уже
   запушена в форк / открыт PR):**
   ```bash
   git switch task01
   git fetch upstream
   git merge upstream/task01
   # при конфликтах: правим файлы, git add, git commit
   git push origin task01
   ```

   **Rebase (чище история, но требует force-push; только если PR ещё не смотрели
   и ветка не нужна никому другому):**
   ```bash
   git switch task01
   git rebase upstream/task01
   git push --force-with-lease origin task01
   ```
   Никогда `--force` без `-with-lease`.
4. Дождаться CI на GitHub (Actions → `cmake.yml`, запускается на push и PR).
   Если CI не стартовал — пустой коммит: `git commit --allow-empty -m "retrigger CI"`.
5. Если PR уже открыт — после пуша он обновится сам; при необходимости обновить
   вывод локального запуска и CI в описании (шаблон: `.github/pull_request_example.md`).

### 2.4. Перенос изменений из `main` в мою рабочую ветку

Ветки `taskNN` в upstream обычно содержат условие задания и уже включают нужное из
`main` на момент ответвления. Если нужно подтянуть свежее из `main` (например
исправление в `libs/`):

```bash
git switch task03
git merge upstream/main        # конфликтов быть не должно, если правил только решение
```

Предпочитаю `merge`, а не `cherry-pick` по одному коммиту — меньше риска потерять
часть правок.

## 3. Если пуш отклонён (non-fast-forward)

Значит, форк и upstream разошлись. Не форсить вслепую:

```bash
git fetch origin upstream
git log --oneline --left-right origin/task01...upstream/task01
```

- Слева (`<`) — коммиты только в форке: моя работа → делать merge/rebase (п. 2.3).
- Справа (`>`) — только в upstream: новые правки преподавателей.
- Если слева пусто, а справа есть — это обычный ff, пушится без force.
- Если на upstream переписали историю (force-push преподавателей): `git fetch
  upstream` покажет `forced update`; тогда сравнить `git range-diff
  origin/task01...upstream/task01` и осознанно решить: перенести свои коммиты
  (`git rebase --onto upstream/task01 <старая_база> task01`) или сбросить зеркало.
  Зеркальные (без моей работы) ветки в этом случае можно обновить
  `git push --force-with-lease origin refs/remotes/upstream/taskNN:refs/heads/taskNN`.

## 4. Новое задание (появилась ветка `task06` и т.д.)

```bash
git fetch upstream --prune
git branch -r | grep upstream/                       # убедиться, что ветка появилась
git push origin refs/remotes/upstream/task06:refs/heads/task06
git switch -c task06 --track origin/task06           # локальная ветка смотрит на форк
```

Также добавить имя в списки в разделах 1–2.

## 5. Коммит и пуш в мой форк

Все коммиты идут в `origin` (форк). Работа над заданием N — только в ветке `taskNN`.

### 5.1. Подготовка

```bash
git remote -v                       # origin = alexserafim/..., upstream = GPGPUCourse/...
git switch task01                   # нужная ветка
git branch -vv                      # должно быть [origin/task01]
git fetch upstream                  # нет ли новых правок преподавателей?
git log --oneline task01..upstream/task01   # пусто = обновляться не нужно (иначе раздел 2.3)
```

Если ветка ещё не создана локально:
`git switch -c task02 --track origin/task02`.

### 5.2. Что коммитить и что нет

Коммитим: исходники решения (`src/*.cpp`, `src/kernels/**`), правки `CMakeLists.txt`,
если они нужны заданию.

Не коммитим (уже в `.gitignore`: `/build*/`, `cmake-build*`, `.idea`, `.vs`,
`generated_kernels/*`, `*_cl.h`) — сборочные каталоги, IDE-файлы, артефакты.
Дополнительно локально исключить индекс поиска:

```bash
echo ".codesearch.db/" >> .git/info/exclude
```

Не добавлять большие бинарные файлы и профили (`.ncu-rep` и т.п.) без необходимости.

### 5.3. Коммит

```bash
git status                          # что изменено / не отслеживается
git diff                            # просмотреть правки
git add src/main_aplusb_matrix.cpp src/kernels/cl/aplusb_matrix.cl   # явные пути, не `git add -A`
git diff --staged                   # проверить, что в коммит попало только нужное
git commit -m "task01: coalesced и non-coalesced кернелы сложения матриц"
```

Рекомендации: явно перечислять файлы вместо `git add .`; одно логическое изменение —
один коммит; сообщение коротко описывает *что* и *зачем*. Хуки (`--no-verify`) не обходить.

### 5.4. Пуш в форк

```bash
git push origin task01              # или просто `git push`, трекинг уже на origin
```

Первый пуш новой локальной ветки: `git push -u origin task02`.

Проверка, что дошло:

```bash
git status -sb                      # ## task01...origin/task01 (без ahead/behind)
git log --oneline origin/task01 -3
```

и на GitHub: https://github.com/alexserafim/GPGPUTasks2026/tree/task01 — вкладка
Actions покажет сборку `cmake.yml` (запускается на каждый push).

### 5.5. Типичные проблемы

| Симптом | Причина / решение |
|---------|-------------------|
| `rejected ... (non-fast-forward)` | В форке есть коммиты, которых нет локально: `git fetch origin`, затем `git pull --rebase origin task01` или merge (см. раздел 3). Не форсить. |
| `refusing to allow ... to create or update workflow` | У токена нет права `workflow`, а коммит меняет `.github/workflows/*`. Обновить токен или перейти на SSH. |
| push уходит в upstream / 403 | Ветка отслеживает `upstream/...`: `git branch --set-upstream-to=origin/<ветка> <ветка>`. |
| Закоммитил лишнее | Если ещё не запушено: `git reset --soft HEAD~1`, поправить, закоммитить заново. Если запушено — новый коммит, `reset` + force только через `--force-with-lease` и только в своей ветке. |
| Коммит не в той ветке | Если не запушено: `git switch нужная_ветка && git cherry-pick <sha>`, затем в неверной ветке `git reset --hard origin/<неверная>` (только после проверки, что коммит уже есть в нужной). |
| Нужно отложить правки перед обновлением | `git stash -u`, после операции `git stash pop`. |

## 6. Отправка работы (PR)

1. Работа ведётся в `taskNN`, трекается `origin/taskNN`.
2. `git push origin taskNN`.
3. На GitHub: PR из `alexserafim:taskNN` → `GPGPUCourse:taskNN`
   (ветки **одноимённые**), заголовок `TaskNN Имя Фамилия Аффиляция`.
4. Описание — по `.github/pull_request_example.md` (локальный вывод + вывод CI).

## 7. Чеклист и шпаргалка

Перед любой операцией:

- [ ] `git status` чистый (или stash)
- [ ] `git remote -v` — origin = форк, upstream = курс
- [ ] `git fetch upstream --prune` (и `git fetch origin`)
- [ ] понимаю, на какой ветке стою и куда она пушится: `git branch -vv`

Никогда:

- не пушить в `upstream` (нет прав, и нужно держать `upstream` read-only; можно
  заблокировать: `git remote set-url --push upstream DISABLED`);
- не делать `git push --force` — только `--force-with-lease`;
- не мёржить `profiles` в рабочие ветки.

Полезное:

```bash
git remote set-url --push upstream DISABLED           # защита от случайного пуша в чужой репозиторий
git branch -vv                                        # кто на что смотрит
git ls-remote --heads origin                          # что реально лежит в форке
git config --get-regexp '^branch\.'                   # настройки трекинга
git log --oneline --graph --all --decorate -30        # картина целиком
```
