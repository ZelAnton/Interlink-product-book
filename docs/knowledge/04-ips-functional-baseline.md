# 04. Функциональная база IPS

## 1. Зачем IPS остаётся базой сравнения

IPS важен для Interlink не как схема БД, которую нужно скопировать, а как накопленный набор инженерных задач и привычных результатов. Его сильная сторона — плотное соединение локальной машиностроительной семантики:

- ЕСКД/ЕСТД и техническая документация;
- объект, версия, состав и извещение;
- CAD/PDM и технологическая подготовка;
- НСИ, классификация и инженерный поиск;
- применяемость, конфигурирование и заказ;
- архив/ОТД, подписи и выпуск;
- обмен, производственные ведомости и специализированные рабочие места.

Interlink должен сохранить эту предметную планку, даже если функции будут распределены между ядром, доменными модулями и хостом иначе. [IPS-DOSSIER], [IPS-STRENGTHS].

## 2. Важная оговорка: «IPS» не является одной конфигурацией

Реальная установка определяется комбинацией:

- версии и service pack;
- лицензий и подключённых модулей;
- схем жизненного цикла и уровней продвижения;
- типов объектов, связей, атрибутов и форм;
- правил подбора, AutoMatch/Expert и конфигуратора;
- CAD-коннекторов, плагинов и пользовательских DLL;
- структуры архивов, подписей и внешних интеграций;
- данных, объёмов, глубины составов и эксплуатационных ограничений.

Поэтому утверждение «покрывает IPS» без Customer Configuration Baseline не проверяемо. База ниже описывает продуктовый канон/реконструкцию, а каждый проект миграции должен дополнить её локальным профилем. [IPS-A34], [IPS-A38], [IPS-A44], [IPS-UNCERTAINTY].

## 3. Функциональная карта

| Домен | Потребность пользователя | Базовые сущности/механизмы IPS | Значение для Interlink |
|---|---|---|---|
| Метаданные и object kernel | Расширять предметную модель | типы, атрибуты, relations, формы, domains | Сохранить выразительность, сменить способ поставки. |
| Identity/version/lifecycle | Хранить историю инженерных состояний | object, version, base/current, work copy, iteration, steps/levels | Сжать в Identity/Revision/Draft/Changeset/Maturity. |
| Состав и where-used | Управлять EBOM/MBOM/production views | typed relation, quantity, position, context, recursive navigation | Ядро Interlink, G4 и occurrence semantics. |
| Version selection | Получать правильную дочернюю версию | concretion, applicability, edit context, rules, fallback | Единый explicit resolver с reasons. |
| Configuration | Строить исполнения и точные продукты | options, values, implications, incompatibilities, conditions | OC-A/OC-B domain contracts + coordinates. |
| Changes | Выпускать связные изменения | notices, journals, editing contexts, workflows, signatures | Changeset в ядре, process в хосте. |
| Documents/archives | Управлять файлами и подлинниками | files, vault, OTD, signatures, copies, print packages | Domain artifacts + external vault + ContentHash. |
| Search/classification | Находить и повторно использовать | selections, classifiers, common index, IMBase, IMShape | Typed queries + projections + domain search. |
| CAPP/technology | Формировать техпроцессы и нормы | Techcard, routes, operations, materials, labour, equipment | Domain modules поверх object kernel. |
| AutoMatch/Expert | Подбирать и рассчитывать по правилам | scenarios, conditions, formulas, candidates, trace | Versioned typed rule artifacts/application services. |
| Orders/MRP | Зафиксировать production configuration | order tree, instances/lots, route choices, production sheets | Order domain object + immutable order snapshots; host process. |
| Requirements/projects/SPDM | Связать замысел, проект и расчёты | requirement tree, tasks, solver runs, evidence | Alvatune/domain modules; exact Interlink references. |
| Integrations/exchange | Передавать связный граф | XML, API, WebPortal, ownership, packages, logs | Stable domain contracts, explicit authority and idempotency. |
| Extension platform | Настраивать installation | modules, plugins, forms, SDK, Database Configurator | Versioned modules, typed extension pools, no arbitrary core hooks. |
| Operation | Эксплуатировать enterprise system | scheduler, HA, backup, observability, support | Host/platform services and measured SLO. |

Основания: [IPS-M01], [IPS-DOSSIER], аспекты 01–44.

## 4. Продуктовые персоны IPS

### Инженерные авторы

- конструктор и CAD-пользователь;
- технолог;
- расцеховщик;
- заготовщик;
- нормировщик материалов и труда;
- конструктор оснастки;
- CAE/SPDM-инженер;
- инженер по требованиям.

