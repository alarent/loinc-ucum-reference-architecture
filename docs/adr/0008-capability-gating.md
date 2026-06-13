# ADR-0008 · Capability gating (nullable fields)
- **Статус:** Accepted
- **Уровень внедрения:** сквозной (механизм gating для L1–L4)
- **Дата:** 2026-06-01
- **Связанные ADR:** ADR-0005 (Version-aware mapping), ADR-0006 (API contract + rejection envelope), ADR-0007 (Method aliasing)
- **Связанные документы:** [Уровни зрелости внедрения (capability levels)](../capability-levels.md), `output.schema.json`
## Контекст
Reference-архитектура описана как полный контракт v1.0 (Level 4 в терминах capability levels). Модель поэтапного внедрения вводит уровни L1–L4, где нижние уровни не реализуют часть возможностей (например, method aliasing из ADR-0007 появляется только на L4).
Возникает вопрос: как контракт представляет «срез» возможностей на нижнем уровне, не нарушая принцип additive-only (переход между уровнями только добавляет поля, никогда не ломает форму). Без явного решения nullable-поля в `output.schema.json` (`method_alias_group`, `ucum_conversion`, `remap_source` и др.) выглядят как простая осторожность, а не как намеренный механизм.
## Решение
Schema всегда полная (v1.0). Поля, которые ещё не активированы на текущем уровне внедрения, остаются nullable — и эта nullability фиксируется как **намеренный capability gating**, а не как недоопределённость.
### 1. Единая полная schema для всех уровней
- Нет отдельных schema под каждый уровень и нет веток contract'а. Все уровни валидируются против одного `output.schema.json`.
- Поднятие уровня = заполнение ранее null-овых полей, а не изменение формы. Это прямое следствие additive-only.
### 2. Capability-gated поля привязаны к уровню
- Поля, которые отражают возможность выше текущего уровня, до включения этого уровня не несут значения: version-aware блок и `remap_source` (L2), context-поля (L3), `method_alias_group` (L4). Полная карта «поле → уровень» — в разделе «Закрытые вопросы».
- Это отличается от «значение неприменимо к этому маппингу» (например, `rule_applied = null`, когда правило не сработало). Разведение двух смыслов — см. «Закрытые вопросы».
### 3. Объявление уровня — в конфиге и эхом в audit
- Источник истины об уровне — конфиг развёртывания (объявляется один раз, не пересчитывается на запрос).
- Этот уровень эхом кладётся в каждый ответ как `audit.capability_level` (enum `L1`–`L4`), чтобы ответ был самодостаточен для аудита и нормативной проверки. Optional в v1.0, становится фактически required при поддержке нескольких уровней.
- Runtime-обнаружение возможностей (`GET /v1/capabilities`) — кандидат Phase 2, не часть v1.0.
## Альтернативы
### Breaking change при повышении уровня (разные schema на уровень)
- **Отвергнуто:** прямо нарушает additive-only. Потребителям пришлось бы переписывать интеграцию на каждом переходе уровня — поэтапное внедрение становится фикцией.
### Runtime capability discovery как обязательная часть v1.0
- **Отложено:** честный подход, но добавляет сложность контракта раньше, чем она нужна. Перенесён в Phase 2 как `GET /v1/capabilities`.
### Оставить nullable-поля без явного решения
- **Отвергнуто:** именно текущее состояние. Без фиксации через год возникает вопрос «почему `method_alias_group` в schema, если мы не на L4», и ответить нечем.
## Последствия
### Положительные
- Совместимо с текущим `output.schema.json` для реально nullable полей (`ucum_conversion`, `remap_source`, `replaced_by`, `rule_applied`); для strict-полей (`loinc_version`, `method_alias_group` и т. п.) gating выражается conditional-валидацией (см. «Закрытые вопросы»).
- Поэтапное внедрение работает без форков контракта и версий schema.
- Ретроактивно объясняет nullability-решения, принятые в ADR-0005/0006/0007.
### Отрицательные / trade-offs
- По одной schema нельзя понять, на каком уровне работает слой — нужен out-of-band источник (документация или будущий `/v1/capabilities`).
- null перегружен двумя смыслами: «неприменимо для этого маппинга» и «возможность не включена на этом уровне». Разведено через `audit.capability_level` + карту «поле → уровень» (см. «Закрытые вопросы»).
## Связь с другими ADR
- **ADR-0005/0006/0007:** именно их nullable-поля этот ADR переосмысляет как capability-gated, а не просто опциональные.
- **Страница capability levels:** этот ADR — схемный фундамент для модели уровней; сама модель остаётся отдельным документом.
## Закрытые вопросы
Оба вопроса закрыты 2026-06-01.
### 1. Разведение двух смыслов null — решено
Смысл `null` определяется не кодировкой поля, а сравнением уровня поля (карта ниже) с активным `audit.capability_level`:
- уровень поля ≤ активного → возможность включена, `null` = «неприменимо к этому маппингу» (semantic null);
- уровень поля \> активного → `null`/отсутствие значения = «возможность не включена на этом уровне» (gating null).
Поля вне карты (`audit.ucum_conversion`, `audit.rule_applied` и др.) всегда трактуются как semantic null. `audit.capability_level` (optional, enum `L1`–`L4`) добавлен в `output.schema.json`.
### 2. Карта «поле → уровень» — решено
<table fit-page-width="true" header-row="true">
<tr>
<td>**Уровень**</td>
<td>**Поля, активируемые на уровне (до него — gating null)**</td>
</tr>
<tr>
<td>**L2 — version-aware + priority**</td>
<td>`audit`: `loinc_version`, `ucum_version`, `current_version_status`, `replaced_by`, `remap_source`, `policy_version`, `rule_applied`, `rule_overrode_score`, `top_score_by_resolver`, `score_delta_vs_top`</td>
</tr>
<tr>
<td>**L3 — semantic context + qualitative**</td>
<td>`context`: `sample_type`, `fasting_status`, `method`, `context_completeness`; qualitative-ветка `value.string`</td>
</tr>
<tr>
<td>**L4 — method aliasing + полный контракт**</td>
<td>`audit.method_alias_group`</td>
</tr>
</table>
Базовый контракт L1 (присутствует всегда): `primary`, `value` (численная ветка), `alternatives[]`, `audit.primary_score`, `candidates_returned`, `resolver_version`, `mapping_timestamp`, `capability_level`.
### Механизм enforcement (важно)
Полный strict-`output.schema.json` — это контракт L4: ряд gated-полей там required-non-null (`loinc_version`, `current_version_status`, `method_alias_group` и т. п.), то есть «просто nullable» их не гейтит. Чтобы gating был реальным при валидации, нижние уровни используют ту же schema с per-level ослаблением (gated-поля становятся nullable/optional), выраженным через conditional `if`/`then` по `audit.capability_level`. В v1.0 (артефакт без реализации, см. README) фиксируем механизм и карту; полное conditional-кодирование в schema — задача Phase 2 при появлении многоуровневой реализации.
## Changelog
- 2026-06-01 — создан ADR-0008 (capability gating) при формализации модели capability levels
- 2026-06-01 — закрыты оба открытых вопроса: разведение semantic-null vs gating-null через `audit.capability_level`; карта «поле → уровень»; добавлен `audit.capability_level` в output.schema; зафиксирован механизм enforcement (conditional `if`/`then`, encoding в Phase 2)
