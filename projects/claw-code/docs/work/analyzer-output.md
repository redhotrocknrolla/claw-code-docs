# Анализ модуля `rust/crates/tools/src/lib.rs`

## 1. Назначение модуля

Модуль `rust/crates/tools/src/lib.rs` реализует систему управления инструментами для Claw Code — платформы автоматизации. Он предоставляет:

- Регистрацию и управление инструментами (встроенными и плагинными)
- Выполнение инструментов с проверкой разрешений
- Нормализацию имен инструментов и поддержку псевдонимов
- Создание спецификаций инструментов для API
- Управление разрешениями для каждого инструмента

## 2. Публичные сущности

### ToolManifestEntry
- **Вид сущности:** struct
- **Сигнатура:** `pub struct ToolManifestEntry { pub name: String, pub source: ToolSource }`
- **Назначение:** Запись манифеста инструмента с именем и источником
- **Параметры / поля:**
  - `name: String` — имя инструмента
  - `source: ToolSource` — источник инструмента (Base или Conditional)
- **Возвращаемое значение:** —
- **Ошибки / ограничения:** —
- **Связанные элементы:** ToolSource

### ToolSource
- **Вид сущности:** enum
- **Сигнатура:** `pub enum ToolSource { Base, Conditional }`
- **Назначение:** Перечисление типов источников инструментов
- **Параметры / поля:**
  - `Base` — встроенный инструмент
  - `Conditional` — условный инструмент
- **Возвращаемое значение:** —
- **Ошибки / ограничения:** —
- **Связанные элементы:** ToolManifestEntry

### ToolRegistry
- **Вид сущности:** struct
- **Сигнатура:** `pub struct ToolRegistry { entries: Vec<ToolManifestEntry> }`
- **Назначение:** Реестр инструментов для хранения и доступа к записям манифеста
- **Параметры / поля:** `entries: Vec<ToolManifestEntry>` — список записей манифеста
- **Возвращаемое значение:** —
- **Ошибки / ограничения:** —
- **Связанные элементы:** ToolManifestEntry
- **Публичные методы:**
  - `new(entries: Vec<ToolManifestEntry>) -> Self` — создание реестра
  - `entries(&self) -> &[ToolManifestEntry]` — получение записей

### ToolSpec
- **Вид сущности:** struct
- **Сигнатура:** `pub struct ToolSpec { pub name: &'static str, pub description: &'static str, pub input_schema: Value, pub required_permission: PermissionMode }`
- **Назначение:** Спецификация инструмента с метаданными
- **Параметры / поля:**
  - `name: &'static str` — статическое имя инструмента
  - `description: &'static str` — описание инструмента
  - `input_schema: Value` — JSON-схема входных параметров
  - `required_permission: PermissionMode` — требуемый уровень разрешений
- **Возвращаемое значение:** —
- **Ошибки / ограничения:** —
- **Связанные элементы:** PermissionMode, Value (serde_json)

### GlobalToolRegistry
- **Вид сущности:** struct
- **Сигнатура:** `pub struct GlobalToolRegistry { plugin_tools: Vec<PluginTool> }`
- **Назначение:** Глобальный реестр инструментов, объединяющий встроенные и плагинные инструменты
- **Параметры / поля:** `plugin_tools: Vec<PluginTool>` — список плагинных инструментов
- **Возвращаемое значение:** —
- **Ошибки / ограничения:** —
- **Связанные элементы:** PluginTool
- **Публичные методы:**
  - `builtin() -> Self` — создание реестра с встроенными инструментами
  - `with_plugin_tools(plugin_tools: Vec<PluginTool>) -> Result<Self, String>` — создание с плагинными инструментами (проверка дубликатов)
  - `normalize_allowed_tools(values: &[String]) -> Result<Option<BTreeSet<String>>, String>` — нормализация разрешенных инструментов
  - `definitions(allowed_tools: Option<&BTreeSet<String>>) -> Vec<ToolDefinition>` — получение спецификаций
  - `permission_specs(allowed_tools: Option<&BTreeSet<String>>) -> Vec<(String, PermissionMode)>` — получение спецификаций разрешений
  - `execute(name: &str, input: &Value) -> Result<String, String>` — выполнение инструмента