### Управление изменением и выпуском

- инициатор изменения;
- редактор извещения;
- рецензент/согласующий/подписант;
- архивист/ОТД;
- оператор печати и учёта копий;
- руководитель проекта/главный специалист.

### Данные и платформа

- администратор метамодели и Database Configurator;
- администратор НСИ/IMBase;
- администратор Workflow/прав;
- интегратор и разработчик расширений;
- migration team/data steward;
- эксплуатация, поддержка и аудит.

### Производство и внешние участники

- производственный планировщик/диспетчер;
- пользователь ERP/MES interface;
- поставщик/кооператор/удалённая площадка;
- исполнитель задания и делопроизводитель.

Полный продукт должен сохранять job-to-be-done этих ролей, но не обязан копировать их desktop-экраны. [IPS-A17], [IPS-A18], [IPS-A23], [IPS-A32], [IPS-A41].

## 5. Сквозные сценарии, которые нельзя потерять

### Сценарий A. Изменение изделия и выпуск

1. Найти exact исходные versions детали и сборки.
2. Взять их на изменение в согласованном контексте.
3. Изменить атрибуты, документы и состав.
4. Проверить влияние на where-used.
5. Согласовать/подписать exact комплект.
6. Провести все resulting versions атомарно.
7. Назначить применяемость и состояние выпуска.
8. Сохранить неизменяемую историю и точный переданный состав.

Критические инварианты: не править released version in place; не провести половину комплекта; approval относится к exact content; старый состав остаётся читаем. [IPS-M03 §Сценарий 2], [IPS-A09], [IL-VERS-IPS §С2].

### Сценарий B. Чтение динамического состава

1. Открыть exact либо selected root revision.
2. Для каждого occurrence определить target identity.
3. Применить concretion/applicability/edit context/configuration/rule.
4. Получить не более одной target version либо явную проблему.
5. Сохранить occurrence quantity/position/context.
6. Продолжить recursion с контролем cycles/depth.
7. Показать статус/причину выбора.

Подробное сопоставление — документ 05. [IPS-M03 §Сценарий 1], [IPS-K03].

### Сценарий C. Производственный заказ

1. Выбрать головное изделие, series/date и комплектацию.
2. Разрешить versions, variants, substitutions и routes.
3. Устранить ambiguity.
4. Зафиксировать exact order tree с occurrence payload.
5. Сформировать documents/production sheets/MRP views.
6. Материализовать партии/экземпляры и фактические отклонения.
7. Не менять прошлый заказ после изменения нормативного изделия.

Инвариант: order result — самостоятельное evidence, а не повторно вычисляемый today-view. [IPS-M03 §Сценарий 3], [IPS-A13].

### Сценарий D. Технологическая подготовка

1. Получить выпущенный или рабочий конструкторский состав.
2. Построить/выбрать маршрут и операции.
3. Подобрать материалы, оборудование, оснастку и нормативы.
4. Использовать AutoMatch/Expert с exact rule/input versions.
5. Выпустить комплект технологической документации.
6. Передать exact production view и нормы.

Interlink должен дать устойчивые objects/relations/queries/changesets; CAPP-семантика остаётся доменной. [IPS-A11], [IPS-A23].

### Сценарий E. Управляемый документ

1. Создать document identity/version и связать с инженерными объектами.
2. Управлять файлами, замечаниями и exact revision.
3. Подписать hash полного содержания, включая files.
4. Зарегистрировать оригинал в архиве/ОТД.
5. Выдать и учесть копии/замены.
6. Сохранить provenance и retention.

Interlink не хранит binary в предметных таблицах; vault и document lifecycle строятся хостом/модулем. [IPS-A06], [IL-PLM §8], [IL-HOST §4].

### Сценарий F. Перенос конфигурации предприятия

1. Зафиксировать source installation profile.
2. Выбрать model/module versions и dependencies.
3. Сравнить semantic model delta.
4. Проверить миграцию данных и customizations.
5. Развернуть в TEST/UAT.
6. Выполнить acceptance/regression.
7. Продвинуть exact package в PROD либо откатить по доказанному плану.

В IPS это часто распределено между configurator, скриптами, файлами и практикой внедрения. Interlink делает pipeline частью продукта. [IPS-A44], [IL-MIG], [IL-DSL §10–§11].

## 6. Сильные стороны IPS, которые следует сохранять

