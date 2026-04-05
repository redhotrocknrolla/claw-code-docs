# Модуль tools: Система управления инструментами для Claw Code

**Дата создания документа:** 4 апреля 2026 г.  
**Модуль:** `/home/work/22245342@sigma.sbrf.ru/rust/crates/tools/src/lib.rs`  
**Версия модуля:** —  
**Язык:** Rust  
**Статус:** Финальная документация

---

## 1. Назначение модуля

Модуль `rust/crates/tools/src/lib.rs` реализует систему управления инструментами для Claw Code — платформы автоматизации. Он предоставляет абстракции для регистрации, выполнения и управления встроенными и плагинными инструментами.

Основные функции:
- Реестр инструментов с проверкой конфликтов имен
- Выполнение инструментов с соблюдением разрешений
- Нормализация имен инструментов и поддержка псевдонимов
- Создание спецификаций для API

---

## 2. Основные сущности

### 2.1 ToolManifestEntry

Запись манифеста инструмента с именем и источником.

**Сигнатура:**
```rust
pub struct ToolManifestEntry {
    pub name: String,
    pub source: ToolSource,
}
```

**Поля:**
| Имя | Тип | Описание |
|-----|-----|----------|
| `name` | `String` | Имя инструмента |
| `source` | `ToolSource` | Источник инструмента (Base или Conditional) |

**Связанные типы:** ToolSource

**Ограничения:** —

---

### 2.2 ToolSource

Перечисление типов источников инструментов.

**Сигнатура:**
```rust
pub enum ToolSource {
    Base,
    Conditional,
}
```

**Варианты:**
| Вариант | Описание |
|---------|----------|
| `Base` | Встроенный инструмент |
| `Conditional` | Условный инструмент |

**Связанные типы:** ToolManifestEntry

**Ограничения:** —

---

### 2.3 ToolRegistry

Реестр инструментов для хранения и доступа к записям манифеста.

**Сигнатура:**
```rust
pub struct ToolRegistry {
    entries: Vec<ToolManifestEntry>,
}
```

**Поля:**
| Имя | Тип | Описание |
|-----|-----|----------|
| `entries` | `Vec<ToolManifestEntry>` | Список записей манифеста |

**Публичные методы:**
| Метод | Возвращаемое значение | Описание |
|-------|----------------------|----------|
| `new(entries: Vec<ToolManifestEntry>) -> Self` | `ToolRegistry` | Создание реестра |
| `entries(&self) -> &[ToolManifestEntry]` | `&[ToolManifestEntry]` | Получение записей |

**Связанные типы:** ToolManifestEntry

**Ограничения:** —

---

### 2.4 ToolSpec

Спецификация инструмента с метаданными.

**Сигнатура:**
```rust
pub struct ToolSpec {
    pub name: &'static str,
    pub description: &'static str,
    pub input_schema: Value,
    pub required_permission: PermissionMode,
}
```

**Поля:**
| Имя | Тип | Описание |
|-----|-----|----------|
| `name` | `&'static str` | Статическое имя инструмента |
| `description` | `&'static str` | Описание инструмента |
| `input_schema` | `Value` | JSON-схема входных параметров |
| `required_permission` | `PermissionMode` | Требуемый уровень разрешений |

**Связанные типы:** PermissionMode, Value (serde_json)

**Ограничения:**
- `name` и `description` являются статическими строками, что требует постоянных строковых литералов

---

### 2.5 GlobalToolRegistry

Глобальный реестр инструментов, объединяющий встроенные и плагинные инструменты.

**Сигнатура:**
```rust
pub struct GlobalToolRegistry {
    plugin_tools: Vec<PluginTool>,
}
```

**Поля:**
| Имя | Тип | Описание |
|-----|-----|----------|
| `plugin_tools` | `Vec<PluginTool>` | Список плагинных инструментов |

