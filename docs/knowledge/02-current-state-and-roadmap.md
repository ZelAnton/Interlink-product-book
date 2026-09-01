# 02. Текущее состояние и путь реализации

## 1. Срез состояния

Дата: **2026-08-31**. `HEAD` Interlink — `3df67f5`, ветка `main`, совпадает с локальным `origin/main` на момент исследования.

Рабочее дерево содержит значительный незакоммиченный этап 8 фазы 2. В этой базе он отмечается как **локальная работа**, а не как принятая возможность.

## 2. Что уже доказано

### Реализовано в `main`

- IR метамодели, builders и validator базовых инвариантов;
- контракты query algebra, session/runtime и host-neutral типов;
- собственный lexer и recursive-descent parser DSL;
- composition модулей, imports и порядок `after`;
- binding AST → IR, диагностики и восстановление после ошибок;
- нормализованный IR v2, manifest v2 и SHA-256 hash;
- stable IDs, maturity scale, seed declarations и named queries в грамматике;
- контракты Coordinates, ThreadPath и StructureDiff;
- versioned module как единица поставки: version, prefix, requires и `modules[]`; versionless module остаётся организационной единицей и средством порядка;
- G1/G2 codegen до record materializers, реализованный по этап 7 фазы 2;
- sample как эталон формы будущего generated API;
- контрактные тесты, фиксирующие публичную поверхность и phase gates.

Это доказывает, что модель может быть разобрана, связана, нормализована и превращена в значительную часть типизированного C#. Это пока не доказывает развертывание предметной БД и выполнение PLM-запроса. [IL-README §Разработка], [IL-PLAN-2], [IL-CODE-COMPILER], [IL-CODE-CODEGEN].

### Локальная незакоммиченная работа

- `ModelGenerator.Generate` как единая дверь генерации;
- CLI `interlink generate <project>`;
- `generate --check` с побайтной сверкой;
- очистка только принадлежащих генератору `*.g.cs`;
- перевод sample G1/G2 на файлы `Generated/*.g.cs`;
- регенерационный CI-step;
- тесты безопасной работы с путями, публикации и диагностики.

Этап 9 — golden-приёмка и закрытие всей фазы 2 — ещё не начат. Следовательно, корректная формулировка: **«этап 8 реализован локально; фаза 2 не принята»**. [IL-PLAN-2 §Этап 8, §Этап 9], [IL-WIP-GENERATE].

На этом рабочем дереве точная последовательность локального CI — Release build, затем `dotnet test Interlink.slnx --no-build -c Release` — прошла **3837 из 3837** тестов. Это подтверждает согласованность текущего среза, но не готовность runtime: заметная часть тестов фиксирует контракты и ожидаемые phase gates. Комбинированный вызов `dotnet test ... --no-restore` без предварительной отдельной сборки в этой среде обнаружил ноль тестов и завершился с кодом 5; поэтому его нельзя принимать за эквивалент CI или доказательство зелёного прогона.

## 3. Что существует только как контракт

В sample и библиотеках уже видны формы будущих API, но многие тела сознательно бросают `PhaseNotImplementedException`.

К этой категории относятся:

- рукописные frozen specimens будущих generated typed command facades; текущий `ModelGenerator` команды ещё не выпускает;
- обычный writer объектов, ревизий, связей, списков и кортежей;
- lifecycle/disposition operations;
- G3 query execution;
- G4 structural query execution;
- resolution policies beyond carrier types;
- snapshot command;
- host-enlisted object commit semantics;
- cache invalidation/fence runtime.

Наличие сигнатуры защищает будущую совместимость. Оно не является свидетельством готовой функции. [IL-POC §3, фазы 0, 3–8], [IL-CODE-RUNTIME], [IL-SAMPLE].

## 4. Чего ещё нет в solution

На текущем срезе отсутствуют исполняемые слои, которые превращают проект в работающий object store:

- PostgreSQL migration planner/emitter/runtime;
- развернутый `meta` catalog и DDL предметных таблиц;
- `validate-db`, drift detection против реальной БД;
- реализация writer и audit/fence;
- SQL compiler/renderer/executor полного G3/G4;
- интеграционная Testcontainers-фикстура фаз 3–8;
- runtime implementation Changeset/Effectivity/business Snapshot;
- tenant storage profile.