### mvp_tool_specs()
- **Вид сущности:** fn
- **Сигнатура:** `pub fn mvp_tool_specs() -> Vec<ToolSpec>`
- **Назначение:** Возвращает список встроенных спецификаций инструментов (MVP — minimum viable product)
- **Параметры:** —
- **Возвращаемое значение:** `Vec<ToolSpec>` — список спецификаций
- **Ошибки / ограничения:** —
- **Связанные элементы:** ToolSpec
- **Примечание:** Возвращает 19 встроенных инструментов (bash, read_file, write_file, edit_file, glob_search, grep_search, WebFetch, WebSearch, TodoWrite, Skill, Agent, ToolSearch, NotebookEdit, Sleep, SendUserMessage, Config, StructuredOutput, REPL, PowerShell)

### execute_tool()
- **Вид сущности:** fn
- **Сигнатура:** `pub fn execute_tool(name: &str, input: &Value) -> Result<String, String>`
- **Назначение:** Выполняет инструмент по имени с предоставленными входными данными
- **Параметры:**
  - `name: &str` — имя инструмента
  - `input: &Value` — входные данные в формате JSON
- **Возвращаемое значение:** `Result<String, String>` — результат в формате JSON или ошибка
- **Ошибки / ограничения:** Возвращает ошибку "unsupported tool" для неизвестных инструментов
- **Связанные элементы:** execute_bash, read_file, write_file, edit_file, glob_search, grep_search, и другие

## 3. Непубличные, но значимые элементы

### Нормализация имен
- Функция `normalize_tool_name(value: &str) -> String` — преобразует имя инструмента (trim, replace('-', '_'), to_ascii_lowercase)
- Используется для поддержки псевдонимов (read, write, edit, glob, grep)

### Сопоставление разрешений
- Функция `permission_mode_from_plugin(value: &str) -> PermissionMode` — преобразует строковое представление разрешения в PermissionMode
- Поддерживает: "read-only", "workspace-write", "danger-full-access"
- Паникует при неподдерживаемом значении

### Выполнение инструментов
- `execute_bash`, `execute_web_fetch`, `execute_web_search`, `execute_todo_write`, `execute_skill`, `execute_agent`, `execute_tool_search`, `execute_notebook_edit`, `execute_sleep`, `execute_brief`, `execute_config`, `execute_structured_output`, `execute_repl`, `execute_powershell` — все функции вызываются через execute_tool

### Вспомогательные функции
- `to_pretty_json<T: serde::Serialize>(value: T) -> Result<String, String>` — красивая сериализация JSON
- `io_to_string(error: std::io::Error) -> String` — преобразование ошибок IO в строки
- `from_value<T: for<'de> Deserialize<'de>>(input: &Value) -> Result<T, String>` — десериализация JSON

### Структуры ввода (Input Structures)
- `ReadFileInput`, `WriteFileInput`, `EditFileInput`, `GlobSearchInputValue`, `WebFetchInput`, `WebSearchInput`, `TodoWriteInput`, `SkillInput`, `AgentInput`, `ToolSearchInput`, `NotebookEditInput`, `SleepInput`, `BriefInput`, `ConfigInput`, `StructuredOutputInput`, `ReplInput`, `PowerShellInput`
- Все имплементируют `Deserialize` для JSON-десериализации

## 4. Сценарии работы

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

## 5. Места повышенного риска интерпретации

### Требует уточнения по соседним модулям
- `PermissionMode` — тип разрешений (импортирован из runtime)
- `PermissionPolicy` — политика разрешений (импортирован из runtime)
- `PluginTool` — тип плагинного инструмента (импортирован из plugins)
- `ToolDefinition` — определение инструмента API (импортирован из api)
- `ProviderClient`, `ApiClient`, `ConversationRuntime` — типы для API-взаимодействия (импортированы из runtime/api/runtime)
- `TodoItem`, `TodoStatus` — структуры для списка задач (импортированы из runtime)
- `ContentBlock`, `ConversationMessage`, `Session` — типы для сессий (импортированы из runtime)

## 6. Заготовка структуры документации
