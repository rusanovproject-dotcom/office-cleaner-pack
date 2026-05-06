# Cleanup Anti-patterns — что точно НЕ делать

**Источники:**
- ksimback `tech-debt-skill/SKILL.md`
- Anthropic Memory Tool API docs
- Compound Engineering `document-review` plugin
- Claude AI Cursor Database Deletion incident (2026-04-28)

**Зачем:** 5 классов anti-patterns когда уборщик становится врагом. Каждый класс — с конкретным правилом-защитой.

---

## 5 классов anti-patterns

### 1. Удаление нужного

**Реальный кейс (2026-04-28):** Claude-powered Cursor agent удалил production-базу за 9 секунд. Источник: cxtoday.com, futurism.com, euronews.com.

**Защита (Anthropic Memory Tool API):**
> «Every change to a memory creates an immutable memory version, giving you an audit trail and point-in-time recovery for everything the agent writes.»
> «Versions belong to the store (not the individual memory) and survive even after the memory itself is deleted.»
> «Archiving makes a store read-only and prevents it from being attached to new sessions. Archiving is one-way; there is no unarchive.»

**Правила Риты:**
1. **Append-only > delete.** Старые версии в `_archive/<YYYY-MM-DD>/` с датой. Не stomp.
2. **Soft-delete + redact.** Move в `_archive/`, не `rm`. Если PII — `[REDACTED]` в отчёте.
3. **Двойная защита:** allowed-tools НЕ содержит `Write`, `Bash(rm:*)`, `Bash(mv:*)`. Settings.json deny-rule на `Edit(.env*)`, `Bash(rm:*)`.
4. **Approval gate:** Phase 4 yes/«только важное»/«1,3»/нет — закон. Без явного ответа — стоп.

### 2. Over-cleaning (выкосил полезное)

Из tech-debt-skill: *«Read code before judging it — a pattern that looks wrong in isolation may be load-bearing.»*

**Реальные load-bearing patterns в нашем шаблоне:**
- `failures.md` >300 строк — это feature (append-only), не bug
- Symlink между папками — может быть фундамент архитектуры
- `prework.md` с YAML-frontmatter — template для setup, не полузаполнение
- `_archive/*` — намеренно история, не редактируется
- `_template/`, `_template-audience/` — два разных шаблона, не дубль (хотя пересекаются)
- Memory у Demiurg в другом формате чем у Director — намеренно (Demiurg специфический)

**Правила Риты:**
1. **Section "Looks bad but is actually fine" обязательна** в каждом отчёте. Если пуста → отчёт FAIL, не сохраняется. Это форсирует thinking.
2. **Auto-fix только для one-clear-correct-fix.** Tier-1 findings: frontmatter правки, datestamps, INDEX.md обновления. Всё остальное — рекомендация в отчёте.
3. **Approval gate (Phase 4).** Никаких изменений без явного yes / «только важное» / «1,3» / нет.
4. **Reading code before judging.** Если паттерн «выглядит плохо» — поищу ли я зачем оно. 1-2 параграфа mental model перед finding.

### 3. Looping cleanup (бесконечные уборки)

Из failure-таксономий — **looping** самый частый failure-mode. Уборщик может застрять в цикле «нашёл противоречие → попытался разрешить → создал новое противоречие → снова нашёл».

**Правила Риты:**
1. **Hard cap на время.** scan ≤5 мин, tidy ≤15 мин, deep ≤60 мин. AutoDream rule: 8-10 минут на цикл.
2. **Hard cap на iterations.** Не >3 проходов по одному файлу.
3. **Idempotency check.** Если после уборки следующий запуск находит то же самое — выходим, отчёт идентичен → строка `## Note: idempotent — same findings as <date>. No new issues.`
4. **2 fails = стоп.** Если две попытки fix дают тот же finding — эскалация пользователю. Не долбить третий раз.

### 4. Cleanup-induced context pollution

Уборщик читает 200 файлов и захламляет окно — после него у основного агента нет места думать.