Локальная bootstrap-площадка PostgreSQL существует, но это инфраструктурный стенд, а не доказанный предметный pipeline. [IL-README §Текущая фаза], [IL-POC §3].

## 5. Фазовая карта

| Фаза | Содержание | Статус 2026-08-31 | Что она доказывает |
|---|---|---|---|
| 0 | IR, Query/Runtime contracts, frozen generated surface | Реализовано | Понятия и API можно зафиксировать до физики. |
| 1 | DSL compiler, diagnostics, composition, normalization, manifest | Принято | Текстовая модель даёт стабильный проверяемый IR. |
| 1.5 | Configuration contracts: Coordinates, ThreadPath, Snapshot forms | Принято | Будущие IPS/configurator/DT-сценарии не требуют ломать carrier types. |
| 1.6 | Module delivery contracts | Принято | Независимые модули можно скомпоновать и идентифицировать. |
| 2 | G1/G2 codegen и CLI generation | В работе; этапы 1–7 в `main`, этап 8 локально, этап 9 открыт | Модель даёт пригодную статическую C#-поверхность. |
| 3 | Migrations, catalog и минимальный writer | Не начато | Реальная БД соответствует DSL, запись идёт одним путём. |
| 4 | G3 current-only | Не начато | Типизированное простое чтение и явный минимальный context. |
| 5 | G4 current-only | Не начато | BOM/where-used/roll-up/paths на реальном PostgreSQL. |
| 6 | Live resolver и streaming | Не начато | Pin + policies + per-edge policy + причины и тяжёлые планы. |
| 7 | Ownership cycle | Не начато | Customer customization переживает deploy и возвращается в DSL. |
| 8 | Load + technical projection snapshot | Не начато | Архитектура выдерживает объёмы и materialized snapshot-safe outgoing occurrence graph. |

Фазы 0–5 дают обязательное ядро; 6–8 нужны для вывода об идеологической устойчивости. [IL-POC §3].

## 6. Критический вертикальный путь

```text
P2 accepted codegen
  ↓
P3 writer contract tests
  ↓
P3 seed through commands
  ↓
P4 current-only typed root
  ↓
P5 current-only recursive structure
  ↓
P6 live policies, reasons and streaming
  ↓
P8 projection snapshot + load evidence
```

Порядок важен:

- seed не должен создавать второй raw-DML путь до writer-а;
- структурный запрос не должен скрыто выбирать ревизию до context injection;
- resolver нельзя считать доказанным только на in-memory модели;
- snapshot строится только после полного G4 и atomic publication;
- нагрузочный вывод имеет смысл только на той же семантике, которая пойдёт в продукт.

## 7. После PoC

Принятый порядок расширения:

1. Полный многообъектный Changeset и `CommitBundle` поверх typed staging.
2. Нормативная проверка host seam: object commit + host audit/event/outbox в одной транзакции.
3. OC-A: Effectivity, declared axes, constant coordinates и минимальный configurator.
4. Business API `SnapshotKind.Order` и `snapshotref`.
5. Classification schemes.
6. OC-B: распространение option values по пути, nested configurations и N:M substitutions на occurrence threads.
7. Policy IR и настраиваемые compiled policies.
8. Artifact vault и ContentHash с файлами.
9. Workflow поверх Changeset.
10. Полный tenant storage profile.
11. Второй SQL dialect — только при доказанной необходимости.

Параллельно развивается общий фундамент `Unit`/`Lot`, as-built и historized facts для IPS order parity и digital twin. [IL-POC §9], [IL-ADR ADR-051–052].

## 8. Связь с Alvatune

Alvatune делит upstream на три обязательных gate:

| Gate | Interlink должен доказать | Что блокируется без gate |
|---|---|---|
| `I0` | Полный object-kernel PoC | Надёжная композиция доменной модели Alvatune. |
| `I1` | Full versioning + host seam | Permanent object writes, exact approval/commit flow и общий outbox boundary. |
| `I2` | Tenant storage profile | Multi-tenant storage, provisioning, pool/cache isolation и rollout. |

