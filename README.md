# LOINC/UCUM Reference Architecture
Reference-архитектура нормализационного слоя для лабораторных данных. Архитектурный артефакт, не библиотека: реализация ожидается в Phase 2 (см. scope).
## Статус
- design: v1.0 complete
- implementation: planned (Phase 2)
- license: `Apache-2.0 AND CC-BY-4.0` (код/схемы под Apache 2.0; документация под CC-BY 4.0)
## Зачем эта архитектура существует
Интероперабельность лабораторных данных формально решена: есть LOINC (коды наблюдений) и UCUM (единицы измерения), оба — открытые международные стандарты. Но типовые интеграции собирают их как абстрактный справочник без явных архитектурных инвариантов. Характерные паттерны отказа:
- код без единиц измерения, единицы — в свободной строке
- потеря семантического контекста (sample type, метод, состояние натощак)
- маппинги без версионирования, тихо деградирующие при обновлении стандартов
- разрешение конфликтов по ситуации на уровне интеграционного кода
Проблема не в стандартах, а в отсутствии архитектурного слоя поверх них.
## Эмпирическое подтверждение
Архитектурные инварианты слоя — обязательный UCUM-препроцессинг, explicit priority policy + alternatives, version-aware mapping, semantic context as first-class — сформулированы как структурные требования независимо от того, какими стандартами они реализуются. LOINC и UCUM выбраны как наиболее зрелая реализационная лексика для этих инвариантов, а не как отправная точка дизайна.
Эмпирически разрыв подтверждается типовым ландшафтом российского healthtech: крупные платформы строят интеграцию через внутренние «абстрактные справочники биомаркеров» без UCUM-канонизации и version-aware mapping. Это типовая, а не маргинальная картина.
## До и после
**До** — фрагмент, который приходит из OCR-системы выше по потоку:
```json
{
  "lab": "Helix",
  "date": "2026-04-02",
  "type": "кровь",
  "biomarkers": [
    {
      "raw_name": "Гемоглобин",
      "raw_value": "110",
      "raw_unit": "г/л",
      "raw_ref": "от 1 до 14 лет 110-150",
      "raw_comment": null
    }
  ]
}
```
**После** — что получает потребитель после нормализационного слоя:
```json
{
  "status": "success",
  "mapping_id": "map_2026-04-02_a1b2c3d4",
  "primary": {
    "system": "LOINC",
    "code": "718-7",
    "display": "Hemoglobin [Mass/volume] in Blood"
  },
  "value": {
    "numeric": 110,
    "string": null,
    "unit": "g/L",
    "ucum_canonical": "g/L"
  },
  "context": {
    "sample_type": "blood",
    "fasting_status": null,
    "method": null,
    "context_completeness": "partial"
  },
  "alternatives": [
    {"system": "LOINC", "code": "30313-1", "display": "Hemoglobin [Mass/volume] in Arterial blood", "score": 0.62}
  ],
  "audit": {
    "loinc_version": "2.82",
    "ucum_version": "2.2",
    "mapping_timestamp": "2026-04-02T10:23:00Z",
    "primary_score": 0.98,
    "top_score_by_resolver": 0.98,
    "score_delta_vs_top": 0,
    "rule_applied": null,
    "rule_overrode_score": false,
    "method_alias_group": "loinc:component:e3b0c4",
    "ucum_conversion": null,
    "remap_source": null,
    "current_version_status": "current",
    "replaced_by": null,
    "candidates_returned": 2,
    "policy_version": "1.0.0",
    "resolver_version": "semantic-resolver-v1"
  }
}
```
Ключевое различие — не «добавили коды», а зафиксированная структура контракта: единицы канонизированы, альтернативы и число confidence (`audit.primary_score`) доступны явно, audit trail трассируем, версия стандарта прибита к каждому маппингу.
## Ментальная модель
Слой состоит из трёх компонентов со строгими контрактами между ними:
```javascript
Input adapter  →  Normalization core  →  Output interface
(локализация,     (LOINC resolver +      (единый контракт
парсинг,          UCUM normalizer +      для всех потребителей)
базовая           context detector +
валидация)        conflict resolver)
```
## Что в scope, что нет
Детально — в документе scope. Коротко:
- в scope v1.0: архитектура, контракты, ADR, examples
- не в scope: OCR/парсинг (см. preconditions), клиническая интерпретация, reference ranges
Phase 2 (reference implementation, адаптеры под реальные форматы, production-grade данные LOINC/UCUM) — отдельный проект, а не обещание расширения v1.x. Это архитектурный артефакт, не библиотека: не ждите `pip install`.
## Структура репозитория
```javascript
loinc-ucum-reference-architecture/
├── README.md
├── LICENSE
├── LICENSE-docs
├── NOTICE
├── CHANGELOG.md
├── CONTRIBUTING.md
├── docs/
│   ├── architecture.md
│   ├── scope.md
│   ├── ip-statement.md
│   ├── preconditions.md
│   ├── versioning.md
│   ├── glossary.md
│   ├── ucum-transliteration.md
│   ├── capability-levels.md
│   ├── roadmap.md
│   └── adr/
├── examples/
└── schemas/
    ├── input.schema.json
    ├── output.schema.json
    └── map-response.schema.json
```
## Для кого
- healthtech-команды, строящие интеграцию с лабораторными источниками
- ML / CDSS-инженеры, работающие с медицинскими данными
- интеграторы EHR-систем и команд federated healthcare-платформ
## Как использовать
- читать как архитектурную референс-точку
- внедрять поэтапно по уровням зрелости — [Уровни зрелости внедрения (capability levels)](docs/capability-levels.md) (`docs/capability-levels.md`): выбрать уровень по текущей боли и расти additive-only
- сделать форк и адаптировать под enterprise-стек
- прислать граничный случай или вопрос через GitHub Issues
## Связанные работы и происхождение формата
Это не реализация конвертера и не замена существующим инструментам, а референс-архитектура: проектные решения задокументированы как ADR, проверены на examples и зафиксированы в схемах.
**Существующие инструменты (нормализация LOINC/UCUM).** Слой конверсии кодов и единиц уже решается рядом проектов — например, [miracum/loinc-conversion](https://github.com/miracum/loinc-conversion) (REST-сервис), [PrecisionHealthIntelligence/loinc_unit_harmonization](https://github.com/PrecisionHealthIntelligence/loinc_unit_harmonization) (гармонизация с провенансом) и валидаторы [LHNCBC](https://github.com/lhncbc/loinc-mapping-validator). Это решения уровня кода. Наша дельта — не сам конвертер, а явно задокументированная архитектура вокруг него: политика reject-don't-guess, фиксация семантики «что измерено», контракт провенанса.
**Происхождение формата.** Жанр «решения как документация» взят из зрелых open-source-проектов: [rust-lang/rfcs](https://github.com/rust-lang/rfcs) и [stan-dev/design-docs](https://github.com/stan-dev/design-docs) (научный проект, design-документы по модели Rust RFC). Шаблон ADR следует [MADR](https://adr.github.io/madr/) и формату Nygard.
**Чего мы не нашли.** Открытой референс-архитектуры этого слоя в ADR-формате — и тем более на русском — на момент публикации нет. Этот репозиторий заполняет именно этот пробел.
## Лицензия
SPDX expression: `Apache-2.0 AND CC-BY-4.0`. Dual licensing финализирована для v1.0:
- Код, JSON Schema (`schemas/*.json`), примеры (`examples/**/*.json`), reference-fixtures — [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) (файл `LICENSE`)
- Документация (`docs/**/*.md`, `README.md`, examples в markdown) — [CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/) (файл `LICENSE-docs`)
Граница проводится по расширению файла, без смешанных. LOINC-коды и UCUM-токены — вне охвата этих лицензий, регулируются Regenstrief Institute и UCUM Copyright Notice соответственно.
См. ip-statement для полного обоснования двойной модели и позицию о независимом IP.
Рабочие копии лицензий: [LICENSE](LICENSE) · [LICENSE-docs](LICENSE-docs).
## Контакт
- **Email:** [zaverachnikita@gmail.com](mailto:zaverachnikita@gmail.com)
- **GitHub:** [@alarent](https://github.com/alarent)
- **Telegram:** [@alarent](https://t.me/alarent)
