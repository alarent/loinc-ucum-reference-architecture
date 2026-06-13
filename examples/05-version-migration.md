# Worked example 05 · Version migration
**Уровень внедрения:** L2
## Назначение примера
Валидирует ADR-0005 (Version-aware mapping). Показывает полный цикл snapshot-versioned mapping + explicit remap workflow на кейсе `REPLACED_BY` между релизами LOINC. Коды plausible (структурно реальны), конкретная deprecation — иллюстративная.
## Сценарий
В 2024-03-15 лаборатория измерила у пациента cortisol в serum, утренний забор (AM context), fasting, метод — ECLIA. Результат: 18 µg/dL.
Mapping выполнен в момент receipt лабрезультата (2024-03-15) при актуальной LOINC версии 2.76 и UCUM 2.1.
## Input (2024-03-15)
```json
{
	"lab": "lab-orchestrator-v3",
	"date": "2024-03-15",
	"type": "сыворотка крови, утро натощак",
	"biomarkers": [
		{
			"raw_name": "Cortisol",
			"raw_value": "18",
			"raw_unit": "µg/dL",
			"raw_ref": "6.2–19.4",
			"raw_comment": "Метод: ECLIA; забор 08:15"
		}
	]
}
```
## Initial mapping output (2024-03-15, LOINC 2.76)
```json
{
	"status": "success",
	"mapping_id": "map_2024_03_15_abc123",
	"primary": {
		"system": "LOINC",
		"code": "2143-6",
		"display": "Cortisol [Mass/volume] in Serum or Plasma"
	},
	"alternatives": [
		{
			"system": "LOINC",
			"code": "15083-9",
			"display": "Cortisol [Mass/volume] in Serum or Plasma --8 AM specimen",
			"score": 0.74
		}
	],
	"value": {
		"numeric": 18,
		"string": null,
		"unit": "µg/dL",
		"ucum_canonical": "ug/dL"
	},
	"context": {
		"sample_type": "serum",
		"fasting_status": "fasting",
		"method": "ECLIA",
		"context_completeness": "full"
	},
	"audit": {
		"loinc_version": "2.76",
		"ucum_version": "2.1",
		"mapping_timestamp": "2024-03-15T08:42:11Z",
		"primary_score": 0.88,
		"top_score_by_resolver": 0.88,
		"score_delta_vs_top": 0,
		"rule_applied": null,
		"rule_overrode_score": false,
		"method_alias_group": "loinc:component:c0a8f1e2",
		"ucum_conversion": {
			"input_unit": "µg/dL",
			"canonical_unit": "ug/dL",
			"factor": 1,
			"path": ["identity"]
		},
		"remap_source": null,
		"current_version_status": "current",
		"replaced_by": null,
		"candidates_returned": 2,
		"policy_version": "1.0.0",
		"resolver_version": "2024.03"
	}
}
```
Комментарии:
- Primary = `2143-6` (generic serum/plasma cortisol) по score, без rule override.
- AM-specific код `15083-9` в alternatives — score ниже, rule «AM context favors AM-specific code» не сработало (not in v1.0 priority policy).
- `current_version_status: "current"` в момент mapping (это же latest версия).
## Релиз LOINC 2.82 (начало 2026, иллюстративный сценарий)
Допустим, в LOINC 2.82 код `2143-6` получает статус *deprecated* с релацией `REPLACED_BY = 91056-5` («Cortisol \[Mass/volume\] in Serum or Plasma --specimen» — новая каноническая форма с явным specimen-qualifier).
Reference layer обновляет внутренние таблицы: `reference_layer.loinc_version = "2.82"`. **Существующий mapping ****`map_2024_03_15_abc123`**** не трогается — он immutable.**
## Retrieval после релиза (2026-06-01, LOINC 2.82 active)
Потребитель запрашивает `GET /mapping/map_2024_03_15_abc123`. Reference layer возвращает сохранённый result, **но вычисляет ****`current_version_status`**** в момент retrieval**:
```json
{
	"status": "success",
	"mapping_id": "map_2024_03_15_abc123",
	"primary": {
		"system": "LOINC",
		"code": "2143-6",
		"display": "Cortisol [Mass/volume] in Serum or Plasma"
	},
	"alternatives": [
		{
			"system": "LOINC",
			"code": "15083-9",
			"display": "Cortisol [Mass/volume] in Serum or Plasma --8 AM specimen",
			"score": 0.74
		}
	],
	"value": {
		"numeric": 18,
		"string": null,
		"unit": "µg/dL",
		"ucum_canonical": "ug/dL"
	},
	"context": {
		"sample_type": "serum",
		"fasting_status": "fasting",
		"method": "ECLIA",
		"context_completeness": "full"
	},
	"audit": {
		"loinc_version": "2.76",
		"ucum_version": "2.1",
		"mapping_timestamp": "2024-03-15T08:42:11Z",
		"primary_score": 0.88,
		"top_score_by_resolver": 0.88,
		"score_delta_vs_top": 0,
		"rule_applied": null,
		"rule_overrode_score": false,
		"method_alias_group": "loinc:component:c0a8f1e2",
		"ucum_conversion": {
			"input_unit": "µg/dL",
			"canonical_unit": "ug/dL",
			"factor": 1,
			"path": ["identity"]
		},
		"remap_source": null,
		"current_version_status": "replaced",
		"replaced_by": "91056-5",
		"candidates_returned": 2,
		"policy_version": "1.0.0",
		"resolver_version": "2024.03"
	}
}
```
Ключевые наблюдения:
- `primary.loinc_code` всё ещё `2143-6` — *interpretation immutable*, lab value, маппированный в 2024-м, так и остаётся.
- `audit.loinc_version` = `2.76` — snapshot version, immutable.
- `audit.current_version_status` = `"replaced"` — вычислено dynamically, signal для потребителя.
- `audit.replaced_by` = `"91056-5"` — указывает LOINC код-преемник.
- Snapshot audit-поля (`top_score_by_resolver`, `score_delta_vs_top`, rule audit, `method_alias_group`, `ucum_conversion`, reproducibility-поля) возвращаются как были в момент mapping — immutable, отображаются в полном виде.
## Потребитель вызывает /remap
```javascript
POST /remap
{
	"mapping_id": "map_2024_03_15_abc123"
}
```
Reference layer выполняет новый mapping в текущей версии (LOINC 2.82), на том же input (восстановлен из audit исходного mapping). Результат:
```json
{
	"status": "success",
	"mapping_id": "map_2026_06_01_xyz789",
	"primary": {
		"system": "LOINC",
		"code": "91056-5",
		"display": "Cortisol [Mass/volume] in Serum or Plasma --specimen"
	},
	"alternatives": [
		{
			"system": "LOINC",
			"code": "15083-9",
			"display": "Cortisol [Mass/volume] in Serum or Plasma --8 AM specimen",
			"score": 0.74
		}
	],
	"value": {
		"numeric": 18,
		"string": null,
		"unit": "µg/dL",
		"ucum_canonical": "ug/dL"
	},
	"context": {
		"sample_type": "serum",
		"fasting_status": "fasting",
		"method": "ECLIA",
		"context_completeness": "full"
	},
	"audit": {
		"loinc_version": "2.82",
		"ucum_version": "2.2",
		"mapping_timestamp": "2026-06-01T11:03:47Z",
		"primary_score": 0.91,
		"top_score_by_resolver": 0.91,
		"score_delta_vs_top": 0,
		"rule_applied": null,
		"rule_overrode_score": false,
		"method_alias_group": "loinc:component:c0a8f1e2",
		"ucum_conversion": {
			"input_unit": "µg/dL",
			"canonical_unit": "ug/dL",
			"factor": 1,
			"path": ["identity"]
		},
		"remap_source": "map_2024_03_15_abc123",
		"current_version_status": "current",
		"replaced_by": null,
		"candidates_returned": 2,
		"policy_version": "1.0.0",
		"resolver_version": "2026.06"
	}
}
```
Ключевые наблюдения:
- Новый `mapping_id` — `map_2026_06_01_xyz789`. Исходный `map_2024_03_15_abc123` остаётся retrievable.
- `primary.loinc_code` теперь `91056-5` (преемник).
- `audit.loinc_version` = `2.82`, `mapping_timestamp` сегодняшний.
- Новое поле `audit.remap_source` — указывает на исходный mapping. Обеспечивает провенанс цепочки remap'ов.
- `current_version_status` = `"current"`, так как mapping выполнен на latest версии.
- Score вырос (0.88 → 0.91) — новый код более точно описывает этот specimen (ожидаемый эффект deprecation ради специфичности).
## Оба mapping_id остаются retrievable
Потребитель решает, какой mapping канонический для этой lab value:
- **Clinical record retrieval** («покажи мне результат как он был в 2024») → `map_2024_03_15_abc123`.
- **Current-state queries** («мои последние cortisol-результаты в latest LOINC») → `map_2026_06_01_xyz789`.
- **Audit trail / compliance** («что именно было в EHR на момент X») → исходный.
Никакого автоматического предпочтения между ними на стороне реф-слоя. Это в логике ADR-0001 А4: «референсный слой не принимает решений за потребителя».
## Что валидирует этот пример
1. **Immutability snapshot mapping**: исходный mapping_id возвращает тот же primary LOINC code и через 2 года после release deprecation этого кода.
2. **Dynamic visibility deprecation**: `current_version_status` и `replaced_by` дают потребителю сигнал без silent rewrite.
3. **Explicit remap workflow**: по инициативе потребителя, возвращает новый mapping_id с своей audit-fingerprint.
4. **Provenance chain**: `remap_source` связывает derived mapping с исходным, потребитель может восстановить историю.
5. **Консистентность с ADR-0001 А4**: reference layer не выбирает «canonical» mapping за потребителя.
6. **Консистентность с ADR-0003 audit**: все audit поля из ADR-0003 (loinc_version, primary_score, rule_applied, rule_overrode_score, top_score_by_resolver, score_delta_vs_top) сохраняются при всех retrieval'ах.
## Выявленные вопросы (закрыты 2026-05-29)
- **~~`remap_source`~~****\~\~ в audit**: это поле возникло в worked example, но не было явно зафиксировано в ADR-0005.\~\~ Закрыто в [ADR-0005 · Version-aware mapping](../docs/adr/0005-version-aware-mapping.md) (решение п.4 + audit-секция). Семантика: nullable, присутствует только в mapping'ах, созданных через `/remap`. Для фреш mapping = null.
- **\~\~Изменяемые vs immutable audit-поля**: `current_version_status` и `replaced_by` явно dynamic, всё остальное — immutable snapshot.\~\~ Закрыто в [ADR-0005 · Version-aware mapping](../docs/adr/0005-version-aware-mapping.md) (audit-секция «Snapshot vs computed»).
- **~~`/remap`~~****\~\~ идемпотентность**: dedup-ключ не был зафиксирован явно.\~\~ Закрыто в [ADR-0005 · Version-aware mapping](../docs/adr/0005-version-aware-mapping.md) (решение п.4): dedup-ключ `(source_mapping_id, loinc_version, ucum_version)`, повторный `/remap` возвращает существующий derived mapping.
## Открытые вопросы
- **Reconstruction input из audit**: example предполагает, что reference layer может восстановить исходный input из сохранённого mapping. Это требует хранить input snapshot вместе с mapping. Альтернатива — потребитель предоставляет input при вызове `/remap`. Решить в ADR-0006 (API contract).
- **Sort order в alternatives после remap**: в example `15083-9` остался в alternatives при remap. Это ожидаемо? Или alternatives переоцениваются в latest версии с нуля? Да, переоцениваются — это new mapping в новой версии, просто в этом primer'е результат совпал.
- **AM context в latest версии**: в LOINC 2.82 возможно `15083-9` (AM-specific) стал бы primary при наличии rule «AM время забора предпочитает AM-specific code». Эта rule не в v1.0 priority policy. Surface finding: возможный кандидат в ADR-0003 v1.1 priority policy.
## История изменений
- 2026-05-29 — v1.0 draft. Валидирует ADR-0005 полным циклом snapshot → retrieval с deprecation signal → explicit remap. Surfaced: `remap_source` (патч ADR-0005), audit snapshot vs computed split, dedup по (source, version).
- 2026-05-29 — все surface findings закрыты в патче ADR-0005 (remap_source, snapshot/computed split, idempotency dedup-ключ).
- 2026-06-05 — все три envelope приведены к полному канону output.schema: добавлены `status`, `primary`/`alternatives` с `system`+`code`, `value.string`, отдельный `context`-объект и все 16 audit-полей (включая `method_alias_group`, `ucum_conversion`, reproducibility-поля).