Alvatune может параллельно проводить discovery и clickable/concierge proof, но не должна создавать второй generic Record kernel. [ALV-ADR068], [ALV-TRANSITION §6–§8].

## 9. Что можно демонстрировать сейчас

- DSL реалистичной инженерной модели;
- диагностики неправильной модели;
- deterministic normalized IR/manifest;
- module composition и dependency checks;
- generated descriptors, identifiers, records, links, catalog и record materializers;
- planned typed command/query API;
- продуктовую модель resolver-а и snapshot на документах;
- связь с будущим host seam.

## 10. Что нельзя демонстрировать как готовое

- создание/миграцию предметной PostgreSQL schema из DSL;
- запись/commit реальных объектов и ревизий;
- исполнение BOM/where-used на PostgreSQL;
- production-ready version selection;
- full Changeset/Effectivity/configurator;
- неизменяемый business order/baseline;
- tenant isolation;
- Agent controls pane поверх работающего Interlink;
- функциональную эквивалентность конкретной установки IPS.

## 11. Расхождения документации, влияющие на статус

1. `README.md` отстаёт от локального этапа 8 и всё ещё называет CodeGen каркасом.
2. `overview/README.md` описывает target architecture и не является readiness report.
3. Старые `analysis/investigations/*current-state*` могли быть написаны до старта фазы 2.
4. Фраза «одинаковый normalized DSL → одинаковый C#» теперь требует также exact module delivery map и CodegenVersion.
5. «Каждый business result можно зафиксировать snapshot» ограничен snapshot-safe full outgoing occurrence graph.
6. Часть Alvatune skeleton descriptors ещё отражает отменённую generic Record-модель.

Эти расхождения не отменяют решения, но должны быть устранены либо оговорены перед внешней продуктовой презентацией.

## 12. Evidence gates для приёмки фаз

### Фаза 2

- весь generated set детерминирован;
- sample собирается только из принятого output;
- `generate --check` зелёный в чистом checkout;
- public API golden осознанно принят;
- этап 9 закрыт и план помечен принятым;
- README синхронизирован со статусом.

### Фаза 3

- чистая БД разворачивается из DSL;
- повтор миграции идемпотентен;
- drift блокирует запуск по принятой политике;
- seed идёт только через writer;
- immutable/fixed history и CAS доказаны реальной БД;
- rollback не оставляет частичных данных;
- PoC host-enlisted writer ownership/isolation/fence semantics испытаны; полный `ContentHash`/`CommitPermit` и атомарность object commit + host outbox остаются воротами I1.

### Фазы 4–6

- нет overload без context у revision-selecting query;
- unsupported profile отказывает до SQL;
- pin сильнее policy;
- `NoMatch` наблюдаем;
- where-used не воскрешает историческую parent revision;
- generated SQL сравнен с эталонным;
- cache key и invalidation учитывают полный fingerprint.

### Фаза 8

- полный `projection` snapshot разрешённого snapshot-safe outgoing occurrence graph атомарен; `baseline`/`order` — post-PoC business kinds;
- любой recursive `NoMatch` даёт ноль header/entries;
- provenance artifacts и readers проверяются;
- планы generated structural queries на тех же наборах данных укладываются в критерий ≤1.5× рукописного эталона;
- измерены UUID, глубина, ширина, table count и snapshot scan;
- вопросы из `IL-ADR §2`, для которых источником решения назначены измерения фазы 8, получили данные; эксплуатационные и host-dependent вопросы остаются открытыми до соответствующего опыта.

## 13. Продуктовый критерий завершения PoC

PoC завершён не тогда, когда существует много кода, а когда можно на одном сквозном сценарии показать:

```text
изменение DSL
→ осмысленный diff и миграция
→ типизированная запись инженерных объектов
→ рекурсивный состав с явным подбором и причинами
→ воспроизводимый технический снимок
→ сравнимые с эталоном планы и измеренная цена
```

До этого Interlink остаётся хорошо проработанным и частично реализованным основанием, но не доказанной платформой.
