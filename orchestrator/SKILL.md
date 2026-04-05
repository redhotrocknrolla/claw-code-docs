---
name: orchestrator
description: >
  Координатор пайплайна документирования. Определяет язык, выбирает набор skill,
  управляет последовательностью шагов, проверяет артефакты.
  Используй для запуска полного цикла документирования по любому репозиторию или файлу.
  Триггеры: документировать, сгенерировать документацию, запустить пайплайн, orchestrator
---

# Orchestrator

## Правило №1 — главное

Выполняй шаги строго по порядку. Не переходи к следующему шагу, пока не создан артефакт текущего.
Если что-то неясно — задай пользователю один вопрос. Только один.

---

## Шаг 0. Определи входные данные

**Что тебе передали?**

→ Если Git URL (`https://...` или `git@...`):
  1. Запусти `git-repo-scanner`
  2. Дождись `docs/work/repo-overview.md`
  3. Используй его для определения языка и целевого модуля
  4. Зафиксируй commit hash

→ Если локальный путь к **файлу** (`.java`, `.rs`, `.py` и т.д.):
  - Пропусти git-repo-scanner
  - Перейди сразу к Шагу 1

→ Если локальный путь к **директории**:
  1. Запусти `git-repo-scanner` с этим путём
  2. Дождись `docs/work/repo-overview.md`
  3. Спроси пользователя: «Какой файл документировать?» (покажи список из repo-overview)

→ Если неясно что передано:
  - Задай пользователю: «Это URL репозитория или путь к файлу?»

---

## Шаг 1. Определи имя проекта

**Правило:**

| Источник | Имя проекта |
|----------|------------|
| `https://github.com/org/my-project` | `my-project` |
| `/home/user/repos/my-project/` | `my-project` |
| `/home/user/repos/my-project/src/Foo.java` | `my-project` |
| Один файл без явной структуры | имя файла без расширения |

Запиши имя проекта. Используй его везде далее.

---

## Шаг 2. Определи язык и выбери skill-набор

Посмотри на расширение целевого файла:

| Расширение | Язык | Набор skill |
|-----------|------|------------|
| `.java` | Java | `java-module-analyzer` → `java-doc-writer` → `java-style-auditor` |
| `.rs` | Rust | `rust-module-analyzer` → `rust-doc-writer` → `rust-style-auditor` |
| `.py` | Python | `universal-module-analyzer` → `universal-doc-writer` → `universal-style-auditor` |
| `.ts`, `.tsx` | TypeScript | `universal-module-analyzer` → `universal-doc-writer` → `universal-style-auditor` |
| `.go` | Go | `universal-module-analyzer` → `universal-doc-writer` → `universal-style-auditor` |
| всё остальное | — | `universal-module-analyzer` → `universal-doc-writer` → `universal-style-auditor` |

Если файл указан без расширения — используй `universal-*`.

Запиши выбранный набор. Проверь, что все три skill доступны.
Если какой-то skill недоступен — замени весь набор на `universal-*` и зафиксируй это.

---

## Шаг 3. Создай структуру папок

Создай следующие директории:

```
projects/<project-name>/docs/work/
projects/<project-name>/docs/outputs/
projects/<project-name>/docs/dialogs/
projects/<project-name>/docs/notes/
projects/<project-name>/reports/
```

Создай журнал оркестратора `projects/<project-name>/docs/dialogs/orchestrator-session.md`:

```markdown
# Orchestrator Session

## Параметры запуска
- Источник: <URL или путь>
- Имя проекта: <имя>
- Целевой файл: <путь>
- Язык: <язык>
- Набор skill: <список>
- Commit hash: <hash или «не применимо»>
- Дата: <дата>

## Лог шагов
### Шаг 0
- Действие: <что сделано>
- Результат: успех / пропущен

### Шаг 1 (Анализ)
- Статус: ожидание / успех / ошибка
- Артефакт: ...

### Шаг 2 (Writer)
- Статус: ...

### Шаг 3 (Аудит)
- Статус: ...

### Шаг 4 (Confluence)
- Статус: ...

### Шаг 5 (Report)
- Статус: ...
```

---

## Шаг 4 (Анализ). Запусти module-analyzer

Запусти: `<язык>-module-analyzer` или `universal-module-analyzer`

Дождись файлов:
```
projects/<project-name>/docs/work/analyzer-output.md     ← ОБЯЗАТЕЛЬНО
projects/<project-name>/docs/dialogs/analyzer-session.md ← ОБЯЗАТЕЛЬНО
```

**Проверка перед продолжением:**
- Файл `analyzer-output.md` создан? → если НЕТ: повтори шаг
- Раздел «Публичные сущности» заполнен? → если НЕТ: повтори шаг
- Есть хотя бы одна сущность? → если НЕТ: зафиксируй «модуль не содержит публичных сущностей» и продолжай

