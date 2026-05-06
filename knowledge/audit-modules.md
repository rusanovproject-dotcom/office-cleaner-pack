# Audit modules — карта 6 архетипов уборщиков + Florian-rubric

**Зачем здесь:** общий обзор модулей с детализацией по реализации. Используется агентом как routing-карта в Phase 2 Audit.

**Структура:** **6 архетипов уборщиков** (M1-M6, реальные ревизии) + **8-мерная Florian-rubric** (финальная сборка score 0-100 в deep-режиме). Архетипы — это что Рита находит. Rubric — как складывается итоговая оценка по dimensions.

> Историческая справка: ранее этот файл назывался `7-modules.md` (6 архетипов + rubric = «7 модулей по сути»). Переименован в `audit-modules.md` чтобы устранить терминологическую путаницу — rubric не равноправный «модуль уборщика», а отдельная финальная сборка результатов.

---

## Архетипы и rubric — обзор

| # | Модуль | Тип | Когда запускать | Output |
|---|--------|------|-----------------|--------|
| 1 | Memory Cleaner | архетип уборщика (AutoDream) | Все режимы. В scan — surface, в tidy/deep — детально | M1.X findings |
| 2 | Skill Auditor | архетип уборщика (Skills Check + Florian D3-D4) | Все режимы | M2.X findings |
| 3 | Routing Validator | архетип уборщика (ln-310-multi-agent-validator) | Все режимы. Особенно после `/install-agent` | M3.X findings |
| 4 | Structure Linter | архетип уборщика (Repository Cleanup + Florian D2) | Все режимы | M4.X findings |
| 5 | Knowledge Dedup | архетип уборщика (ln-630-test-auditor pattern) | Только в deep | M5.X findings |
| 6 | Failure Pattern Detector | архетип уборщика (Arize + MindStudio) | Только в deep | M6.X findings |
| — | Florian-rubric scoring | финальная сборка (не архетип) | После всех 6 архетипов, только в deep | Score 0-100 + tier |

---

## Module 1 — Memory Cleaner

**Что чистит:** redundancy / contradictions / stale timestamps / outdated debugging notes (см. `auto-dream-rules.md`).

**Лимиты для клиентского профиля:**
- `memory.md` агента: soft 500 строк, hard 1000 строк
- `failures.md`: без лимита (append-only feature)
- `.claude/MEMORY.md` (если есть): ≤200 строк

**Реализация:**
```bash
# Размеры
find office/agents -maxdepth 3 -name "memory.md" -exec wc -l {} \;
find office/agents -maxdepth 3 -name "failures.md" -exec wc -l {} \;
test -f .claude/MEMORY.md && wc -l .claude/MEMORY.md

# Stale timestamps
grep -rn -E '(yesterday|вчера|последнее время|недавно|пару дней|на прошлой неделе)' \
  office/agents -name "memory.md"

# Outdated references — для каждого упомянутого пути проверить существование
grep -rn -oE '(office/agents/[a-z-]+/[a-z-]+\.md)' office/agents | \
  while read line; do
    # parse path, test -f
  done
```

**Detail см. `auto-dream-rules.md`.**

---

## Module 2 — Skill Auditor

**Что проверяет:**
- Frontmatter корректный (`name:`, `description:` с TRIGGERS, `allowed-tools:` если нужно)
- Размеры: SKILL.md ≤500 строк, core.md ≤200 строк
- Hardcoded secrets (security)
- Frontmatter `updated:` свежий (если есть)