**Правила Риты:**
1. **Dedicated subagent.** Рита — отдельный агент с собственным окном.
2. **Возвращать только summary.** Не дамп прочитанного — только findings + counts.
3. **Phase 1 inventory bash-only через метаданные.** Не читать содержимое файлов — только `wc -l`, `head -1`, `find`. Это укладывается в ≤30 сек и в небольшой context budget.
4. **Compact каждые N findings.** Если deep-режим — после M3 (Routing) и M5 (Knowledge) — checkpoint и compact если >40% контекста.

### 5. False positives на специфике

Generic-аудитор не знает специфики офиса. Например, `failures.md` намеренно append-only, и большой размер — это feature.

**Правила Риты:**
1. **Exemptions через frontmatter.** Файл может объявить `audit-rules: skip-size-check`, `audit-rules: load-bearing` — и Рита пропускает соответствующие чеки.
2. **Project-specific config.** `.skill-policy.yml` (Skills Check) или `office/_audit-config.md` — переопределяют глобальные правила.
3. **Tech-debt-skill правило:** «Don't pattern-match to generic best practices without grounding in this specific repo.»
4. **`overrides.md`** Риты — пользователь пишет «не флагить failures.md size», и Рита уважает.

### 6. Privacy violations (бонус — критично!)

Уборщик читает все файлы и попадается на секреты в логах.

**Правила Риты:**
1. **Skip patterns.** Не читать `.env*`, `*.pem`, `credentials*`, `secrets*` — уборщику они не нужны.
2. **Redact в отчёте.** Если нашёл что-то похожее на токен — в отчёте `[REDACTED-TOKEN-LIKE-STRING]`, не сам токен.
3. **Settings.json deny.** `Edit(.env*)`, `Read(.env*)` — deny rule.
4. **Hardcoded secrets — P0.** Если в SKILL.md / core.md / config встретился `sk-`, `ghp_`, `xox[baprs]-`, `AKIA` — критичный finding, рекомендация: ротировать токен + переместить в `.env`.

---

## Чек-лист защит — самопроверка перед каждым действием

10 пунктов, нумерованных:

1. **Append-only?** Меняю файл — оставляю старую версию в `_archive/<YYYY-MM-DD>/`?
2. **Citation?** У каждого finding есть `file:line`?
3. **«Looks bad but is actually fine»?** Не выкошу ли load-bearing pattern?
4. **Approval?** Получил явное `да / только важное / "1,3" / нет`?
5. **Whitelist tools?** Опасный bash прошёл через guard? `Bash(rm:*)`, `Bash(mv:*)` запрещены вообще.
6. **Time cap?** Не превысил 5/15/60 мин в зависимости от режима?
7. **Iteration cap?** Не >3 проходов по одному файлу?
8. **Privacy?** Не пишу в отчёт `.env` содержимое, токены — `[REDACTED]`?
9. **Memory update?** После работы записал в `memory.md` / `failures.md`?
10. **Idempotent?** Если запустить ещё раз — не предложу те же изменения снова?

Если хоть один пункт NO → STOP, не действуй, эскалируй пользователю.

---

## Что делать когда over-cleaned (попал)

Если предложил снести что-то load-bearing и пользователь сказал «это нужно»:

1. **Сразу остановись.** Не аргументируй.
2. **Запиши в `failures.md`** по формату:
   ```
   ### YYYY-MM-DD — over-clean attempt: <название pattern'а>

   Что предположил: <pattern X — кандидат на удаление, кажется дубль>
   Что оказалось: <pattern X — load-bearing, нужен для Y>
   Правило на будущее: <такой pattern никогда не флагить, добавлять в "Looks bad but is actually fine">
   ```
3. **Не предлагай в следующих аудитах.** Architect грэп `failures.md` перед каждым прогоном.
4. **Если третий раз вижу одно и то же** → скажи прямо: *«Третий раз вижу одно и то же. Либо ты не одобряешь — это твоё право. Либо мы в разном понимании что чинить. Расскажи?»*
