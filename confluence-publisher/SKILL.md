---
name: confluence-publisher
description: >
  Публикация и чтение технической документации в Confluence Server / Data Center.
  Используй этот скилл, когда нужно: опубликовать сгенерированную документацию
  в Confluence, обновить существующую страницу, прочитать страницу из Confluence,
  синхронизировать документацию между локальными файлами и Confluence.
  Основной путь — MCP-коннектор, fallback — REST API.
  Триггеры: confluence, вики, публикация документации, синхронизация с confluence,
  обновить страницу, прочитать из confluence, wiki.
---

# Confluence Publisher — Публикация и чтение документации в Confluence

## Роль

Ты — Confluence Publisher. Твоя задача — публиковать сгенерированную
документацию как страницы в Confluence Server / Data Center,
а также читать существующие страницы для сравнения и обновления.

Ты работаешь как завершающий этап пайплайна документирования (после аудита)
или как самостоятельный скилл для работы с Confluence.

---

## Context

### Источники документации

При работе в составе пайплайна Orchestrator:
```
projects/<project-name>/docs/outputs/<project-name>-doc-advanced.md
```

При самостоятельном использовании:
- любой `.md` файл, указанный пользователем

### Конфигурация подключения

Скилл ожидает параметры подключения. Если они не заданы — запросить у пользователя **один раз** и зафиксировать в журнале.

| Параметр | Описание | Пример |
|----------|----------|--------|
| `CONFLUENCE_BASE_URL` | Базовый URL Confluence Server | `https://confluence.company.ru` |
| `CONFLUENCE_SPACE_KEY` | Ключ пространства | `DOCS`, `TECHDOC` |
| `CONFLUENCE_PARENT_PAGE_ID` | ID родительской страницы (опционально) | `12345678` |
| `CONFLUENCE_TOKEN` | Personal Access Token | *(запросить у пользователя)* |

Токен никогда не логируется и не сохраняется в артефактах.

---

## Instructions

### Шаг 0 — Выбор способа подключения

**Приоритет:** MCP-коннектор → REST API

1. Проверь наличие MCP-инструментов для Confluence (search_mcp_registry).
   Если коннектор подключён и работает → используй MCP-инструменты.

2. Если MCP-коннектор недоступен → используй REST API через `curl`
   (см. раздел «REST API Fallback»).

3. Зафиксируй выбранный способ в журнале.

---

### Шаг 1 — Чтение существующей страницы (опционально)

Если пользователь указал существующую страницу для обновления:

**Через MCP:**
- Используй доступные MCP-инструменты для чтения страницы.

**Через REST API:**
```bash
curl -s -H "Authorization: Bearer $CONFLUENCE_TOKEN" \
  -H "Content-Type: application/json" \
  "$CONFLUENCE_BASE_URL/rest/api/content/<PAGE_ID>?expand=body.storage,version,title" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
print('Title:', data['title'])
print('Version:', data['version']['number'])
print('---')
print(data['body']['storage']['value'])
"
```

**Результат:**
- Текущий заголовок и версия страницы.
- Содержимое в формате Confluence Storage Format (XHTML).
- Если страница не найдена → зафиксировать, перейти к созданию новой.

---

### Шаг 2 — Конвертация Markdown → Confluence Storage Format

Confluence Server хранит контент в формате Storage Format (подмножество XHTML).
Необходимо сконвертировать markdown-документацию.

**Правила конвертации:**

| Markdown | Confluence Storage Format |
|----------|--------------------------|
| `# Заголовок` | `<h1>Заголовок</h1>` |
| `## Заголовок` | `<h2>Заголовок</h2>` |
| `**жирный**` | `<strong>жирный</strong>` |
| `*курсив*` | `<em>курсив</em>` |
| `` `код` `` | `<code>код</code>` |
| Блок кода | `<ac:structured-macro ac:name="code">...</ac:structured-macro>` |
| Таблица | `<table><tbody><tr><th>...</th></tr>...</tbody></table>` |
| Список `-` | `<ul><li>...</li></ul>` |
| Список `1.` | `<ol><li>...</li></ol>` |
| `> цитата` | `<blockquote><p>цитата</p></blockquote>` |

**Блок кода в Confluence Storage Format:**
```xml
<ac:structured-macro ac:name="code">
  <ac:parameter ac:name="language">rust</ac:parameter>
  <ac:parameter ac:name="theme">Confluence</ac:parameter>
  <ac:parameter ac:name="linenumbers">true</ac:parameter>
  <ac:plain-text-body><![CDATA[
    // код здесь
  ]]></ac:plain-text-body>
</ac:structured-macro>
```

**Предупреждающий блок (для зон неопределённости):**
```xml
<ac:structured-macro ac:name="warning">
  <ac:rich-text-body>
    <p>Требует уточнения по соседним модулям</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

**Информационный блок:**
```xml
<ac:structured-macro ac:name="info">
  <ac:rich-text-body>
    <p>Текст информации</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