**Публичные методы:**
| Метод | Возвращаемое значение | Описание |
|-------|----------------------|----------|
| `builtin() -> Self` | `GlobalToolRegistry` | Создание реестра с встроенными инструментами |
| `with_plugin_tools(plugin_tools: Vec<PluginTool>) -> Result<Self, String>` | `Result<GlobalToolRegistry, String>` | Создание с плагинными инструментами (проверка дубликатов) |
| `normalize_allowed_tools(values: &[String]) -> Result<Option<BTreeSet<String>>, String>` | `Result<Option<BTreeSet<String>>, String>` | Нормализация разрешенных инструментов |
| `definitions(allowed_tools: Option<&BTreeSet<String>>) -> Vec<ToolDefinition>` | `Vec<ToolDefinition>` | Получение спецификаций |
| `permission_specs(allowed_tools: Option<&BTreeSet<String>>) -> Vec<(String, PermissionMode)>` | `Vec<(String, PermissionMode)>` | Получение спецификаций разрешений |
| `execute(name: &str, input: &Value) -> Result<String, String>` | `Result<String, String>` | Выполнение инструмента |

**Связанные типы:** PluginTool

**Ограничения:**
- При создании через `with_plugin_tools()` проверяются дубликаты имен между встроенными и плагинными инструментами

---

### 2.6 mvp_tool_specs()

Возвращает список встроенных спецификаций инструментов (MVP — minimum viable product).

**Сигнатура:**
```rust
pub fn mvp_tool_specs() -> Vec<ToolSpec>
```

**Параметры:** —

**Возвращаемое значение:** `Vec<ToolSpec>` — список спецификаций

**Описание:** Возвращает 19 встроенных инструментов:
1. `bash` — выполнение shell-команды
2. `read_file` — чтение текстового файла
3. `write_file` — запись текстового файла
4. `edit_file` — замена текста в файле
5. `glob_search` — поиск файлов по шаблону
6. `grep_search` — поиск содержимого файлов по регулярному выражению
7. `WebFetch` — загрузка веб-страницы
8. `WebSearch` — поиск в интернете
9. `TodoWrite` — управление списком задач
10. `Skill` — вызов скилла
11. `Agent` — вызов агента
12. `ToolSearch` — поиск инструментов
13. `NotebookEdit` — редактирование Jupyter-ноутбука
14. `Sleep` — пауза выполнения
15. `SendUserMessage` — отправка сообщения пользователю
16. `Config` — конфигурация
17. `StructuredOutput` — структурированный вывод
18. `REPL` — интерактивная оболочка
19. `PowerShell` — выполнение PowerShell-команды

**Ограничения:** —

---

### 2.7 execute_tool()

Выполняет инструмент по имени с предоставленными входными данными.

**Сигнатура:**
```rust
pub fn execute_tool(name: &str, input: &Value) -> Result<String, String>
```

**Параметры:**
| Имя | Тип | Описание |
|-----|-----|----------|
| `name` | `&str` | Имя инструмента |
| `input` | `&Value` | Входные данные в формате JSON |

**Возвращаемое значение:** `Result<String, String>` — результат в формате JSON или ошибка

**Описание:** Функция выполняет инструмент, определяя его тип по имени и вызывая соответствующую функцию execute_*().

**Ограничения:**
- Возвращает ошибку "unsupported tool" для неизвестных инструментов
- Требует, чтобы инструмент существовал в `mvp_tool_specs()`

**Связанные элементы:** execute_bash, read_file, write_file, edit_file, glob_search, grep_search, и другие

---

## 3. Ключевые сценарии работы

### 3.1 Регистрация инструментов

**Сценарий:** Создание глобального реестра инструментов с встроенными и плагинными компонентами.

**Шаги:**
1. Создание реестра через `GlobalToolRegistry::builtin()` для встроенных инструментов
2. Или использование `GlobalToolRegistry::with_plugin_tools(plugin_tools)` с плагинными инструментами
3. Проверка дубликатов имен между встроенными и плагинными инструментами
4. Использование `definitions()` для получения спецификаций API

