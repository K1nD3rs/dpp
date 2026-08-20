# Анализ тестов проекта dpp

Вся тестовая система находится в каталоге `tests/` и построена на фреймворке
**unit_threaded**. Тесты проверяют как отдельные функции (unit-тесты), так и
полный цикл трансляции и компиляции (`#include` → D → dmd) — они запускают
реальные компиляторы `dmd` / `clang` / `g++`.

## Структура каталога

```
tests/
├── test_main.d     — главный раннер всех тестов
├── common/         — общие утилиты тестов
├── ut/             — unit-тесты
├── contract/       — контрактные тесты libclang
└── it/             — интеграционные тесты
    ├── c/          — тесты C (compile / run / dstep)
    └── cpp/        — тесты C++
```

## Общая инфраструктура

- **`tests/test_main.d`** — точка входа. Подключает **два набора** (отдельно для
  версии `dpp2` и для обычной сборки), а с версии `version(dpp2)`:
  - `dpp2`: `ut.translation.type.*`, `ut.translation.node.structs`,
    `ut.transform.*`, `it.c.compile.struct_`;
  - обычная сборка: полный набор `dpp.runtime`, `dpp.translation`,
    `macro_`, `expansion`, все unit/contract/ite tests (включая копированные из
    dstep) и весь C/C++.
- **`tests/it/package.d`** — целая «песочница» для интеграционных тестов:
  - `IncludeSandbox` — временная папка с методами
    `writeFile`, `shouldCompile`, `shouldNotCompile`, `shouldCompileButNotLink`,
    `shouldCompileAndRun`, `runPreprocessOnly`;
  - UDA-маркеры **`C`, `Cpp`, `D`, `RuntimeArgs`, `DFlags`** — позволяют оборачивать
    прямо в теле теста куски C/C++/D-кода;
  - выбор компилятора: `dCompiler()` — из переменной окружения `DC` (по умолчанию
    `dmd`), `cCompilerName`/`cppCompilerName` — `clang`/`gcc` или `clang++`/`g++`
    (на Travis — gcc/g++);
  - `writeHeaderAndApp` пишет `.h`/.hpp + `app.dpp` и запускает полный цикл.
- **`tests/common/`** — утилиты: `printChildren`, `shouldMatch` (kind+spelling).

## 1. Unit-тесты (`tests/ut/`)

Модульные тесты отдельных функций:
- `ut/expansion.d` — разбор `#include "foo.h"` / `<foo.h>` через `getHeaderName`.
- `ut/old/type.d` — перевод типов (старые unit-тесты).
- `ut/package.d` — реэкспорт `unit_threaded` и `text`.

## 2. Контрактные тесты (`tests/contract/`)

Контрактные тесты на поведение **libclang** (по идее Contract Test Мартина
Фаулера). Ключевая особенность — фреймворк из `contract/package.d`:

- `@ContractFunction` / `mixin Contract!(...)` — по одному фрагменту C/C++-кода
  тест **проверяет реальный `Cursor` libclang** (режим `TestMode.verify`), и
  одновременно строит **`MockCursor` / `MockType`** (`TestMode.mock`), который
  ведёт себя идентично. Это позволяет проверять сложные случаи и на живом
  libclang, и на «симуляторе» (unit-тесты без внешних зависимостей).
- Помощники: `expect`, `expectLength`, `expectEqual` (ассерт или установка
  значения в зависимости от режима).

Покрытие (`tests/contract/*.d`):
- `aggregates` — struct/union/enum, поля, вложенность;
- `array`, `constexpr`, `enums`, `functions`, `inheritance`, `issues`,
  `macro_`, `member`, `methods`, `namespace`, `operators`, `templates`,
  `typedef_`.

Отдельные входы: `tests/contract/main.d` — прогон только контрактных тестов.

## 3. Интеграционные тесты (`tests/it/`)

Самый большой набор — проверяют полный цикл «заголовок → D/компиляция», иногда
с линковкой и запуском.

### Общие `it`-модули (в корне папки)
- `issues.d` (≈2 259 строк) — каждый тест соответствует **конкретному GitHub-issue**
  dpp (тег `@Tags("issue")`, имя — номер issue). Ключевой источник защиты от
  регрессий трансляции.
- `expansion.d` — разворачивание `#include`, работа с namespace-ами, курсорами.
- `docs.d` — сохранение **комментариев/документации** C/C++ в D-коде.

### C-тесты (`it/c/`)
- **compile** (`it/c/compile/*`):
  - `preprocessor` — макросы и препроцессинг;
  - `struct_`, `union_`, `enum_`, `array`, `typedef_`, `function_` — типы;
  - `projects` (957 строк) — известные кейсы из **реальных C-проектов**
    (многобайтовые литералы `'ABCD'`, `L""`-строки и т.п.);
  - `runtime_args` — аргументы командной строки для .dpp;
  - `collision` — конфликты имен → `pragma(mangle)`;
  - `extensions` — расширения языка.
- `run` (`struct_`, `c`) — тесты, считая которые C-объявления**компилируются + линкуются + запускаются** (через `shouldCompileAndRun` с реальным C/C++ компилятором).
- `dstep` (`ut`, `functional`, `issues`) — тесты, **перенесённые из проекта dstep**.

### C++-тесты (`it/cpp/`)
- `function_`, `class_`, `templates`, `misc` — перевод C++-конструкций;
- `opaque` — **непрозрачные типы** (`dpp.Opaque`);
- `run` (669 строк) — C++, который должен реально выполняться: конструкторы
  (move/copy), виртуальные функции и т.п.

## Точки входа

- `tests/test_main.d` — основной раннер со всеми тестами обеих сборок.
- `tests/contract/main.d` — только контрактные.
- `tests/it/main.d` — только интеграционные (C + C++).

## Вывод

Тесты — важная часть инженерии проекта:
- **contract + mock-среда** гарантируют стабильность взаимодействия с libclang;
- **огромный набор интеграционных C/C++ тестов** (`it/`, особенно `it.issues` и
  C++-тесты) ловит регрессии трансляции на реальных конструкциях и реальных
  компиляторах;
- благодаря UDA-структуре в одном тесте — это и C-заголовка, и D-код, и запуск,
  что упрощает добавление новых регрессионных случаев.