# Architect of Order — install manifest

Машиночитаемые метаданные для установки пака через скилл `/install-agent architect-of-order`.

---

## Metadata

```yaml
agent_id: architect-of-order
agent_name_human: Рита — Хранительница офиса
agent_name_in_chat: Рита
short_role: |
  Рита — Хранительница офиса. Единственный агент с правом ревизии структуры, памяти и
  качества команды. Не делает контентную работу — следит чтобы офис не превращался в свалку.
  Три режима: scan (5 мин, read-only inventory), tidy (15 мин, soft-fixes под approval),
  deep (60 мин, полный аудит по 6 архетипам уборщиков + 8-мерной Florian-rubric).
trigger_keywords:
  - рита
  - Рита
  - позови риту
  - позови Риту
  - рита проверь
  - рита наведи порядок
  - рита, проверь офис
  - рита, наведи порядок
  - рита аудит
  - рита почисти
  - ритуся
  - хранительница
  - хранительница офиса
  - наведи порядок
  - проверь офис
  - офис аудит
  - office audit
  - аудит офиса
  - аудит
  - есть дыры
  - что не так с офисом
  - почистим офис
  - ревизия офиса
  - ревизия
  - стало мусорно
  - почему офис тормозит
  - архитектор офиса
  - архитектор-порядка
  - architect of order
  - office architect
  - /office-architect
  - /architect-scan
  - /architect-tidy
  - /architect-deep
version: 1.0.1
requires: []  # standalone agent — не зависит от других паков
provides_pipeline:
  - office-architect-pipeline  # 5 фаз: Inventory → Audit → Report → Approval → Memory Update
supersedes:
  - audit-project   # старый скилл `/audit-project` — install автоматически перенесёт его в `.claude/skills/_archive/audit-project-superseded/` (если ещё не перенесён)
```

---

## Files to copy

Источник — `_agent-packs/architect-of-order/`, цель — `office/agents/architect-of-order/`. Копируется рекурсивно.

```yaml
files:
  - src: core.md
    dest: office/agents/architect-of-order/core.md
  - src: soul.md
    dest: office/agents/architect-of-order/soul.md
  - src: CLAUDE.md
    dest: office/agents/architect-of-order/CLAUDE.md
  - src: overrides.md
    dest: office/agents/architect-of-order/overrides.md
    preserve_if_exists: true
  - src: memory.md
    dest: office/agents/architect-of-order/memory.md
    preserve_if_exists: true
  - src: failures.md
    dest: office/agents/architect-of-order/failures.md
    preserve_if_exists: true
  - src: knowledge/
    dest: office/agents/architect-of-order/knowledge/
    recursive: true
  - src: skills/
    dest: .claude/skills/
    recursive: true

# Обязательная папка для отчётов аудитов — Архитектор пишет только сюда.
folders:
  - dest: office/ops/audits/
    create_if_missing: true
    initial_file:
      path: office/ops/audits/.gitkeep
      content: ""
  - dest: .claude/skills/_archive/
    create_if_missing: true   # для архивации устаревших скиллов (audit-project и т.д.)
```

---

## Pre-install: archive superseded skills

**Перед копированием** проверь и перенеси устаревшие скиллы которые дублируют функционал Архитектора:

```yaml
archive_skills:
  - source: .claude/skills/audit-project/
    dest: .claude/skills/_archive/audit-project-superseded/
    if_exists: true
    add_deprecation_note: |
      Этот скилл заменён на `/office-architect` (агент Архитектор-Порядка).
      Сохранён в архиве для истории. Не запускается.
    update_frontmatter:
      status: deprecated
      superseded_by: office-architect
      archived_on: <today>
```

Если папка `.claude/skills/audit-project/` отсутствует (уже заархивирована или никогда не существовала) — пропускаем шаг без ошибки.

После архивации install-agent сообщает в post-install: *«Старый скилл `/audit-project` переехал в архив, его триггеры теперь у Архитектора офиса.»*

---

## Critical security: enforce settings.json

**Файл `.claude/settings.json` ОБЯЗАТЕЛЕН для установки этого агента.** Архитектор имеет в `allowed-tools` `Read`/`Edit` без узкого scope — единственная реальная защита `.env*` и других секретов идёт через `permissions.deny` в settings.json.

