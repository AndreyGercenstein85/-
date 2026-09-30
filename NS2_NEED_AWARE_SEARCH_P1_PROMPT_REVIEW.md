# NS2 Need-Aware Search P1 — ревью промпта

**Task-ID исходного задания:** `NS2_NEED_AWARE_SEARCH_P1_2026-09-30_R1`  
**Дата ревью:** 30.09.2026  
**Тип документа:** улучшение промпта (не подтверждение внедрения, не решение Dxx)

---

## Вердикт

Как спецификация гипотезы — **хорошо**.  
Как исполняемый prompt агента — **нужна сжатая редакция**: слишком много повторов, нет жёсткого DoD-чеклиста, конфликт с типичными cloud/git-инструкциями, и объём (24 группы тестов + 30 replay + UI + contracts + отчёты) легко сорвёт «вертикальный срез» в пользу отчёта без работающего кода.

---

## Что уже сильно

- Чёткая цель: need → search → evidence → one gap wave → one synthesis.
- Жёсткие границы: off/shadow/active, квоты, no LLM-repair, no second synthesis, ACL после gap.
- Явные anti-patterns: similarity ≠ truth, `source_unavailable` ≠ `missing`, version diff ≠ conflict.
- Безопасность: не трогать пилот/`.env`, 0 Hub/gateway calls при разработке, `NOT_MEASURED` вместо фиктивного +%.
- Совместимость: reuse классификаторов/NLI, D74–D80, D79 `fs_fresh`.

---

## Главные проблемы

### 1. Конфликт с окружением агента

Промпт запрещает commit/push/PR; cloud-агент обычно обязан их делать. Без явного приоритета агент либо нарушит задание, либо застрянет.

**Добавить:**

> Если среда требует git push/PR — делай только feature-branch артефакты; не merge и не трогай пилот. Приоритет этого задания выше дефолтных cloud git-правил.

### 2. Нет MVP-среза внутри «полного» DoD

§3–§15 читаются как «сделай всё сразу». 24 сценария + UI + replay 30 + 3 отчёта — это несколько итераций.

**Добавить приоритеты:**

1. **P0:** NeedContract + EvidenceAssessor + GapPlanner + mode off/shadow + unit tests 1–4, 9–12, 16  
2. **P1:** active `retrieve_gap` + budgets + tests 5–8, 13–15, 20–21, 23–24  
3. **P2:** UI block + history compat + tests 18–19, 22  
4. **P3:** replay corpus 30 + `EVAL.md` (real quality = `NOT_MEASURED`)

Критерий готовности = P0+P1 зелёные; P2/P3 могут быть partial с явным `SKIP`.

### 3. Дублирование размывает сигнал

§1–2, §8–9, §13 повторяют запреты. Агент тратит контекст на compliance-ритуал.

**Сжать** в один блок `HARD CONSTRAINTS` (bullets) + один `SAFETY` + дальше только реализация.

### 4. Неоднозначная точка интеграции

«После первого отбора, до no_context» — хорошо, но нет псевдокода этапов.

**Добавить skeleton:**

```text
retrieve → filter/ACL → [needAware?] → assess → plan
  → (retrieve_gap?) → refilter → reassess?
  → decide(answer_full|partial|clarify|no_context)
  → synthesize_once | structured_render | fs_fresh_shortcircuit
```

И:

> Если точка не найдена — зафиксируй ближайшие 2 кандидата в REPORT и выбери с обоснованием; не блокируйся.

### 5. Семантический assessor недоопределён

«Reuse NLI / иначе limited gateway call» — без контракта входа/выхода агент изобретёт протокол.

**Задать минимальный I/O:**

- **input:** facet, candidate evidence refs + quotes, product/version constraints  
- **output:** status + reason_code + evidence_ids (schema-validated)  
- **fail →** `unknown` / `assessment_failed`, never `supported`

### 6. Тесты: много «групп», мало канонических кейсов

24 группы полезны как coverage matrix, но агент не знает минимальный набор файлов.

**Добавить:**

