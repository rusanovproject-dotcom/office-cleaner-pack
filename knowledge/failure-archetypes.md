# Failure Archetypes — 8 типов сбоев AI-агента

**Источники:**
- Arize «Why AI Agents Break» (`https://arize.com/blog/common-ai-agent-failures/`)
- MindStudio «6 Ways Agents Fail» (`https://www.mindstudio.ai/blog/ai-agent-failure-pattern-recognition`)
- Partnership on AI «Real-Time Failure Detection in Agents»

**Зачем здесь:** Failure Pattern Detector (Модуль 6 Риты) использует эти archetypes как словарь для группировки записей в `failures.md` агентов. Если 3+ записей в офисе ≤5 агентов (или 5+ для большего) попадают в один archetype за 30 дней — предложение перенести правило в `core.md` агента.

---

## 8 archetypes

### 1. Premature action without grounding

**Что:** агент действует без контекста — не прочитал что нужно, начал делать.

**Сигналы в `failures.md`:**
- «не прочитал memory перед задачей»
- «начал писать оффер без чтения JTBD_анализа»
- «пропустил Pre-flight check»

**Правило в `core.md`:** Pre-flight gate с обязательным чтением (как в Алексе: `agent-state.md` + `memory.md` + `failures.md`).

### 2. Over-helpfulness substituting missing entities

**Что:** додумывает чего нет. Цифры, имена, цитаты.

**Сигналы:**
- «придумал кейс клиента которого нет»
- «сгенерировал цитату вместо дословной»
- «оценил рынок цифрой без источника»

**Правило:** все непроверенное → `[ГИПОТЕЗА, уверенность X%]`. Дословные цитаты — никогда не перефразировать.

### 3. Distractor-induced context pollution

**Что:** отвлёкся на постороннее. Скилл вызвался не на свою тему.

**Сигналы:**
- «пользователь спросил про X, ответил про Y»
- «вместо JTBD начал писать оффер»
- «сорвал scope в середине Step 3»

**Правило:** Stage Lock + срыв scope = обязательная запись в `failures.md` до продолжения.

### 4. Fragile execution under load

**Что:** ломается на длинной сессии или при большом контексте.

**Сигналы:**
- «после 50K токенов забыл что было в начале»
- «потерял текущий шаг JTBD»
- «agent-state.md не обновлён после прерывания»

**Правило:** compact at 60% (не 95%). Save state каждые N взаимодействий. interrupted-флаг в `agent-state.md`.

### 5. Tool use avoidance

**Что:** игнорирует инструменты, делает руками. Например — вместо grep по failures.md «вспоминает».

**Сигналы:**
- «не сделал grep failures, повторил ошибку»
- «не прочитал routing-patterns»
- «не запустил `/office-cleaner` хотя пользователь попросил «проверь офис»»

**Правило:** self-trigger rules с явными «ОБЯЗАТЕЛЬНО прочитай X перед Y». Чек-лист Pre-flight.

### 6. Looping

**Что:** самый частый failure-mode. Агент застревает в цикле «попробовал → не получилось → попробовал ещё раз».

**Сигналы:**
- «3 раза предложил одно и то же»
- «correctional prompts не дают результат»
- «итерации Validator → Refiner без сходимости»

**Правило:** **2 fails — стоп**. Останавливаюсь, говорю «две попытки не сработали, подхожу иначе». Hard cap на iterations (Validator: max 2 раунда Refiner; Рита: max 3 прохода по одному файлу).

### 7. Last-mile execution failures

**Что:** не довёл до конца. 90% сделал — последние 10% не закрыл.

**Сигналы:**
- «собрал отчёт но не сохранил в `office/ops/audits/`»
- «не сделал append в memory.md после задачи»
- «не записал failure в failures.md после промаха»

**Правило:** Phase 5 «Memory Update» обязательная во всех скиллах. Self-check перед отдачей.

### 8. Coherence degradation under extended operation

**Что:** на длинной сессии теряет когерентность — стиль плывёт, тон меняется, противоречит сам себе.

**Сигналы:**
- «в начале сессии говорил X, в конце Y»
- «тон поменялся с партнёрского на корпоративный»
- «перестал соблюдать voice.md правила»

**Правило:** save state + checkpoint при >70% контекста. New session с чистым state.json. Voice-check каждые N сообщений.

---

## Как Рита использует эти archetypes

В deep-режиме Failure Pattern Detector:

```
1. Собрать все записи из office/agents/*/failures.md за 30 дней
2. Для каждой записи попробовать сматчить с одним из 8 archetypes (через keyword overlap)
3. Сгруппировать по archetype + agent
4. Если group.size >= 3 (для офиса ≤5 агентов) ИЛИ >= 5 (для офиса >10 агентов):
   - Сформировать finding M6.X — Repeating pattern
   - Предложить добавить правило в `core.md` агента секцию Guardrails
5. Если 3+ агентов имеют записи в одном archetype:
   - Сформировать finding M6.X — Cross-agent pattern
   - Предложить добавить правило в корневой `CLAUDE.md` или `protocols/CLEANUP.md`
```

Output формы:

```markdown
**Finding M6.1 — Repeating pattern (Tool use avoidance)**
Files: office/agents/alex-marketer/failures.md (4 записи 2026-04-15..2026-04-30)
Issue: 4 раза агент не сделал grep по failures.md перед задачей,
       повторил ту же ошибку (cited: failures.md:42, :78, :115, :142)
Impact: учится медленно, паттерн закрепляется
Fix: добавить в core.md секция Guardrails:
     «ОБЯЗАТЕЛЬНО grep по failures.md ключевые слова задачи перед Pre-flight»
Severity: P1
```
