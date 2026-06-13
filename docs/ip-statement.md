# Положение о независимом IP
## Краткая формулировка (для публикации)
Данная reference-архитектура является личной интеллектуальной собственностью автора, разработанной независимо от любого работодателя или коммерческого контракта. Архитектура не инкорпорирует проприетарный код, схемы, таблицы маппинга или внутренние имплементации какой-либо компании. Любые будущие коммерческие имплементации в рамках консалтинговых договоров регулируются отдельными контрактами и не влияют на открытость данного архитектурного артефакта.
## English version (для размещения в репозитории)
```javascript
This reference architecture is the author's personal intellectual property,
developed independently and prior to publication. It does not incorporate
proprietary code, schemas, mapping tables, or internal implementations from
any employer or commercial engagement. Any future commercial implementations
under enterprise consulting arrangements are governed by separate contracts
and do not affect the openness of this architectural artifact.
```
## Зачем это положение существует
Три причины:
1. **Защита от NDA-конфликтов.** Архитектура разрабатывается независимо, параллельно с работой по NDA с работодателем (компания не называется намеренно). Явное положение чисто отделяет публичное IP от любых внутренних реализаций работодателя и снимает риск претензий.
2. **Сигнал для enterprise procurement.** Apache 2.0 + явная патентная оговорка + положение о независимом IP = enterprise-команды могут брать архитектуру без юридического анализа происхождения каждой строки.
3. **Подкрепление позиции «независимый архитектурный консалтинг».** В B2B-разговорах архитектура — материальный артефакт, который автор приносит как свой. Без явного IP-statement позиция держится только на словах.
## Лицензия: dual licensing (финал v1.0)
Репозиторий лицензируется по двойной модели:
- **Код, схемы, примеры** — [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0). Канонический файл: `LICENSE`. Охват: `schemas/*.json`, примеры в `examples/**/*.json`, любой reference-код и `transliteration_fixtures` в `docs/ucum-transliteration.md`
- **Документация** — [Creative Commons Attribution 4.0 International (CC-BY 4.0)](https://creativecommons.org/licenses/by/4.0/). Канонический файл: `LICENSE-docs`. Охват: `README.md`, `docs/**/*.md` (ADR, architecture, scope, preconditions, versioning, glossary, ip-statement, ucum-transliteration), `examples/**/*.md`
### Почему dual, а не single Apache 2.0
1. **Архитектура — прежде всего документ.** Объёмная часть репозитория — проза (ADR с rationale, scope discipline, граничные случаи, IP-statement). Apache 2.0 покрывает их по факту, но документы обычно лицензируются CC-BY — это ожидаемая для читателя модель
2. **Attribution-дружелюбность для портфолио.** CC-BY явно требует attribution при цитировании. Это работает на portfolio-функцию: всякий, кто берёт в работу ADR-ы или scope discipline, должен упомянуть источник
3. **Нет SA-вируса.** CC-BY (не BY-SA) — производные работы (например, продуктовые внутренние документы enterprise-команд) не обязаны публиковаться под той же лицензией. Это enterprise-friendly
4. **Patent grant в кодовой части.** Apache 2.0 явно выдаёт patent grant от контрибьюторов к пользователям — критично для любого артефакта в healthcare-домене
5. **Совместимость с отраслевыми прецедентами.** FHIR издаётся HL7 по CC-BY-подобной модели; LOINC — по собственной free-license с attribution-требованиями; UCUM — free non-commercial + attribution. Двойная модель выравнивается с ожиданиями домена
### Что это означает практически
- Кто берёт schemas + код в продукт — следует Apache 2.0 (NOTICE + attribution + patent grant)
- Кто цитирует ADR-ы/scope/архитектурные принципы в внутренней документации или публикациях — следует CC-BY 4.0 (attribution с именем автора и ссылкой на репозиторий)
- Граница «код/схемы» vs «документы» проводится по расширению файла: `.json` и `.ts/.py/.sql` → Apache; `.md` → CC-BY. Смешанных файлов нет (markdown-блоки `json` внутри `.md` — иллюстрации, отдельно не лицензируются)
- README в верхнем блоке явно указывает обе лицензии в формате SPDX (`Apache-2.0 AND CC-BY-4.0`)
### Отказы и ограничения
- Ни одна из лицензий не покрывает LOINC-коды и LOINC-объявления — они не являются IP автора и регулируются LOINC лицензией Regenstrief Institute
- UCUM-токены в schemas и fixtures — фактологические ссылки на UCUM-стандарт, подпадают под UCUM Copyright Notice
## Где размещается в репозитории
- Отдельный файл `docs/ip-statement.md`
- Ссылка в README в разделе «Independent IP» одной строкой
- Упоминание в LICENSE-блоке: «See `ip-statement.md` for the author's IP position regarding employers and commercial engagements.»
- `LICENSE` (Apache 2.0 полный текст, SPDX header `Apache-2.0`) в корне репозитория
- `LICENSE-docs` (CC-BY 4.0 deed reference + ссылка на legalcode, SPDX header `CC-BY-4.0`) в корне репозитория
- README license-блок: SPDX expression + пояснение «code under Apache 2.0; docs under CC-BY 4.0» и ссылки на оба файла
## Что НЕ декларируется
Положение не утверждает:
- что архитектура «революционна» или «впервые в индустрии» — это структурная сборка известных стандартов с явными инвариантами
- что архитектура не пересекается с патентами третьих сторон — Apache 2.0 даёт patent grant в сторону пользователей, но не страхует от внешних claims
- что любая будущая коммерческая работа автора будет открытой — наоборот, явно разделяет открытое IP и коммерческие контракты
## Сценарии использования
- Контрагент по B2B-консалтингу спрашивает «откуда у вас эта архитектура» — ссылка на репозиторий + ip-statement
- Текущий или будущий работодатель претендует на эти материалы — ссылка на дату публикации репозитория и ip-statement
- Procurement-юристы enterprise-заказчика проверяют чистоту IP — статус «Apache 2.0 + ip-statement» снимает большую часть вопросов