```yaml
settings_json:
  ensure_exists: true
  template_path: .claude/settings.json   # шаблон уже есть в client-office-template
  required_deny_rules:
    - "Read(.env)"
    - "Read(.env.*)"
    - "Read(**/.env)"
    - "Read(**/.env.*)"
    - "Read(*.pem)"
    - "Read(**/*.pem)"
    - "Read(credentials*)"
    - "Read(**/credentials*)"
    - "Read(secrets*)"
    - "Read(**/secrets*)"
    - "Edit(.env)"
    - "Edit(.env.*)"
    - "Edit(**/.env)"
    - "Edit(**/.env.*)"
    - "Write(.env)"
    - "Write(.env.*)"
    - "Write(**/.env)"
    - "Write(**/.env.*)"
    - "Bash(rm:*)"
    - "Bash(rm -rf:*)"
    - "Bash(git push --force:*)"
    - "Bash(git push --force-with-lease:*)"
    - "Bash(git reset --hard:*)"
  on_missing: create_from_template   # install копирует шаблон settings.json из корня шаблона
  on_partial: merge_deny_rules        # если файл есть, но deny неполный — допиши недостающие правила (deep_merge)
```

Поведение install-agent:
1. Если `.claude/settings.json` отсутствует — копирует шаблон из `client-office-template/.claude/settings.json`. Сообщает пользователю: *«Создал .claude/settings.json с защитой .env и опасных команд — это часть базовой настройки.»*
2. Если файл есть, но deny-правила неполные — делает deep merge, добавляет недостающие. Сохраняет существующий `permissions.allow` нетронутым.
3. Если файл есть и все нужные правила уже на месте — пропускает шаг.

**Без этого шага установку не завершать.** Защита `.env` через prompt-инструкции (`core.md` G4) — недостаточно.

---

## Updates to existing files

```yaml
updates:
  - file: CLAUDE.md
    section: "## Обязательный layered include при старте"
    add_line: "@office/agents/architect-of-order/core.md"

  - file: office/AGENTS.md
    section: "## Активная команда"
    add_row: |
      | **Рита** (Хранительница офиса) | Ревизия структуры офиса, памяти, роутинга. Чистит дубли, ловит коллизии триггеров, собирает паттерны ошибок. Три режима — быстрый скан, лёгкая чистка, полный аудит. Никаких правок без твоего согласия. | *«рита», «позови риту», «рита проверь офис», «рита наведи порядок», «аудит», «office audit», «стало мусорно»* |

  - file: office/agents/director/core.md
    section: "## Команда офиса"
    add_bullet: |
      - **Рита (Хранительница офиса)** — следит за порядком в команде. Ревизует структуру, чистит память, ловит коллизии. Не делает контент — только следит чтобы офис не зарастал. Три режима: быстрый скан / лёгкая чистка / полный аудит. Зовётся напрямую: «рита, проверь офис» / «рита, наведи порядок».

  - file: office/agents/director/core.md
    section: "## Роутинг (knowledge/routing-patterns.md — расширенные паттерны)"
    # Заменяем старую строку про /audit-project (Демиург) на новую про /office-architect (Архитектор офиса).
    # Старый вариант: "проверь офис" / "наведи порядок" / "есть дыры" / "аудит офиса" → Архитектор через /audit-project
    # `audit-project` устарел и архивируется (см. секцию `Pre-install`). Демиург по-прежнему делает /build (сборка нового агента),
    # но НЕ делает аудит — это теперь зона Архитектора-Порядка.
    replace_line:
      match: '"проверь офис" / "наведи порядок" / "есть дыры" / "аудит офиса"'
      with: |
        - "рита" / "позови риту" / "рита, проверь" / "рита, наведи порядок" / "проверь офис" / "наведи порядок" / "есть дыры" / "аудит" / "ревизия" / "стало мусорно" / "почему тормозит" → **Рита (Хранительница офиса)** через `/office-architect` (по умолчанию режим scan; «полный аудит» → deep, «почистим» → tidy)
    add_rows: |
      - "что не так с офисом" / "office audit" / "рита, полный аудит" → **Рита** через `/office-architect` (deep)

  - file: office/agents/director/knowledge/routing-patterns.md
    section: "## Core-роутинг"
    # Заменяем строку про /audit-project (она была единственная про аудит) на три новые про /office-architect.
    replace_row:
      match_contains: "/audit-project"
      with_rows: |
        | "рита" / "позови риту" / "рита, проверь" / "проверь офис" / "наведи порядок" / "есть дыры" / "аудит" / "ревизия" / "стало мусорно" | **Рита (Хранительница офиса)** → `/office-architect` | scan по умолчанию (5 мин read-only); «полный аудит» / «что не так» → deep; «почистим» → tidy |
        | "рита, почисти" / "почистим офис" / "лёгкая чистка" | **Рита** → `/office-architect tidy` | 15 мин под approval, soft-fixes только |
        | "рита, полный аудит" / "полный аудит" / "deep audit" / "что не так с офисом" | **Рита** → `/office-architect deep` | 60 мин, все 6 архетипов + 8-мерная Florian-rubric |
    # Уточняем существующее правило про Демиурга чтобы не было путаницы — он всё ещё делает /build, но не аудит.
    add_note_after_table: |
      > **Что важно:** аудит офиса делает **Рита (Хранительница офиса)** через `/office-architect`, а не Демиург. Демиург остаётся ответственным за `/build` (сборка новых агентов) и реорганизацию структуры по запросу. Если пользователь говорит «рита» / «аудит» / «проверь офис» — роут на Риту.
```

