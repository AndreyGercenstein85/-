# NS2 Need-Aware Search P1 — ревью R2.1

**Task-ID:** `NS2_NEED_AWARE_SEARCH_P1_2026-09-30_R2.1`  
**Дата ревью:** 01.10.2026  
**База:** `NS2_NEED_AWARE_SEARCH_P1_TASK.md`  
**Предыдущее ревью R1:** `NS2_NEED_AWARE_SEARCH_P1_PROMPT_REVIEW.md`

---

## Вердикт

R2.1 **закрывает почти все критические дыры R1** как исполняемый бриф: фазы P0–P3, HC, STOP, канон конфигов, assessment I/O, shadow lifecycle, clarify→P2, engineering ready ≠ pilot ready.

Осталось **3–4 операционных риска** перед выдачей агенту (не блокеры смысла спецификации). Документ ещё нельзя считать «готовым к запуску без владельца»: launch fields пустые → агент обязан STOP.

---

## Закрытие замечаний к R1

| # | Замечание к R1 | Статус в R2.1 | Комментарий |
| --- | --- | --- | --- |
| 1 | Конфликт cloud-git vs запрет commit/push | **Закрыто решением владельца** | Среда локальная; commit/merge/push — человек. Для cloud-агента остаётся внешний конфликт среды, но внутри задания приоритет ясен. |
| 2 | Нет MVP / фаз | **Закрыто** | P0–P3, engineering ready = P0+P1. |
| 3 | Дублирование запретов | **Частично** | HC-1…14 хороши; полное тело R1 всё ещё приложено → риск следовать отменённым §1/§6–8. |
| 4 | Нет skeleton интеграции | **Закрыто** | Канонический маршрут + 2 кандидата в REPORT. |
| 5 | Assessor I/O недоопределён | **Закрыто** | Input/output, validation, assess1/assess2, no repair. |
| 6 | 24 теста без привязки к файлам | **Закрыто** | Checklist + PASS только со ссылкой на test; фазы назначены. |
| 7 | Abort без NS2 | **Закрыто** | Preflight STOP / launch fields. |
| 8 | Config «например» | **Закрыто** | Канон `NS_NEED_AWARE_*` + `SHADOW_SAMPLE_RATE`. |
| 9 | UI без constraint | **Закрыто** | Только existing panel; блок только active+assessed. |
| 10 | EVAL смешан с DoD | **Закрыто** | Engineering PASS не зависит от NOT_MEASURED. |

---

## Что в R2.1 сделано особенно удачно

- **Clarify вынесен в P2** — убирает главный объём/ambiguity trap из P1.
- **Shadow после terminal как bounded best-effort** + запрет fire-and-forget + fallback «до synthesis» — реалистичнее старого «измерить latency, но не менять UX».
- **Gap reasons разделены** (`assessment_invalid` / `assessment_unavailable` / `coverage_unknown`…) и явно запрещают бесполезный gap retrieval.
- **HC-10/11** (исходные gates vs gap evidence; packing/rebind) закрывают типичный путь «допоиск открыл docs channel / supported без final context».
- **D81/№44 → `coverage_unknown`** без catalog monitor — правильный scope cut.
- **Test 18 SKIP N/A** не валит P0/P1, но остаётся блокером пилота.

---

## Оставшиеся риски / правки перед выдачей

### 1. Launch fields пустые → STOP обязателен

Поля `<ПУТЬ>`, `<ВЕТКА>`, `<SHA>`, merge `answer/docs-lost` не заполнены.  
**До выдачи агенту владелец обязан заполнить**, иначе корректный исход — только STOP, не реализация.

### 2. Полное тело R1 всё ещё создаёт шум

Несмотря на таблицу отмен, агент может снова прочитать §1 («изолируй непересекающуюся часть»), §6 (`clarify` в действиях GapPlanner), §8 (shadow latency по-старому).

**Рекомендация:** в выдаче агенту либо (a) только R2.1 + «R1 appendix по ссылке», либо (b) в теле R1 зачеркнуть/пометить `REVOKED` у отменённых абзацев сильнее, чем одной строкой-примечанием.

### 3. Shadow timing vs test 2

R2.1 предпочитает shadow **после** terminal event → пользовательская latency пути не должна расти.  
R1 test 2 всё ещё говорит «доп. задержка измеряется отдельно».

**Уточнить одной фразой в test 2:**  
- post-terminal shadow: user-path latency ≈ baseline; diagnostic latency — отдельный counter;  
- pre-synthesis fallback: diagnostic latency входит в request duration и обязательно в REPORT как отклонение.

### 4. Budget formula неполная в одном месте

«Remaining budget = Hub timeout × calls + assessment timeout» — верхняя оценка стоимости волны, но не сказано явно:  
`start_only_if cost_upper_bound ≤ remaining(NS_NEED_AWARE_EXTRA_BUDGET_MS) ∧ remaining(request deadline)`.

Смысл угадывается из других абзацев; лучше склеить в одно условие.

### 5. Мелочи схемы

- `facets_template_only` упомянут в E2E paraphrases, но не в assessment state / contract fields — добавить в схему или переименовать в уже существующий flag.
- GapPlanner action `clarify` в R1 §6 при P1 user-path = partial: ок как internal, но в P1 лучше писать `answer_partial(+ambiguity)` чтобы не плодить мнимый user event.
- `NS_NEED_AWARE_ASSESSMENT_TIMEOUT_MS=20000` помечен как непроверенное допущение — хорошо; оставить в REPORT как assumption, не SLA.

### 6. Дублирование shadow fallback

Есть два почти одинаковых абзаца: «если scope уничтожается…» и «если bounded post-terminal невозможен…». Свести к одному правилу приоритета:

1. post-terminal bounded registered task;  
2. иначе pre-synthesis (не между answer и terminal);  
3. иначе `skipped_*` / `not_assessed`, без user impact.

---

## Готовность к выдаче агенту

| Критерий | Статус |
| --- | --- |
| Спецификация смысла / границ | Готова |
| Исполняемость на заполненном NS2 tree | Готова после fill launch fields |
| Сжатость prompt | Средняя (HC хороши, R1 appendix тяжёлый) |
| Engineering DoD | Ясен (P0+P1) |
| Pilot enable | Явно запрещён этим заданием |

**Итог:** R2.1 — успешная коррекция R1. Перед запуском: заполнить launch fields и (желательно) убрать/заглушить revoked R1-абзацы, чтобы агент не исполнил отменённое.

Проверка кода NS2 / внедрение по этому заданию **не выполнялись** (в текущем workspace нет репозитория NeuroSearch).
