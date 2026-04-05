# Documentation Pipeline — Multi-Agent Skill System

Мультиагентный пайплайн для автоматической генерации технической документации по исходному коду. Работает через систему skill для GigaCode CLI.

## Как это работает

Пайплайн состоит из специализированных агентов (skill), каждый из которых отвечает за свой этап. Orchestrator управляет последовательностью, выбирает нужный набор skill по языку и контролирует качество на каждом шаге.

```
Git URL / локальный путь
        │
        ▼
┌─────────────────────┐
│  git-repo-scanner   │  Шаг 0 — сканирование репо (опционально)
│  структура + история│  → repo-overview.md
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  orchestrator       │  Определяет язык, имя проекта, набор skill
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  *-module-analyzer  │  Шаг 1 — извлечение фактов из кода
│                     │  → analyzer-output.md
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  *-doc-writer       │  Шаг 2 — генерация черновика документации
│                     │  → doc-draft.md
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  *-style-auditor    │  Шаг 3 — аудит, стиль, верификация
│                     │  → *-doc-advanced.md (финальный)
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ confluence-publisher │  Шаг 4 — публикация в Confluence (опционально)
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  report             │  Шаг 5 — итоговый отчёт
│                     │  → *-report-YYYY-MM-DD.md
└─────────────────────┘
```

## Поддерживаемые языки

| Язык | Набор skill | Статус |
|------|------------|--------|
| Rust | `rust-module-analyzer` / `rust-doc-writer` / `rust-style-auditor` | Готов |
| Java | `java-module-analyzer` / `java-doc-writer` / `java-style-auditor` | Готов |
| Любой другой | `universal-module-analyzer` / `universal-doc-writer` / `universal-style-auditor` | Готов |
| Python | `python-*` | В планах |
| TypeScript | `ts-*` | В планах |
| Go | `go-*` | В планах |

Если языко-специфичный набор не найден — автоматически используется `universal-*`.

## Структура репозитория

```
.gigacode/skills/
├── README.md                          ← этот файл
├── orchestrator/SKILL.md              ← координатор пайплайна
├── git-repo-scanner/SKILL.md          ← сканер репозиториев
├── confluence-publisher/SKILL.md      ← публикация в Confluence
│
├── rust-module-analyzer/SKILL.md      ← Rust: анализатор
├── rust-doc-writer/SKILL.md           ← Rust: писатель
├── rust-style-auditor/SKILL.md        ← Rust: аудитор
│
├── java-module-analyzer/SKILL.md      ← Java: анализатор
├── java-doc-writer/SKILL.md           ← Java: писатель
├── java-style-auditor/SKILL.md        ← Java: аудитор
│
├── universal-module-analyzer/SKILL.md ← Universal: анализатор
├── universal-doc-writer/SKILL.md      ← Universal: писатель
├── universal-style-auditor/SKILL.md   ← Universal: аудитор
│
├── challenge-report.md                ← отчёт VibeCoding Challenge #2
│
└── projects/
    └── claw-code/                     ← пример проекта
        ├── docs/
        │   ├── work/                  ← промежуточные артефакты
        │   ├── outputs/               ← финальная документация
        │   ├── dialogs/               ← журналы агентов
        │   └── notes/                 ← пояснительные записки
        └── reports/
            └── claw-code-report-2026-04-05.md
```

## Структура проекта (Variant B)

Каждый документируемый репозиторий получает изолированную директорию:

```
projects/<project-name>/
├── docs/
│   ├── work/           ← analyzer-output.md, doc-draft.md, repo-overview.md
│   ├── outputs/        ← <project-name>-doc-advanced.md
│   ├── dialogs/        ← журналы: orchestrator, analyzer, writer, auditor, scanner, publisher
│   └── notes/          ← advanced-note.md
└── reports/
    └── <project-name>-report-YYYY-MM-DD.md
```

## Ключевые принципы

**Анти-галлюцинация.** Каждый агент работает строго по коду. Если факт нельзя подтвердить — ставится пометка `Требует уточнения по соседним модулям`. Никаких домыслов.

**Двойная проверка.** Analyzer делает Primary Pass + Expert Review Pass. Writer делает генерацию + Fresh Eyes Review. Auditor верифицирует всё по коду повторно.

**Quality Gates.** Каждый скилл содержит чеклист, который должен быть пройден перед передачей артефакта на следующий шаг.

**Трассируемость.** Все действия логируются в `dialogs/`. Каждый шаг фиксирует входные данные, принятые решения, ограничения и созданные файлы.

## Скиллы-утилиты

| Скилл | Назначение |
|-------|-----------|
| `git-repo-scanner` | Сканирование структуры и истории Git-репозитория, определение языков, hotspots, changelog |
| `confluence-publisher` | Публикация документации в Confluence Server/Data Center (MCP или REST API) |

## Быстрый старт

1. Скопировать `.gigacode/skills/` в корень своего проекта или рабочей директории GigaCode.
2. Вызвать orchestrator, передав URL или путь к репозиторию.
3. Orchestrator автоматически определит язык, создаст структуру `projects/<name>/` и проведёт документирование.

## Лицензия

MIT
