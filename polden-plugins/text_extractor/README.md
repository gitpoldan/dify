# Text Extractor

**Author:** polden
**Version:** 0.1.0
**Type:** tool
**Tool name:** `text_extractor`

Извлекает текст из приложенных документов и отдаёт его одним markdown-блоком.
Форк плагина `langgenius/dify_extractor` 0.0.12 с тремя отличиями:

1. файловые параметры необязательные, поэтому инструмент можно подключать к ноде
   Agent (у оригинала обязательный параметр `file` типа `file` вызывал
   `ValueError: file type parameter file not supported in agent`);
2. за один вызов обрабатываются **все** приложенные файлы, а не первый;
3. явно поддерживаются `.txt`, `.text`, `.log`; неизвестные форматы не
   декодируются «как текст» молча, а попадают в `skipped_files` с причиной.

## Поддерживаемые форматы

`.pdf`, `.docx`, `.pptx`, `.xlsx`, `.xls`, `.csv`, `.md`, `.markdown`, `.mdx`,
`.htm`, `.html`, `.json`, `.yaml`, `.yml`, `.txt`, `.text`, `.log`

Для `.doc`, `.ppt`, `.rtf`, `.odt`, `.xlsm` инструмент возвращает подсказку,
в какой формат пересохранить файл.

## Параметры

| Параметр | Тип | Форма | Назначение |
|---|---|---|---|
| `files` | files | form | Основной вход. В Chatflow подключите `sys.files` |
| `file` | file | form | Совместимость со старыми workflow на один файл |
| `file_url` | string | llm | Запасной вход по http(s); несколько URL через запятую или перевод строки |
| `max_content_chars` | number | form | Общий бюджет символов, `0` — без ограничения |
| `file_base_url` | string | form | База вроде `http://api:5001` для относительных `/files/...` |

## Результат

Текст: содержимое одного файла как есть, либо блоки `## имя файла` при нескольких.

| Переменная | Что внутри |
|---|---|
| `documents` | Документы всех обработанных файлов |
| `images` | Картинки, извлечённые из pdf / docx / pptx |
| `processed_files` | `filename`, `extension`, `source`, `characters`, `truncated` |
| `skipped_files` | `filename`, `extension`, `reason` |

Вызов не падает из-за одного проблемного файла: ошибка чтения, неподдерживаемый
формат и исчерпанный бюджет символов попадают в `skipped_files`, остальные файлы
обрабатываются. Если не извлёкся ни один файл, текст содержит перечень причин.

## Сценарии

**Workflow.** Подключите `sys.files` к `files`, `max_content_chars` оставьте `0`,
берите `text` или `documents` из выхода ноды.

**Agent.** Поле `files` в настройках инструмента внутри ноды **оставьте пустым**.
Привязка `{{#sys.files#}}` к параметру инструмента ломает вызов ноды:

```
TypeError: Object of type File is not JSON serializable
```

Dify конвертирует `File` в JSON только для параметров верхнего уровня
(`convert_parameters_to_plugin_format`), а вложенные в `tools[].runtime_parameters`
объекты остаются как есть. Поэтому вложения передавайте через стратегию
`polden/tool_capture_agent`: `{{#sys.files#}}` — в её параметр `Files`, а оттуда в
инструмент через io_wiring:

```json
{"wires": [
  {"from": "agent.files",
   "to": ["tool.text_extractor.files"],
   "optional": true}
]}
```

Параметр становится управляемым: модель его не видит и не заполняет, у неё остаётся
только `file_url` на случай ссылки. Задайте `max_content_chars` (например, `40000`),
чтобы большой документ не вытеснил контекст диалога.

## Разработка

```bash
python tests/test_dispatch.py
python tests/test_extract_flow.py
python package.py
```

`package.py` собирает `polden-text_extractor_0.1.0.difypkg` рядом с папкой плагина.
