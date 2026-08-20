# Анализ приложения dpp

## Общее описание

`dpp` (d++) — это D-пакет, который позволяет включать C/C++ заголовочные файлы
напрямую в исходный код на D. Файлы с расширением `.dpp` обрабатываются
специальным образом: директива `#include` разворачивается inline, а объявления из
подключённых заголовков автоматически переводятся на D. Результатом является
валидный `.d` файл, который затем компилируется выбранным D-компилятором.

Весь исходный код находится в каталоге `source/`:

```
source/main.d                              — точка входа
source/dpp/runtime/                        — «обвязка»: разбор аргументов, оркестрация
    app.d                                  — основной поток выполнения
    context.d                              — состояние трансляции
    options.d                              — CLI-опции
    package.d                              — агрегат модуля dpp.runtime
source/dpp/expansion/package.d             — разворачивание #include через libclang
source/dpp/translation/                    — перевод C/C++ конструкций в D
source/dpp/clang/package.d                 — вспомогательные функции поверх libclang
source/dpp/from.d                          — утилита локальных импортов
```

## Точка входа (source/main.d)

```
main(args):
    Options(args)            — разбор аргументов
    options.earlyExit?       — выход (например, запрошен --help)
    run(options)             — основной поток
    on Exception → код 1    — пользовательская ошибка
    on Throwable  → код 2    — фатальная ошибка
```

## Разбор аргументов (source/dpp/runtime/options.d)

`Options` построен вокруг `std.getopt` с `config.passThrough` — неизвестные
аргументы не считаются ошибкой, а «прозрачно» передаются дальше D-компилятору.
Ключевые моменты:

- **Входной файл** — первый аргумент с расширением `.dpp` (`dppFileNames`).
- **Аргументы D-компилятора** — все остальные аргументы минус имя бинарника и
  `.dpp`-файлы, плюс сгенерированные имена `.d` файлов.
- **Автоподстановки**: если не задан `-of`, имя выходного exe выводится из имени
  `.dpp`-файла; на Windows автоматически добавляется флаг архитектуры (`-m64` и
  т.п.); если нужен `--c++-std-lib`, добавляется `-L-lstdc++`.
- **Include-пути** по умолчанию дополняются системными путями libclang
  (`systemPaths`), если не задан `--no-sys-headers`.
- Часть опций влияет на трансляцию: `--ignore-ns`, `--ignore-path`,
  `--ignore-cursor`, `--prebuilt-header`, `--function-macros`, `--hard-fail`,
  `--scoped-enums`, стандарты C/C++ (`--c++-standard`, `--c-standard`) и т.д.
- Метод `dup()` глубоко копирует `Options` (нужен, потому что `Context` создаёт
  себе собственную копию опций).

## Основной поток (source/dpp/runtime/app.d)

`run(options)`:

1. Для каждого `.dpp`-файла вызывается `preprocess(...)` — создаёт `.d` файл.
2. Если задан `--preprocess-only`, на этом остановка.
3. При необходимости (`--c++-std-lib`) генерируется и компилируется C++ «заглушка»
   `CppBoilerplateCode` (файл `cpp_boilerplate.cpp` с `std::vector<bool>`), чтобы
   подтянуть stdlib C++ при линковке (issue #102). Временный `.obj` удаляется по
   `scope(exit)`.
4. Выполняется D-компилятор (`options.dlangCompiler` + аргументы + объект
   заглушки). Неудача — ошибка с выводом лога компилятора.
5. Если не задан `--keep-d-files`, временные `.d` файлы удаляются.

### preprocess()

Это ядро генерации `.d`:

1. Создаётся временный файл `<имя>.d.tmp`.
2. `translationText(...)`:
   - читает входной файл, отделяет строки `module ...` (они сохраняются как есть);
   - из остальных строк извлекает имена `#include` (`getHeaderName`) и собирает
     список в отдельный временный файл `#include`-директив;
   - определяет язык: C или C++ (по расширению подключаемых заголовков или
     `--parse-as-cpp`);
   - создаёт `Context` и вызывает `expand(...)` — основную трансляцию;
   - после неё `context.fixNames()` выполняет финальные правки;
   - возвращает объявление модуля и переведённый код.
3. В выходной файл пишутся: строки `#undef`, объявление модуля, `preamble(...)`
   (общие D-декларации-помощники), переведённый код и «сырые» строки оригинального
   `.dpp` (не `module`, не `#include`).
4. Если макросы не игнорируются (`--ignore-macros`), файл дополнительно
   прогоняется через C-препроцессор (`clang-cpp` или `cpp`), чтобы развернуть
   переопределённые макросы. Строки, начинающиеся с `#`, вычищаются.

### preamble()

Вставляет фиксированные D-декларации: `va_list`, `Int128/UInt128`,
`__locale_data`, `_Bool`, а также структуру-помощник `dpp` с:
- `Opaque(N)` — непрозрачный тип заданного размера (для приватных/неигнорируемых
  полей);
- `isEmpty`, `Move`, `move` — замены C++-интринсиков;
- mixin-шаблон `EnumD` для дублирования enum-констант.

## Разворачивание #include (source/dpp/expansion/package.d)

`expand()`:

1. Пишет в контекст `extern(C)` или `extern(C++)` + `{` — всё переведённое
   содержимое попадает в блок с нужной линковкой.
2. Парсит временный файл с `#include` через libclang (`parseTU`) с аргументами:
   `-I<path>`, `-D<define>`, `--clang-option`, `-std=...` и флагом
   `DetailedPreprocessingRecord` (нужен для макросов).
3. `canonicalCursors()` — сортирует top-level курсоры, объединяет повторяющиеся
   объявления (в C допустимо объявить тип несколько раз, в D — нет):
   - `trueCursors()` сортирует и группирует курсоры по каноничности
     (`sortCursors`, `sameCursorForChunking`), объединяя каждую группу через
     `mergeCursors`; предпочтение отдаётся «определению», затем «канонической
     декларации»;
   - namespace-ы объединяются рекурсивно (`mergeNodes`);
   - макросы (`MacroDefinition`) переносятся в конец списка.
4. Каждый курсор переводится `translateTopLevelCursor` (пропускает «запретные»
   сущности) с учётом `Context.hasSeen/rememberCursor`, чтобы ничего не
   определять дважды.

`getHeaderName` распознаёт строки вида `#include "x"` / `#include <x>` и через
`fullPath` ищет файл по include-путям и переменной окружения `CPATH`.

## Состояние трансляции — Context (source/dpp/runtime/context.d)

`Context` хранит всю промежуточную информацию:

- `_lines` — накопленные строки переведённого кода.
- `_seenCursors` — уже обработанные курсоры (по spelling/kind/типам).
- `_nickNames` — выдуманные имена для анонимных структур (`_Anonymous_N`).
- `_aggregateDeclarations`, `_aggregateSpelling`, `_aggregateParents`,
  `_aggregateTypeLines` — книги учёта агрегатов и их переименований (необходимо,
  т.к. имя структуры может совпасть с ключевым словом D или другим именем).
- `_fieldStructSpellings` — упоминания необъявленных структур (в указателях и
  сигнатурах); они «договариваются» в конце через `declareUnknownStructs`
  (`struct Foo;`).
- `_linkableDeclarations` — функции/глобальные переменные; при конфликте имён
  добавляется `pragma(mangle, ...)`.
- `_macros`, `_functionMacroDeclarations` — уже определённые макросы.
- `_types` — объявленные пользовательские типы (для распознавания C-кастов в
  макросах).
- `_namespaces` — стек namespace-ов (для удаления префиксов из имён).
- `accessSpecifier`, `language` (C/C++).

Финальные правки — `fixNames()`:
1. `declareUnknownStructs` — объявляет встреченные в указателях, но нигде не
   объявленные структуры.
2. `fixLinkables` — при конфликте имени функции/переменной с агрегатом или
   макросом вставляет `pragma(mangle)`.
3. `fixAggregateTypes` (только для C) — заменяет временный декоратор
   `__dpp_aggregate__ <name>` на полное имя (с учётом вложенности через
   `_aggregateParents`).
4. `fixFields` — переименовывает поля, совпадающие с именами агрегатов.

## Слой перевода (source/dpp/translation/)

### translation.d — диспетчеризация по курсорам

`translateTopLevelCursor` → `skipTopLevel` (фильтрация: игнорируемые пути,
анонимные агрегаты, служебные имена `ulong`/`va_list` и т.д.) → `translate`.

`translate(cursor, context)`:
- проверяет чёрные списки (`ignoredCursors`, а также захардкоженный список
  непереводимых C++ stdlib-сущностей `ignoredCppCursorSpellings`);
- по `Cursor.Kind` выбирает переводчик из таблицы `translators` (struct/class/
  union/enum/function/field/typedef/macro/variable/namespace/template и др.);
- ловит `UntranslatableException`: при `--hard-fail` — ошибка, иначе сущность
  молча пропускается;
- `untranslatable(line)` — эвристическая проверка, что строка ещё не является
  валидным D (маркеры вроде `&)`, `(*`, `template<` и т.п.).
- `debugCursor` — лог курсоров при `--print-cursors`.

### type/package.d — перевод типов

Таблица `Translators[Type.Kind]`:

- простые типы: `long → c_long`, `unsigned → u...`, `wchar_t → wchar`, и т.д.
- `translateAggregate` (Elaborated/Enum/Record) — срезает namespace-префиксы
  (`A::B → A.B`), превращает `struct Foo` → `Foo`, переводит template-аргументы
  `Foo<int, unsigned short>` → `Foo!(int, ushort)`, даёт никнеймы анонимным
  типам;
- `translatePointer` — `*`, но если pointee — функция, `*` не добавляется
  (в D `function` уже указатель); корректно обрабатывает константность;
- `translateFunctionProto` — `int(double)` → `typeof(*(int function(double)).init)`,
  а в параметрах функций — просто `int function(double)`;
- ссылки: lvalue → `ref` в сигнатурах, `*` в остальных случаях; rvalue → `dpp.Move!(T)`;
- массивы: фиксированные `T[N]`, неполные `T[0]` (или `T*` в сигнатурах);
- `translateString` — строка замен: `< >` → `!( )`, `decltype → typeof`,
  `:: → .`, `unsigned char → ubyte`, убирает `typename/template/volatile`,
  `long long → long` и т.д.;
- `translateElaborated` — для C добавляет временный префикс
  `__dpp_aggregate__ `, который позже чинится в `fixAggregateTypes`;
- SIMD-вектора → `core.simd.<тип>N`, с чёрным списком неподдерживаемых.

### function_.d — функции, методы, операторы

- `ignoreFunction`: внутренняя линковка, удалённые функции (`= delete`),
  определения «вне класса» (`Foo::bar()`), конструкторы без параметров у
  struct, и т.д.
- `functionDecl` собирает: префикс (`abstract`/`final`/`override`), тип
  возврата, имя, template-параметры, параметры, `@nogc nothrow`, `const`.
- `maybeMoveCtor` / `maybeCopyCtor` — эмуляция C++ конструкторов перемещения/
  копирования через D-обёртки.
- Операторы (`maybeOperator`, `operatorSpellingD`, `operatorSpellingCpp`):
  C++-операторы переводятся в D-операторы (`opBinary!`, `opAssign`, `opCall`,
  `opIndex`, `opEquals`, `opCast`...) и генерируются функции-«форвардеры» вида
  `extern(D) ... opCppPlus(...)`, которые вызывают настоящий
  `extern(C++) operator+(...)`.
- `translateInheritingConstructor` — наследуемые конструкторы через `_Args...`
  + `forward!args`.

### aggregate.d — struct/class/union/enum

`translateStrass`:
- определяет, что получится в D: `class`, если есть виртуальные методы
  (проверяется по всей иерархии), иначе `struct` (`dKeywordFromStrass`);
- вложенные типы получают префикс `static`;
- базовые классы — `: Base1, Base2` (для class) или поле `_base0` + `alias _base0 this;`
  (для struct, чтобы не нарушать layout);
- `BitFieldInfo` — перевод битовых полей через `std.bitmanip.bitfields` с
  `align(4)` и добавлением паддинга до степени двойки (для совпадения размеров с C);
- приватные поля транслируются как непрозрачные `dpp.Opaque!N`;
- C11-анонимные struct/union эмулируются dummy-переменной + `@property`
  функциями-аксессорами (`maybeC11AnonymousRecords`, `innerFieldAccessors`);
- `maybeOperators` — если есть `operator<`, `>`, `==`, генерируется `opCmp`;
  `operator!` → `opCast!bool`;
- `maybeEnumBaseType` — если значения выходят за `int`, у enum добавляется `: long`;
- `@disable this();` для struct с явным ctor без параметров.

### enum_.d

Перевод констант enum: `name = значение,`. Если enum не `enum class` и не задан
`--scoped-enums`, константы дублируются на верхнем уровне (`enum foo = Foo.foo;`),
чтобы сохранить C-семантику глобальных имён.

### typedef_.d

- функция-typedef → `alias Name = Ret function(params);`;
- анонимный struct в typedef → превращается в обычный именованный struct
  (`typedef struct {...} Foo` → `struct Foo {...}`);
- обычные typedef → `alias Name = underlying;`;
- специальные случаи: `int32_t → int`, `uint64_t → ulong`, `nullptr_t → typeof(null)`;
- `alias Foo = Foo;` не генерируется.

### macro_.d

Макросы транслируются намеренно «по-своему»:

- встроенные/предопределённые макросы пропускаются;
- макрос-литерал (`#define FOO 42`) → `enum FOO = 42;`;
- не-литеральное выражение → попытка `enum` через mixin с проверкой
  `static if(is(typeof({ mixin(...); })))`, либо обёрточная функция;
- макросы, переопределённые после `#undef`, получают строку `#undef`;
- сам макрос переобъявляется через `#define ...`, чтобы C-препроцессор развернул
  его в пользовательском D-коде;
- при `--function-macros` генерируется D-шаблон-функция + `#define _dpp_impl_...`;
- `translateToD` последовательно «чинит» токены макроса: `sizeof(x)` → `(x).sizeof`,
  C-касты → `cast(...)`, `->` → `.`, `NULL` → `null`, строковые литералы
  (суффиксы `L`, `ll`, восьмеричные, wide-строки, multi-char), а также
  распознаёт generic cast-макросы и compound literals.

### variable.d

- глобальные переменные → `extern __gshared <type> name;` (+ `export`, `static`);
- `constexpr` переменные → `enum name = init;`;
- переменные с типом «анонимная структура» предварительно объявляют этот тип;
- переменные, чей тип — struct без определения, пропускаются (нельзя объявить
  в D `extern Foo x;` без определения).

### namespace.d

namespace → `extern(C++, "name") { ... }` с рекурсивной обработкой детей; заданные
`--ignore-ns` пропускаются.

### tokens.d

Перевод токенов: `sizeof`/`alignof` в свойства, `< >` → `!( )` (по токенам, чтобы
корректно работать с «непарными» скобками в макросах).

### docs.d

`getComment` извлекает комментарии libclang для вставки их в D-код.

### exception.d

`UntranslatableException` — сигнал того, что конструкция не имеет D-эквивалента.

## Утилиты (source/dpp/clang/package.d)

- `namespace(cursor)` — поиск ближайшего namespace-родителя.
- `typeNameNoNs` — имя типа без namespace-префиксов.
- `isOverride`/`isFinal` — виртуальные функции: наличие атрибутов `override`/`final`
  или переопределение виртуального метода базового класса.
- `baseClasses` — все базовые классы (рекурсивно).
- `hasAnonymousSpelling`, `isSortaAnonymous` — распознавание анонимных сущностей.

## Итоговая схема работы

```
foo.dpp
   │  Options(args)
   ▼
preprocess(foo.dpp → foo.d)
   │  ─ собрать #include-и, module-строки
   │  ─ Context(C/C++)
   │  ─ expand(includes.tmp):
   │        parse через libclang (Трансляционный юнит)
   │        canonicalCursors (дедупликация/слияние, макросы в конец)
   │        для каждого курсора → translate → строки D
   │  ─ fixNames (pragma(mangle), __dpp_aggregate__, поля, необъявленные struct)
   │  ─ module + preamble + переведённый код + исходный D-код
   │  ─ (опц.) прогон через C-препроцессор для разворачивания макросов
   ▼
foo.d
   │  ─ (опц.) C++ boilerplate для линковки stdlib
   ▼
dmd/ldc2 foo.d ...
   ▼
foo.exe
```

## Ключевые особенности архитектуры

- **libclang как единственный парсер** — анализ C/C++ полностью делегирован
  libclang (курсоры, типы, токены); dpp лишь транслирует результаты.
- **Отсутствие глобального состояния** — всё состояние вынесено в `Context`
  (передаётся ref-ом), включая копию опций.
- **Трёхэтапная генерация D**: базовый перевод → пост-фиксы в `fixNames` →
  (опционально) обработка макросов C-препроцессором и финальная компиляция.
- **Компромиссы ради совместимости**: `pragma(mangle)`, `@nogc nothrow` у всех
  функций, `@disable this()`, `dpp.Opaque`, эмуляция move/копирования и т.п.
- **Проверки без компиляции** — вставки вроде
  `static if(is(typeof({ mixin(...); })))` в рантайме D проверяют валидность
  сгенерированного кода.
- **Игнорирование вместо ошибок** — по умолчанию непереводимые сущности молча
  пропускаются; `--hard-fail` включает строгий режим.
```
