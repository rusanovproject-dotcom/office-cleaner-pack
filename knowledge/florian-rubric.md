# Florian-rubric — 8-мерная адаптированная рубрика для клиентского AI-офиса

**Источник:** Florian Bruniaux, `claude-code-ultimate-guide/tools/audit-prompt.md` v5.0 (700 строк, скачано в `/tmp/audit-prompt-florian.md`).
**Адаптация:** клиентский офис ≠ dev-офис. Веса смещены с MCP/security акцентов на память, identity, knowledge hygiene.
**Веса в сумме:** 100.

---

## Tier'ы

| Score | Tier | Характеристика |
|-------|------|----------------|
| <40 | **Starter** | Офис только что стартовал или сильно запущен. Много P0, фундамент шатается |
| 40-59 | **Growing** | Работает, но много дыр. Есть P0 которые нужно закрыть до клиентов |
| 60-79 | **Established** | Рабочий офис, точечные проблемы. Работа над P1 в течение 1-4 недель |
| 80+ | **Optimized** | Эталон. Только P2 cleanup. Можно отдавать как пример |

---

## Dimensions

### 1. Memory & Context (max 20)

Что мерим: **token budget фиксированного контекста**, который грузится при каждой сессии.

Считаем: размер корневого `CLAUDE.md` + все `@office/agents/*/core.md` через `@include` + `office/CLAUDE.md` (если есть). В символах * 0.25 ≈ токенов.

| Размер | Score | Заметка |
|--------|-------|---------|
| <20K токенов | 18-20 (зелёный) | Эталон. Claude видит задачу с большим запасом |
| 20-40K | 12-17 (жёлтый) | Терпимо, но при росте офиса начнёт давить |
| 40-60K | 6-11 (оранжевый) | Заметные потери качества на длинной сессии |
| >60K | 0-5 (красный) | Критично. Контекст съеден до того как Claude увидел запрос |

**Quick wins:**
- Вынести «Стек инструментов», «Edge cases» из core.md в `knowledge/<agent>/`
- Сжать корневой CLAUDE.md ≤200 строк
- Большие knowledge-карточки → `references/` рядом со скиллом

### 2. Memory Hygiene (max 15) — наш custom

Что мерим: **наполняются ли `failures.md` и `memory.md` агентов**, обновляется ли `MEMORY.md` (если есть).

| Чек | Балл |
|-----|------|
| Все агенты имеют `failures.md` | 2 |
| Все агенты имеют `memory.md` | 2 |
| failures.md наполнен (≥1 запись) у активно используемых агентов | 3 |
| memory.md в лимитах (soft 500, hard 1000 строк) | 2 |
| Записи за последние 30 дней (если офис активный) | 3 |
| `.claude/MEMORY.md` (если есть) ≤200 строк, структурирован | 1 |
| Шаблон унифицирован (Decisions / Patterns / Context) | 2 |

**P0 индикатор:** все `failures.md` пусты при работающем офисе >7 дней — обучаемость нулевая.

### 3. Routing & Skills Quality (max 15)

Что мерим: достижимость агентов + качество SKILL.md.

| Чек | Балл |
|-----|------|
| Каждый агент в `office/agents/*` упомянут в `office/AGENTS.md` | 2 |
| Каждый агент в AGENTS.md физически существует (нет orphan) | 2 |
| Триггеры скиллов не пересекаются (parsing trigger_keywords) | 3 |
| В корневом CLAUDE.md есть `@office/agents/<name>/core.md` для каждого | 2 |
| Все SKILL.md имеют YAML frontmatter | 2 |
| Все SKILL.md имеют `description:` с TRIGGERS | 2 |
| Размер: SKILL.md ≤500 строк, core.md ≤200 строк (или exemption через `audit-rules:`) | 2 |

### 4. Hooks & Settings (max 10)

Что мерим: repo-level governance.

| Чек | Балл |
|-----|------|
| `.claude/settings.json` существует | 2 |
| `permissions.deny` для `.env*`, `Bash(rm:*)`, `Bash(git push --force)` | 3 |
| `.claude/.mcp.json` существует | 2 |
| `.claude/hooks/` существует с минимум 1 hook | 2 |
| SessionStart hook читает context | 1 |

### 5. Identity & Voice (max 10)

Что мерим: соул-слой агентов.

| Чек | Балл |
|-----|------|
| Каждый агент имеет уникальный `core.md` ≤200 строк | 2 |
| Каждый агент имеет `soul.md` (если у других есть — синхронно у всех) | 2 |
| Output contract явный в каждом агенте (что именно возвращает) | 2 |
| Тон/voice описаны конкретно (фразы-маркеры, стоп-слова) | 2 |
| Нет AI-слопа в инструкциях («рад помочь», «отличный вопрос», «инновационный») | 2 |

### 6. Knowledge Hygiene (max 10)

Что мерим: чистота knowledge-базы.

| Чек | Балл |
|-----|------|
| `knowledge/INDEX.md` существует и актуален | 2 |
| Папка с ≥3 файлами имеет INDEX.md (правило DEF-OF-DONE) | 2 |
| Нет heading collisions (две карточки с одним H1) | 2 |
| Jaccard на heading sets между парами файлов <0.7 (нет дублей) | 2 |
| Граница общий vs agent-specific knowledge явная | 2 |

### 7. Freshness (max 10)

Что мерим: актуальность.

| Чек | Балл |
|-----|------|
| git activity <90 дней (последний commit) | 3 |
| Нет deprecated моделей (`claude-2`, `gpt-3.5`, `claude-instant`, `claude-3-haiku`) | 3 |
| `[LATEST YYYY-XX-XX]` маркеры в memory свежие (если используются) | 2 |
| Frontmatter `updated:` <90 дней при активном использовании файла | 2 |

### 8. Security Posture (max 10)

Что мерим: защита от утечек.

| Чек | Балл |
|-----|------|
| `.env` в `.gitignore` | 2 |
| Нет hardcoded secrets в файлах (`sk-`, `ghp_`, `xox[baprs]-`, `AKIA`) | 3 |
| `permissions.deny` в settings.json для `.env*` | 2 |
| Pre-commit hook gitleaks или эквивалент | 2 |
| Нет `disableSkillShellExecution` в settings | 1 |

---

## Расчёт итогового score

```
Total = (1) Memory & Context (20)
      + (2) Memory Hygiene (15)
      + (3) Routing & Skills (15)
      + (4) Hooks & Settings (10)
      + (5) Identity & Voice (10)
      + (6) Knowledge Hygiene (10)
      + (7) Freshness (10)
      + (8) Security Posture (10)
      = 100

Tier:
  <40   → Starter
  40-59 → Growing
  60-79 → Established
  80+   → Optimized
```

---

## Что НЕ забывать при scoring

1. **Каждый балл обосновывается** — нельзя «потолочно». Нужна цитата `file:line`.
2. **Размер офиса учитывается:** офис из 3 агентов и 5 скиллов — не сравнивать с 17-агентным.
3. **Тип офиса учитывается:** marketing-офис vs dev-офис — разные приоритеты.
4. **Молодой офис (<1 месяц) — Starter не равно Bad.** Это нормально для первого месяца.
5. **«Looks bad but is actually fine»** — обязательная секция отчёта, исключения из penalty.
