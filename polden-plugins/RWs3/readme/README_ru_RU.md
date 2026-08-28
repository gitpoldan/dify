# RWs3 — плагин S3 Read/Write

Простой и надёжный Dify-плагин для **S3** и **MinIO**: список объектов, чтение, запись и удаление файлов.

Автор: **gitpoldan**. ID провайдера: `gitpoldan/rws3`.

## Возможности

- **AWS S3 и MinIO** из одного провайдера — path-style для MinIO настраивается автоматически.
- **Опциональный prefix guard** — ограничение всех операций пространством имён внутри бакета.
- **Структурированный JSON** — каждый инструмент возвращает `{ "ok": true, "data": ... }` или `{ "ok": false, "error": ... }`.
- **Лимиты размера** — настраиваемые ограничения чтения/записи для безопасной работы с LLM.

## Инструменты

| Инструмент | Описание |
|------------|----------|
| `s3_list` | Список объектов по префиксу (содержимое «каталога») |
| `s3_read` | Чтение файла как UTF-8 текст (с лимитом и флагом `truncated`) |
| `s3_write` | Запись UTF-8 текста |
| `s3_delete` | Удаление объекта (идемпотентно) |

## Credentials провайдера

| Поле | Обяз. | По умолчанию | Описание |
|------|-------|--------------|----------|
| `endpoint` | Нет | пусто (AWS S3) | Хост MinIO |
| `access_key_id` | Да | — | Ключ доступа |
| `secret_access_key` | Да | — | Секретный ключ |
| `use_https` | Нет | `true` | HTTPS для custom endpoint |
| `bucket` | Да | — | Имя бакета |
| `region` | Нет | `us-east-1` | Регион AWS |
| `prefix` | Нет | пусто | Базовый префикс — все ключи резолвятся под ним |
| `max_read_bytes` | Нет | `204800` | Лимит чтения |
| `max_write_bytes` | Нет | `5242880` | Лимит записи |

При сохранении credentials выполняется проверка `head_bucket()`.

## Ограничения

- Только **текст** (UTF-8) — бинарные данные и base64 не поддерживаются в v0.0.1.
- Большие файлы при чтении **обрезаются** — проверяйте `truncated: true`.
- Ключи не должны содержать `..`, ведущий `/` или `\`.

## Локальная разработка

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
pytest tests/
python -m main
```

## Сборка

```bash
dify plugin package ./
# или
python package.py
```

Результат: `../gitpoldan-rws3_0.0.1.difypkg`.

## Лицензия

MIT. См. [LICENSE](../LICENSE).

## Поддержка

- Исходники: https://github.com/gitpoldan/dify/tree/main/polden-plugins/RWs3
- GitHub Issues: https://github.com/gitpoldan/dify/issues
- Email: bv2020donch@gmail.com
