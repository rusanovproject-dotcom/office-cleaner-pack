# AutoDream Rules — правила консолидации памяти

**Источник:** `zenvanriel.com/ai-engineer-blog/claude-code-autodream-memory-consolidation-guide/`. AutoDream — background sub-agent в Claude Code, который консолидирует MEMORY.md между сессиями, имитируя биологическую консолидацию памяти во время REM-сна.

**Зачем здесь:** Memory Cleaner (Модуль 1 Архитектора) реализует те же 4 категории чистки что AutoDream + adjusted thresholds для клиентского профиля.

---

## 4 категории чистки (AutoDream-канон)

### 1. Redundancy — дубли

**Что:** одна и та же запись в MEMORY.md появляется несколько раз (в разных сессиях, через копирование).

**Пример:**
```markdown
- 2026-04-15 — Никита предпочитает короткие сообщения
- 2026-04-22 — короткие сообщения предпочтение Никиты  ← дубль того же факта
- 2026-04-30 — пользователь хочет короткие ответы          ← ещё дубль
```

**Правило:** объединить в один блок с самой свежей датой, убрать остальные. Старые версии в `_archive/<YYYY-MM-DD>/memory-pre-dedup.md`.

### 2. Contradictions — конфликты

**Что:** две записи противоречат друг другу. AutoDream правило: «keep the current truth», убрать stale.

**Пример:**
```markdown
- 2026-03-10 — Используем PostgreSQL для кэша
- 2026-04-25 — Перешли на Redis для кэша  ← новая правда, PostgreSQL устарел
```

**Правило:** новая правда вытесняет старую (timestamp решает). Старая — в `_archive/`. Финальная запись в memory.md:
```markdown
- 2026-04-25 — Используем Redis для кэша (заменил PostgreSQL, см. _archive/2026-05-04/memory-cache-decision.md)
```

### 3. Stale Timestamps — относительные даты

**Что:** «yesterday», «last week», «недавно», «пару дней назад» в memory. Через 2 недели уже непонятно когда это было.

**Пример:**
```markdown
- yesterday Никита решил перенести релиз     ← stale, дата размылась
```

**Правило:** конвертировать в ISO YYYY-MM-DD. Если контекст потерян — пометить `[date unclear, ~mid-April]` или удалить.

В tidy-режиме это **safe auto-fix** (one-clear-correct-fix) — Архитектор может править `yesterday` → ISO с подтверждением даты у пользователя.

### 4. Outdated Debugging Notes — мёртвые ссылки

**Что:** записи ссылаются на удалённые файлы или функции которых больше нет.

**Пример:**
```markdown
- 2026-03-15 — Bug в `office/agents/cherry/core.md:42` — починили через X
                 ← но cherry-агент удалён, файла нет
```

**Правило:** если файл из ссылки физически отсутствует — flag для archive. Не удалять автоматически (может содержать ценный урок), но пометить `[dead reference, file removed YYYY-XX-XX]`.

---

## Принцип ранжирования (AutoDream)

> «Long-term relevance over short-term specifics.»

Заметка про testing framework важнее чем конкретный bug fix. При архивации:
- **Сохраняй:** правила, паттерны, принципы, dependency-кейсы
- **Архивируй:** конкретные one-off bug fixes старше 90 дней

---

## Триггеры (adjusted для клиентского профиля)

**AutoDream базовое правило:** `>=24h && >=5 sessions` — рассчитано на ежедневного user'а.

**Клиентский профиль:** клиент использует офис нерегулярно (2-3 сессии в неделю). Если ждать 5 сессий — cleanup не сработает за 2 недели.

**Adjusted thresholds:**

| Профиль | Trigger | Cap времени |
|---------|---------|-------------|
| Daily user (≥1 session/day) | `>=24h && >=5 sessions` | 8-10 мин |
| Active client (≥3 sessions/week) | `>=72h && >=3 sessions` | 8-10 мин |
| Sporadic (<3 sessions/week) | `>=168h (7 days) && >=2 sessions` | 5-8 мин |

**В клиентском шаблоне:** Архитектор-Порядка использует **manual trigger** через `/office-architect` — клиент сам решает когда. Cron / hook не имплементируется в MVP.

---

## Защита от over-consolidation

Из cleanup-anti-patterns.md класс 5 (False positives на специфике):

1. **Не консолидируй `failures.md`.** Это append-only history агента, его опыт. Старые записи нужны для grep перед задачей.
2. **Не консолидируй `_archive/`.** Это история, она и так в архиве.
3. **Не консолидируй memory.md если меньше 50 строк.** Маленький — не нуждается в чистке.
4. **Уважай `audit-rules: skip-consolidation` в frontmatter.** Если файл объявил себя exempt — пропускай.

---

## Output формы (для Memory Cleaner)

```markdown
**Finding M1.X — Redundancy**
Files: office/agents/<name>/memory.md:42, :78, :115
Issue: Один и тот же факт «<факт>» повторяется 3 раза
Impact: засоряет grep, увеличивает токен-budget при load
Fix: объединить в одну запись с датой 2026-XX-XX, остальные → _archive/<date>/memory-pre-dedup.md
Severity: P2

**Finding M1.X — Contradiction**
Files: office/agents/<name>/memory.md:42 vs :118
Issue: «<правило A>» (line 42) vs «<правило B>» (line 118)
Impact: агент не знает какому правилу верить, поведение непредсказуемо
Fix: оставить новое (с timestamp <дата>), старое → _archive/
Severity: P1

**Finding M1.X — Stale Timestamp**
Files: office/agents/<name>/memory.md:42, :78
Issue: 2 записи с относительными датами («yesterday», «недавно»)
Impact: через месяц непонятно когда был факт
Fix: конвертировать в ISO. В tidy-режиме — safe auto-fix.
Severity: P2

**Finding M1.X — Outdated Debugging Notes**
Files: office/agents/<name>/memory.md:42
Issue: ссылается на office/agents/cherry/core.md:42 — файл удалён 2026-04-XX
Impact: dead link, контекст потерян
Fix: пометить как [dead reference] или archive если запись о устаревшем bug fix
Severity: P2
```