Обнови лог оркестратора: Шаг 1 — успех / ошибка.

---

## Шаг 5 (Writer). Запусти doc-writer

Запусти: `<язык>-doc-writer` или `universal-doc-writer`

Дождись файлов:
```
projects/<project-name>/docs/work/doc-draft.md           ← ОБЯЗАТЕЛЬНО
projects/<project-name>/docs/dialogs/writer-session.md   ← ОБЯЗАТЕЛЬНО
```

**Проверка перед продолжением:**
- Файл `doc-draft.md` создан? → если НЕТ: повтори шаг
- Раздел «Основные сущности» заполнен? → если НЕТ: повтори шаг

Обнови лог оркестратора: Шаг 2 — успех / ошибка.

---

## Шаг 6 (Аудит). Запусти style-auditor

Запусти: `<язык>-style-auditor` или `universal-style-auditor`

Дождись файлов:
```
projects/<project-name>/docs/outputs/<project-name>-doc-advanced.md  ← ОБЯЗАТЕЛЬНО
projects/<project-name>/docs/dialogs/auditor-session.md              ← ОБЯЗАТЕЛЬНО
projects/<project-name>/docs/notes/advanced-note.md                  ← ОБЯЗАТЕЛЬНО
```

**Проверка перед продолжением:**
- Файл `*-doc-advanced.md` создан? → если НЕТ: повтори шаг
- Файл не пустой? → если пустой: повтори шаг

Обнови лог оркестратора: Шаг 3 — успех / ошибка.

---

## Шаг 7 (Confluence). Публикация — только если запрошено

→ Если пользователь просил опубликовать в Confluence:
  1. Запусти `confluence-publisher`
  2. Дождись `docs/dialogs/publisher-session.md`
  3. Зафиксируй URL страницы в журнале

→ Если не просил:
  - Пропусти этот шаг
  - Запиши в лог: «Шаг 4 — пропущен»

---

## Шаг 8 (Отчёт). Создай итоговый отчёт

Создай файл `projects/<project-name>/reports/<project-name>-report-YYYY-MM-DD.md`:

```markdown
# Отчёт: <project-name>
**Дата:** <дата>

## Параметры
- Источник: <URL или путь>
- Commit: <hash или «не применимо»>
- Язык: <язык>
- Набор skill: <список>

## Созданные артефакты
| Файл | Статус |
|------|--------|
| docs/work/analyzer-output.md | создан |
| docs/work/doc-draft.md | создан |
| docs/outputs/<project-name>-doc-advanced.md | создан |
| docs/dialogs/orchestrator-session.md | создан |
| docs/dialogs/analyzer-session.md | создан |
| docs/dialogs/writer-session.md | создан |
| docs/dialogs/auditor-session.md | создан |
| docs/notes/advanced-note.md | создан |

## Финальный документ
`docs/outputs/<project-name>-doc-advanced.md`

## Зоны неопределённости
<скопировать из advanced-note.md>

## Confluence
<URL страницы или «не публиковалось»>
```

---

## Шаг 9. Финальная проверка

Проверь наличие всех обязательных файлов:

```
[ ] projects/<project-name>/docs/work/analyzer-output.md
[ ] projects/<project-name>/docs/work/doc-draft.md
[ ] projects/<project-name>/docs/outputs/<project-name>-doc-advanced.md
[ ] projects/<project-name>/docs/dialogs/orchestrator-session.md
[ ] projects/<project-name>/docs/dialogs/analyzer-session.md
[ ] projects/<project-name>/docs/dialogs/writer-session.md
[ ] projects/<project-name>/docs/dialogs/auditor-session.md
[ ] projects/<project-name>/docs/notes/advanced-note.md
[ ] projects/<project-name>/reports/<project-name>-report-YYYY-MM-DD.md
```

Если хотя бы один файл отсутствует — пайплайн **НЕ завершён**.
Сообщи пользователю какой файл отсутствует и повтори соответствующий шаг.

Если все файлы на месте — сообщи:
```
Документирование завершено.
Финальный документ: projects/<project-name>/docs/outputs/<project-name>-doc-advanced.md
Отчёт: projects/<project-name>/reports/<project-name>-report-YYYY-MM-DD.md
```

---

## Опциональные артефакты

```
projects/<project-name>/docs/work/repo-overview.md         ← если был git-repo-scanner
projects/<project-name>/docs/dialogs/scanner-session.md    ← если был git-repo-scanner
projects/<project-name>/docs/dialogs/publisher-session.md  ← если была публикация в Confluence
```