**Пример:**
```rust
let registry = GlobalToolRegistry::builtin();
let specs = registry.definitions(None);
```

---

### 3.2 Выполнение инструментов

**Сценарий:** Выполнение инструмента с валидацией разрешений.

**Шаги:**
1. Проверка имени инструмента в `mvp_tool_specs()`
2. Десериализация входных данных через `from_value()`
3. Вызов соответствующей функции execute_*()
4. Сериализация результата в JSON через `to_pretty_json()`

**Пример:**
```rust
let result = execute_tool("read_file", &json!({
    "path": "/example/file.txt"
}));
```

---

### 3.3 Управление разрешениями

**Сценарий:** Ограничение доступа к инструментам через разрешения.

**Шаги:**
1. `normalize_allowed_tools()` обрабатывает список разрешенных инструментов
2. Поддерживает псевдонимы: read→read_file, write→write_file, edit→edit_file, glob→glob_search, grep→grep_search
3. `permission_specs()` возвращает соответствующие уровни разрешений

**Пример:**
```rust
let allowed = registry.normalize_allowed_tools(&["read".to_string(), "write".to_string()])?;
let permissions = registry.permission_specs(allowed.as_ref());
```

---

## 4. Ограничения модуля

### 4.1 Внешние зависимости

Модуль зависит от типов из других модулей, которые требуют уточнения:
- `PermissionMode` — тип разрешений (импортирован из runtime)
- `PermissionPolicy` — политика разрешений (импортирован из runtime)
- `PluginTool` — тип плагинного инструмента (импортирован из plugins)
- `ToolDefinition` — определение инструмента API (импортирован из api)
- `ProviderClient`, `ApiClient`, `ConversationRuntime` — типы для API-взаимодействия (импортированы из runtime/api/runtime)
- `TodoItem`, `TodoStatus` — структуры для списка задач (импортированы из runtime)
- `ContentBlock`, `ConversationMessage`, `Session` — типы для сессий (импортированы из runtime)

### 4.2 Статические имена

Имена инструментов в ToolSpec являются `&'static str`, что требует статических строк и не позволяет динамически генерировать имена во время выполнения.

### 4.3 Обработка ошибок

Функция `permission_mode_from_plugin` паникует при неподдерживаемом значении строки разрешения, что может привести к аварийному завершению программы.

---

## 5. Примечания по использованию

### 5.1 Создание реестра

Для создания реестра с плагинными инструментами используйте `with_plugin_tools()`, который проверяет конфликты имен. В случае дубликата возвращается ошибка.

### 5.2 Обработка пользовательского ввода

Используйте `normalize_allowed_tools()` для обработки пользовательского ввода разрешенных инструментов. Функция поддерживает псевдонимы и возвращает нормализованные имена.

### 5.3 Выполнение инструментов

Используйте `execute()` для выполнения инструмента, который автоматически определяет тип инструмента (встроенный или плагинный) и вызывает соответствующую функцию.

### 5.4 Уровни разрешений

Типы разрешений определяются внешним модулем runtime. Используйте `permission_specs()` для получения соответствующих уровней разрешений для каждого инструмента.

---

## 6. Неочевидные места

### 6.1 Требует уточнения по соседним модулям

| Элемент | Тип | Источник |
|---------|-----|----------|
| `PermissionMode` | enum | runtime |
| `PermissionPolicy` | enum | runtime |
| `PluginTool` | struct | plugins |
| `ToolDefinition` | struct | api |
| `ProviderClient` | struct | runtime/api/runtime |
| `ApiClient` | struct | runtime/api/runtime |
| `ConversationRuntime` | struct | runtime/api/runtime |
| `TodoItem` | struct | runtime |
| `TodoStatus` | enum | runtime |
| `ContentBlock` | struct | runtime |
| `ConversationMessage` | struct | runtime |
| `Session` | struct | runtime |

---

**Конец документа**
