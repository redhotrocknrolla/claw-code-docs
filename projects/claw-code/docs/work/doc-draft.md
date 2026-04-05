# Модуль tools: Система управления инструментами для Claw Code

## Назначение модуля

Модуль `rust/crates/tools/src/lib.rs` реализует систему управления инструментами для Claw Code — платформы автоматизации. Он предоставляет абстракции для регистрации, выполнения и управления встроенными и плагинными инструментами.

Основные функции:
- Реестр инструментов с проверкой конфликтов имен
- Выполнение инструментов с соблюдением разрешений
- Нормализация имен инструментов и поддержка псевдонимов
- Создание спецификаций для API

## Основные сущности

### ToolManifestEntry

Запись манифеста инструмента с именем и источником.

**Поля:**
- `name: String` — имя инструмента
- `source: ToolSource` — источник инструмента (Base или Conditional)

**Связанные типы:** ToolSource

### ToolSource

Перечисление типов источников инструментов.

**Варианты:**
- `Base` — встроенный инструмент
- `Conditional` — условный инструмент

**Связанные типы:** ToolManifestEntry

### ToolRegistry

Реестр инструментов для хранения и доступа к записям манифеста.

**Поля:**
- `entries: Vec<ToolManifestEntry>` — список записей манифеста

**Публичные методы:**
- `new(entries: Vec<ToolManifestEntry>) -> Self` — создание реестра
- `entries(&self) -> &[ToolManifestEntry]` — получение записей

**Связанные типы:** ToolManifestEntry

### ToolSpec

Спецификация инструмента с метаданными.

**Поля:**
- `name: &'static str` — статическое имя инструмента
- `description: &'static str` — описание инструмента
- `input_schema: Value` — JSON-схема входных параметров
- `required_permission: PermissionMode` — требуемый уровень разрешений

**Связанные типы:** PermissionMode, Value (serde_json)

### GlobalToolRegistry

Глобальный реестр инструментов, объединяющий встроенные и плагинные инструменты.

**Поля:**
- `plugin_tools: Vec<PluginTool>` — список плагинных инструментов

**Публичные методы:**
- `builtin() -> Self` — создание реестра с встроенными инструментами
- `with_plugin_tools(plugin_tools: Vec<PluginTool>) -> Result<Self, String>` — создание с плагинными инструментами (проверка дубликатов)
- `normalize_allowed_tools(values: &[String]) -> Result<Option<BTreeSet<String>>, String>` — нормализация разрешенных инструментов
- `definitions(allowed_tools: Option<&BTreeSet<String>>) -> Vec<ToolDefinition>` — получение спецификаций
- `permission_specs(allowed_tools: Option<&BTreeSet<String>>) -> Vec<(String, PermissionMode)>` — получение спецификаций разрешений
- `execute(name: &str, input: &Value) -> Result<String, String>` — выполнение инструмента

**Связанные типы:** PluginTool

### mvp_tool_specs()

Возвращает список встроенных спецификаций инструментов (MVP — minimum viable product).

**Параметры:** —

**Возвращаемое значение:** `Vec<ToolSpec>` — список спецификаций

**Примечание:** Возвращает 19 встроенных инструментов (bash, read_file, write_file, edit_file, glob_search, grep_search, WebFetch, WebSearch, TodoWrite, Skill, Agent, ToolSearch, NotebookEdit, Sleep, SendUserMessage, Config, StructuredOutput, REPL, PowerShell)

### execute_tool()

Выполняет инструмент по имени с предоставленными входными данными.

**Параметры:**
- `name: &str` — имя инструмента
- `input: &Value` — входные данные в формате JSON

**Возвращаемое значение:** `Result<String, String>` — результат в формате JSON или ошибка

**Ограничения:** Возвращает ошибку "unsupported tool" для неизвестных инструментов

**Связанные элементы:** execute_bash, read_file, write_file, edit_file, glob_search, grep_search, и другие

## Ключевые сценарии работы

### Регистрация инструментов

1. Создание GlobalToolRegistry через `builtin()` или `with_plugin_tools()`
2. Проверка дубликатов имен между встроенными и плагинными инструментами
3. Использование `definitions()` для получения спецификаций API

### Выполнение инструментов

1. Проверка имени инструмента в `mvp_tool_specs()`
2. Десериализация входных данных через `from_value()`
3. Вызов соответствующей функции execute_*()
4. Сериализация результата в JSON через `to_pretty_json()`

### Управление разрешениями

1. `normalize_allowed_tools()` обрабатывает список разрешенных инструментов
2. Поддерживает псевдонимы: read→read_file, write→write_file, edit→edit_file, glob→glob_search, grep→grep_search
3. `permission_specs()` возвращает соответствующие уровни разрешений

## Ограничения модуля

1. **Внешние зависимости:** Модуль зависит от типов из других модулей (runtime, api, plugins)
2. **Статические имена:** Имена инструментов в ToolSpec являются `&'static str`, что требует статических строк
3. **Паника при ошибках:** Функция `permission_mode_from_plugin` паникует при неподдерживаемом значении

## Примечания по использованию

1. Для создания реестра с плагинными инструментами используйте `with_plugin_tools()`, который проверяет конфликты имен
2. Используйте `normalize_allowed_tools()` для обработки пользовательского ввода разрешенных инструментов
3. Для выполнения инструмента используйте `execute()`, который автоматически определяет тип инструмента
4. Типы разрешений определяются внешним модулем runtime

## Неочевидные места

### Требует уточнения по соседним модулям

- `PermissionMode` — тип разрешений (импортирован из runtime)
- `PermissionPolicy` — политика разрешений (импортирован из runtime)
- `PluginTool` — тип плагинного инструмента (импортирован из plugins)
- `ToolDefinition` — определение инструмента API (импортирован из api)
- `ProviderClient`, `ApiClient`, `ConversationRuntime` — типы для API-взаимодействия (импортированы из runtime/api/runtime)
- `TodoItem`, `TodoStatus` — структуры для списка задач (импортированы из runtime)
- `ContentBlock`, `ConversationMessage`, `Session` — типы для сессий (импортированы из runtime)