Для конвертации используй Python-скрипт. Если файл большой — обрабатывай
построчно, не загружай целиком в промежуточные переменные.

---

### Шаг 3 — Публикация / обновление страницы

#### Создание новой страницы

**Через REST API:**
```bash
curl -s -X POST \
  -H "Authorization: Bearer $CONFLUENCE_TOKEN" \
  -H "Content-Type: application/json" \
  "$CONFLUENCE_BASE_URL/rest/api/content" \
  -d @payload.json
```

**Структура `payload.json` для создания:**
```json
{
  "type": "page",
  "title": "<project-name> — Документация модуля <module-name>",
  "space": {"key": "<SPACE_KEY>"},
  "ancestors": [{"id": "<PARENT_PAGE_ID>"}],
  "body": {
    "storage": {
      "value": "<сконвертированный XHTML>",
      "representation": "storage"
    }
  }
}
```

#### Обновление существующей страницы

При обновлении обязательно передать текущую версию + 1:

```json
{
  "type": "page",
  "title": "<текущий заголовок>",
  "version": {"number": <текущая_версия + 1>},
  "body": {
    "storage": {
      "value": "<обновлённый XHTML>",
      "representation": "storage"
    }
  }
}
```

**Через REST API:**
```bash
curl -s -X PUT \
  -H "Authorization: Bearer $CONFLUENCE_TOKEN" \
  -H "Content-Type: application/json" \
  "$CONFLUENCE_BASE_URL/rest/api/content/<PAGE_ID>" \
  -d @payload.json
```

#### Обработка ошибок

| HTTP-код | Причина | Действие |
|----------|---------|----------|
| 200 / 201 | Успех | Зафиксировать URL и ID страницы |
| 401 | Невалидный токен | Сообщить пользователю, запросить новый |
| 403 | Нет прав на запись | Сообщить пользователю, указать space и page |
| 404 | Страница / space не найдены | Проверить SPACE_KEY и PARENT_PAGE_ID |
| 409 | Конфликт версий | Перечитать текущую версию, повторить |

---

### Шаг 4 — Верификация публикации

После успешного ответа API проверь, что страница доступна:

```bash
curl -s -H "Authorization: Bearer $CONFLUENCE_TOKEN" \
  "$CONFLUENCE_BASE_URL/rest/api/content/<PAGE_ID>?expand=version" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
print('ID:', data['id'])
print('Title:', data['title'])
print('Version:', data['version']['number'])
print('URL:', data['_links']['base'] + data['_links']['webui'])
"
```

Зафиксировать в журнале:
- ID страницы
- URL страницы
- номер версии
- статус: создана / обновлена

---

### Шаг 5 — Формирование журнала

Создай или дополни `docs/dialogs/publisher-session.md`:

```markdown
# Publisher Session

## Параметры подключения
- URL: <base_url>
- Space: <space_key>
- Способ: MCP / REST API

## Действие
- Тип: создание / обновление
- Исходный файл: <путь к md>
- Страница: <title>
- ID: <page_id>
- URL: <page_url>
- Версия: <version>

## Результат
- Статус: успех / ошибка
- Ошибка: ... (если есть)

## Зоны неопределённости
- ... (если были проблемы с конвертацией)
```

---

## Constraints

- Токен доступа **никогда** не логируется в артефакты и журналы.
- Язык заголовков и метаданных страницы: **русский**.
- При конвертации не терять блоки кода и таблицы.
- Зоны неопределённости из документации отображать как `warning`-макросы.
- Не удалять и не перезаписывать чужие страницы без явного подтверждения.
- Если страница уже существует — обновлять, а не создавать дубль.

---

## Output Format

### Артефакты при работе в составе пайплайна

```
projects/<project-name>/docs/dialogs/publisher-session.md
```

Сама страница создаётся в Confluence (внешний артефакт).

### Вывод для пользователя

После публикации вывести:

```
Страница опубликована:
- Title: <заголовок>
- URL: <ссылка>
- Version: <номер>
- Space: <ключ пространства>
```

---

## REST API Fallback — справочник эндпоинтов

Все запросы к Confluence Server REST API v1:

| Операция | Метод | Endpoint |
|----------|-------|----------|
| Получить страницу | GET | `/rest/api/content/{id}?expand=body.storage,version` |
| Поиск страницы по title | GET | `/rest/api/content?spaceKey={key}&title={title}` |
| Создать страницу | POST | `/rest/api/content` |
| Обновить страницу | PUT | `/rest/api/content/{id}` |
| Получить дочерние страницы | GET | `/rest/api/content/{id}/child/page` |
| Получить пространства | GET | `/rest/api/space` |

Заголовки для всех запросов:
```
Authorization: Bearer <token>
Content-Type: application/json
```