---

## Post-install message to client

```
✅ Новый помощник в команде — Рита, Хранительница офиса.

Думай о ней как о кураторе музея. Не громит, не выкидывает — спокойно
ходит между экспонатами, видит где пыль, где экспонат сместился, где
табличка устарела. Предлагает реставрацию, ничего не делает без твоего «да».

Она не пишет тексты и не делает дизайн. У неё одна работа: смотреть
чтобы офис не зарастал. Через месяц-другой у любого офиса накапливается
мусор — устаревшие записи, дубли, потерянные ссылки. Рита ходит
и наводит порядок.

Я заодно поставил защиту, чтобы она случайно не залезла куда не надо:
личные ключи, пароли, файлы окружения — она их не видит и не трогает.
Опасные команды на удаление и перезапись — заблокированы. Это базовая
безопасность для всего офиса, не только для неё.

Три режима — выбираешь когда нужно. Зови по имени — она откликается:

1. **Быстрый осмотр** (5 минут) — посмотрит и скажет что хорошо, что
   просело. Ничего не трогает. Скажи «рита, проверь офис».

2. **Лёгкая уборка** (15 минут) — посмотрит и приберёт мелочи. Перед
   каждым изменением спросит твоё «да». Скажи «рита, почистим офис».

3. **Глубокий аудит** (час) — всё разложит по полочкам с оценкой 0-100
   и списком что улучшить. Скажи «рита, полный аудит».

Все отчёты копятся в одной папке с датой — через месяц увидишь как офис
стал лучше.

Главное: Рита никогда не удаляет файлы насовсем (только переносит
в архив с датой) и никогда не меняет ничего без твоего «да».
```

---

## First-task suggestion

После установки агента — предлагаем безопасный первый прогон (read-only scan):

```yaml
first_task:
  suggestion: "быстрый осмотр офиса"
  why: "5 минут, ничего не трогает — увидишь актуальную картину структуры и где что просаживается. После сможешь решить что чинить."
  skill: office-architect
  trigger_phrase: "рита, проверь офис"
  mode: scan
  safety_note: "режим осмотра — только смотрю, ничего не трогаю"
```

Сообщение в конце post-install:

> *«Давай Рита начнёт с быстрого осмотра? 5 минут, ничего не трогает — просто покажет что в офисе сейчас в порядке, а что просело. Скажи "рита, проверь офис" и поехали.»*

---

## Uninstall (future)

Для будущей поддержки `/uninstall-agent architect-of-order`:

```yaml
uninstall:
  remove_folders:
    - office/agents/architect-of-order/
    - .claude/skills/office-architect/
  remove_lines_from:
    - path: office/AGENTS.md
      match: "**Архитектор офиса**"
    - path: CLAUDE.md
      match: "@office/agents/architect-of-order/core.md"
    - path: office/agents/director/core.md
      match: "**Архитектор офиса**"
    - path: office/agents/director/knowledge/routing-patterns.md
      match: "/office-architect"
  preserve:
    - office/ops/audits/   # история аудитов — клиентские данные, не трогать
    - .claude/settings.json   # защита `.env*` нужна другим агентам тоже
    - .claude/skills/_archive/   # деприкейтед скиллы
  warn:
    - "Удаление Архитектора офиса не возвращает /audit-project — он остаётся в архиве. Если хочешь старый скилл обратно — ручной revive из .claude/skills/_archive/audit-project-superseded/."
```