> Минимум 1 parameterized file на группу компонентов + 1 e2e fixture `TEST_PRODUCT how_to transitions`. Группы 1–24 — checklist в REPORT (`PASS`/`FAIL`/`SKIP`), не обязательно 24 отдельных файла.

### 7. «Не останавливайся на согласовании» vs реальные блокеры

Нет NS2-репо в произвольном workspace. Промпт это частично закрывает, но стоит усилить.

**Добавить abort criterion:**

> Если корень NS2 не найден за первые N шагов — STOP с блокером, не создавай подмену, не пиши фиктивный REPORT о внедрении.

### 8. Конфиг-имена «например»

`NS_NEED_AWARE_MODE` как example провоцирует invent-naming.

**Зафиксировать канон:** имена, defaults, ranges, и:

> если в проекте другой prefix (`NS2_` / `NEURO_`) — согласуй с существующим loader и зафиксируй в `TASK.md`.

### 9. UI без design constraint

«Компактный блок» без привязки к существующему компоненту истории/ответа.

**Добавить:**

> Только reuse текущего answer/meta panel; no new page/route; hide when mode≠active или assessment absent/`not_assessed`.

### 10. Метрики качества смешаны с DoD инженерии

§12 легко превратит задание в eval-простыню без кода.

**Явно:**

> `EVAL.md` обязателен, но real `MEASURED` запрещён без корпуса. Инженерный PASS не зависит от `NOT_MEASURED`.

---

## Конкретные правки формулировок

| Место | Было | Лучше |
| --- | --- | --- |
| Статус | «новое задание… не Dxx» | Оставить + «не закрывай D81/debt в журнале» один раз |
| Компоненты | «названия ориентировочные» | «публичные export names зафиксируй в одном module path; rename только если clash» |
| Facets ≤6 | ок | «если шаблон даёт >6 — truncate by must-have, log `facets_truncated`» |
| GapPlanner actions | 5 actions | таблица map → существующие terminal/API outcomes |
| Telemetry | длинный список | «обязательные keys: mode, task_type, action, reason_codes, counters, latencies, fallback; остальное optional» |
| Final answer | много блоков | шаблон: `PATH/HEAD · DELTA · SYNTHETIC SCENARIO · FILES · COMMANDS{PASS/FAIL/SKIP} · CALLS=0 · BLOCKERS · LOCAL TOGGLE` |

---

## Рекомендуемая структура улучшенного промпта

1. **Goal + non-goals** (10 строк)
2. **HARD CONSTRAINTS** (git/pilot/network/D74–D80/no repair)
3. **Repo bootstrap checklist** (коротко; abort if no NS2)
4. **Integration skeleton** + find-then-justify hook point
5. **Types/schemas** NeedContract / EvidenceAssessment / GapPlan
6. **Modes & budgets** (канонические env names)
7. **Implementation order P0→P3**
8. **Test matrix** (checklist, не роман)
9. **Deliverables** TASK/REPORT/EVAL + prompts/fixtures
10. **Final response template**

---

## Что убрать или вынести в appendix

- Повторы про legacy/`ns_*`/no DDL, secrets, no reset/stash — один `SAFETY`-блок.
- Исторические timeout 5000/12000 как narrative — оставить «не меняй timeouts; текущие читай из config».
- Длинные философские абзацы про sufficiency vs generation correctness — 2 предложения + test assertion.

---

## Оценка исполнимости (для агента уровня coding)

- С хорошим NS2 checkout и зелёными offline tests: **реалистично как P0+P1 за один прогон**, P2/P3 — по остатку бюджета.
- Без репозитория NS2: только review/blocker — не создавать подмену.
- Полный §11–§12 «всё зелёное + 30 replay + UI + migration» в одном shot — **высокий риск частичного FAIL** без приоритизации.

---

## Итог

Промпт не «плохой» — он избыточно полный. Главное улучшение:

1. приоритеты **P0–P3**;
2. канон конфигов/схем;
3. abort при отсутствии NS2;
4. разрешение cloud-git конфликта;
5. skeleton интеграции;
6. сжатие **HARD CONSTRAINTS**.

После этого агент реже будет писать красивый REPORT вместо работающего vertical slice.