**Реализация:**
```bash
# Frontmatter check (наличие YAML перед --- и ключей)
for f in $(find .claude/skills office/agents -name "SKILL.md" -o -name "core.md"); do
  head -20 "$f" | grep -q "^---$" || echo "MISSING_FRONTMATTER: $f"
  head -20 "$f" | grep -q "^name:" || echo "NO_NAME: $f"
  head -20 "$f" | grep -q "^description:" || echo "NO_DESCRIPTION: $f"
done

# Размеры
find .claude/skills -name "SKILL.md" -exec wc -l {} \;
find office/agents -maxdepth 2 -name "core.md" -exec wc -l {} \;

# Hardcoded secrets
grep -rln -E '(sk-[a-zA-Z0-9]{20,}|ghp_[a-zA-Z0-9]{20,}|xox[baprs]-|AKIA[0-9A-Z]{16})' \
  --include="*.md" --include="*.json" .
```

**Tier-1 finding (P0):** hardcoded secret. Сразу `[REDACTED]` в отчёте + рекомендация ротировать.

---

## Module 3 — Routing Validator

**Что проверяет:**
- Каждый агент в `office/agents/*` упомянут в `office/AGENTS.md`
- Каждая запись в `office/AGENTS.md` имеет физический агент в `office/agents/*`
- Триггеры скиллов не пересекаются (parsing `_agent-packs/*/install.md` поля `trigger_keywords` + `.claude/skills/*/SKILL.md` поле `description:` секция TRIGGERS)
- В корневом `CLAUDE.md` есть `@office/agents/<name>/core.md` для каждого установленного агента
- Ссылки `/<skill-name>` ведут на существующие скиллы

**Реализация:**
```bash
# Список физических агентов
find office/agents -maxdepth 1 -type d | sed 's|office/agents/||' | grep -v '^$' | sort > /tmp/physical-agents.txt

# Список агентов из AGENTS.md (parsing markdown table)
grep -oE '\*\*([A-ZА-Я][a-zа-я-]+)\*\*' office/AGENTS.md | sort -u > /tmp/agents-md.txt

# Diff
diff /tmp/physical-agents.txt /tmp/agents-md.txt

# @include в корневом CLAUDE.md
grep -oE '@office/agents/[a-z-]+/core\.md' CLAUDE.md | sort -u

# Trigger collision (если jq доступен и install.md формат yaml)
for f in _agent-packs/*/install.md; do
  # Parse trigger_keywords
done
```

**Tier-1 finding (P0):** orphan agent — физически есть, в AGENTS.md нет. Или наоборот.

---

## Module 4 — Structure Linter

**Что проверяет:** PARA-структура, INDEX.md в папках с ≥3 файлами, symlinks, settings.json/.mcp.json/hooks/, плейсхолдеры, корректный client-profile.md.

**Реализация:**
```bash
# Папки с ≥3 файлами без INDEX.md
for dir in $(find . -type d -not -path './.git/*' -not -path './_archive/*'); do
  count=$(find "$dir" -maxdepth 1 -type f -name "*.md" | wc -l)
  if [ "$count" -ge 3 ] && [ ! -f "$dir/INDEX.md" ]; then
    echo "MISSING_INDEX: $dir ($count files)"
  fi
done

# Symlinks (защита от false positives)
find . -type l -not -path './.git/*' 2>/dev/null

# Config & hooks
test -f .claude/settings.json || echo "MISSING: .claude/settings.json"
test -f .claude/.mcp.json || echo "MISSING: .claude/.mcp.json"
test -d .claude/hooks || echo "MISSING: .claude/hooks/"

# Плейсхолдеры
grep -rln '{{[A-Z_][A-Z_0-9]*}}' . --include="*.md"

# client-profile.md правильный формат?
grep -lE '(LEAD_SOURCE|Бюджет|Возражения|Текущая сделка)' office/client-profile.md
# Если найдено — это B2B-формат, но client-profile.md должен быть профилем владельца
```

---

## Module 5 — Knowledge Dedup (только в deep)

**Что проверяет:** дубли в `knowledge/` через Jaccard на heading sets (без эмбеддингов — дешевле).