1. **Предметная плотность**, а не только generic object platform.
2. **Единая цифровая нить** CAD → BOM → технология → документы → изменение → производство.
3. **Relation как инженерный факт** с occurrence attributes.
4. **Контекстный подбор versions**, а не всегда hard FK на child revision.
5. **Глубокая конфигурируемость** типов, форм, workflow и ролей.
6. **Локальная нормативная семантика** и специализированные рабочие места.
7. **Глобальные identifiers и пакетный обмен** связного графа.
8. **Технологическая подготовка, НСИ и AutoMatch**, редко столь глубоко связанные с PDM.
9. **Архив/ОТД/подписи** как часть инженерного выпуска.
10. **Работа с фактической customer configuration**, а не только коробочным продуктом.

## 7. Ограничения IPS, которые нельзя перенести незаметно

- универсальная основная модель переносит часть инвариантов из БД в runtime metadata;
- dual local/global identifiers усложняют ссылки;
- несколько механизмов selection/versioning пересекаются;
- ambient contexts создают разные результаты в окнах и фоновых задачах;
- rules/Methods/forms/plugins могут образовывать несколько языков поведения;
- coded applicability затрудняет indexes и constraints;
- fallback может выглядеть как успешный selection;
- exact business result не всегда имеет самодостаточное immutable evidence;
- конфигурация между installations может drift-овать;
- external API рискует раскрывать внутреннюю метамодель;
- Bridge/plugins получают слишком широкое доверие;
- scheduler/retry/operation semantics распределены по подсистемам.

Это не означает, что каждая установка проявляет все проблемы. Это архитектурные риски, против которых Interlink ставит guardrails. [IPS-CRITIQUE], [IPS-K03 §5], [IPS-K04 §4].

## 8. Эксплуатационная планка

Функциональная эквивалентность недостаточна. Для промышленного перехода нужны:

- measured volumes/types/relations/BOM depth;
- SLO для point read, composition, where-used, search и batch operations;
- RPO/RTO и регулярно проверенный restore;
- согласованный backup БД, vault, configuration и queues;
- HA/failover и recovery long-running operations;
- audit/telemetry/business history как разные контуры;
- exact compatibility matrix ОС/СУБД/CAD/add-ins;
- migration rehearsal, reconciliation и dual-run criteria;
- поддержка diagnostic bundle и incident tracing.

Interlink PoC измеряет только часть этой планки; остальное принадлежит будущему продукту/хосту. [IPS-A27–A38], [IL-POC §5–§8].

## 9. Customer Configuration Baseline

Перед заявлением паритета нужно собрать:

| Категория | Что фиксировать |
|---|---|
| Product build | IPS version, SP/hotfix, DB/runtime, licenses |
| Modules | Search/PDM/AVS/Techcard/IMBase/Workflow/Office/etc. |
| Metadata | types, attributes, relations, lifecycle, selection rules |
| Custom code | plugins, handlers, forms, reports, scripts, DLLs |
| Integrations | CAD, ERP/MES, XML, WebPortal, file services |
| Data | counts, sizes, depth, cycles, files, archives, retention |
| Processes | notices, approvals, signatures, order and release flows |
| Security | domains/projects/roles/ACL and exceptional access |
| Operations | backups, jobs, HA, monitoring, known incidents |
| Acceptance | golden scenarios, outputs, reports, performance bounds |

Ответы одной установки нельзя автоматически переносить на другую. [IPS-A34], [IPS-A38], [IPS-UNCERTAINTY].

## 10. Минимальный набор golden scenarios для диалога

1. Floating BOM с released/current/last policy.
2. Hard concretized child и конфликт с applicability.
3. Edit context, содержащий новую version компонента.
4. Rule fallback «не совсем подходящая» version.
5. Series/date applicability с head product.
6. N:M allowed substitution.
7. Configurable occurrence с unknown/default/explicit option value.
8. One assembly reused twice; positions distinguished by full path.
9. Change notice touching parent, child, documents and applicability.
10. Exact order publication and re-read after source evolution.
11. CAD/BOM update with ownership conflict.
12. Metadata/customer customization promotion across environments.
13. Recursive where-used and quantity roll-up on production volume.
14. External exchange with stable IDs, files and replay.
15. Background job whose context cannot change underneath it.

Эти сценарии должны стать общим языком продуктовиков IPS и команды Interlink.

## 11. Вывод

Функциональная база IPS — это не список экранов и не набор таблиц. Это система пользовательских исходов и инженерных инвариантов. Interlink успешен, если его новая композиция механизмов:

- даёт те же необходимые результаты;
- делает границы и причины выбора явными;
- сохраняет точную историю;
- допускает controlled customization;
- измеримо работает на реальных объёмах;
- не требует копировать архитектурные отказные режимы IPS.