**Реализация Jaccard:**
```bash
# 1. Извлечь H1/H2 заголовки из каждого .md в knowledge/
extract_headings() {
  local f="$1"
  grep -E '^#{1,2} ' "$f" | sed 's/^#* //' | sort -u
}

# 2. Для каждой пары посчитать Jaccard
jaccard() {
  local a="$1"
  local b="$2"
  local intersect=$(comm -12 <(echo "$a") <(echo "$b") | wc -l)
  local union=$(echo "$a"$'\n'"$b" | sort -u | wc -l)
  if [ "$union" -eq 0 ]; then echo "0"; return; fi
  echo "scale=2; $intersect / $union" | bc
}

# 3. Если ≥0.7 — кандидат на дубль, если =1.0 — точный heading collision
files=$(find knowledge -name "*.md" -type f)
for f1 in $files; do
  for f2 in $files; do
    if [ "$f1" \< "$f2" ]; then
      h1=$(extract_headings "$f1")
      h2=$(extract_headings "$f2")
      score=$(jaccard "$h1" "$h2")
      if (( $(echo "$score >= 0.7" | bc -l) )); then
        echo "DUP_CANDIDATE: $f1 vs $f2 (Jaccard=$score)"
      fi
    fi
  done
done
```

**Reality check:** для офиса с 30+ knowledge-файлов это N² пар. Имеет смысл ограничивать только парами в одной поддиректории или одного префикса в имени.

---

## Module 6 — Failure Pattern Detector (только в deep)

**Что делает:** см. `failure-archetypes.md`. Группирует записи `failures.md` за 30 дней по 8 archetypes, ищет повторы.

**Пороги:**
- Офис ≤5 агентов: 3+ повтора одного правила за 30 дней → finding
- Офис >10 агентов: 5+ повтора (паттерны размазаны)
- Cross-agent pattern: 3+ агентов имеют записи в одном archetype → правило в корневой CLAUDE.md / protocols/

**Реализация:**
```bash
# Сбор записей за 30 дней
for f in office/agents/*/failures.md; do
  # Parse YYYY-MM-DD заголовки, фильтр >= today-30d
  awk '/^### [0-9]{4}-[0-9]{2}-[0-9]{2}/ {date=$2}
       date >= "'"$(date -v-30d +%Y-%m-%d)"'" {print FILENAME":"NR":"$0}' "$f"
done | tee /tmp/recent-failures.txt

# Группировка по keyword overlap с 8 archetypes
# (упрощённо — по ключевым словам)
```

---

## Florian-rubric scoring (только в deep)

**Это не седьмой архетип — это финальная сборка** результатов работы 6 архетипов в единый score 0-100 по 8 dimensions.

**См. `florian-rubric.md`** для детальной 8-мерной таблицы с весами.

**Process:**
1. После всех 6 архетипов собрать findings counts (P0, P1, P2 по dimensions)
2. Применить веса по 8 dimensions (Memory & Context 20, Memory Hygiene 15, ...)
3. Каждый dimension — 0-max баллов с обоснованием через cited findings
4. Total = сумма
5. Tier по total: Starter (<40), Growing (40-59), Established (60-79), Optimized (80+)

---

## Зависимости между модулями

```
Phase 1 Inventory (всегда первый)
    ↓
    ├── M4 Structure Linter (зависит только от Inventory)
    ├── M2 Skill Auditor (зависит только от Inventory)
    ├── M3 Routing Validator (зависит от M2 + M4 — нужны frontmatter и структура)
    ├── M1 Memory Cleaner (зависит только от Inventory)
    ├── M5 Knowledge Dedup (зависит от M4 — нужно знать структуру knowledge/)
    └── M6 Failure Pattern Detector (независимо)
        ↓
    Florian-rubric scoring (зависит от ВСЕХ — собирает counts; не архетип, а финальная сборка)
        ↓
    Phase 3 Unified Report
        ↓
    Phase 4 Validation Request (только tidy/deep)
        ↓
    Phase 5 Memory Update
```

В scan-режиме архетипы M5, M6 и Florian-rubric пропускаются. В tidy — пропускаются M5 и Florian-rubric.
