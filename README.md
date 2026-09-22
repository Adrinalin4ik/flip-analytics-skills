# Flip Analytics Skills

Плагин-маркетплейс со скиллами для работы с Flip Analytics — аналитикой видимости бренда в ответах нейросетей.

## Установка

В Claude Code:

```
/plugin marketplace add Adrinalin4ik/flip-analytics-skills
/plugin install flip-analytics@flip-analytics-skills
```

После установки скиллы подхватываются автоматически: модель вызывает нужный по описанию, либо его можно позвать явно — например `/flip-generate-report`.

Для локальной разработки вместо первой команды:

```
/plugin marketplace add ./path/to/flip-analytics-skills
```

## Что внутри

```
.claude-plugin/marketplace.json          — каталог плагинов этого репозитория
plugins/flip-analytics/
  .claude-plugin/plugin.json             — манифест плагина
  skills/
    flip-generate-report/SKILL.md        — генерация аналитического отчета
```

### Скиллы

| Скилл | Назначение |
| --- | --- |
| `flip-generate-report` | Сборка аналитического отчета: чтение данных через `get_report_data`, авторские выводы по блокам, сохранение через `create_report`, правка через `get_report_document` + `update_report`. Плюс методология выводов и чек-лист проверок. |

## Как добавить новый скилл

1. Создать папку `plugins/flip-analytics/skills/<имя-скилла>/` и в ней `SKILL.md`.
2. В начале `SKILL.md` — фронтматтер:

   ```markdown
   ---
   name: имя-скилла
   description: Что делает и КОГДА его применять — по этому тексту модель решает, брать скилл или нет.
   ---
   ```

3. Тело файла — инструкции для модели. Крупные справочники выносить в соседние файлы (`references/`) и ссылаться на них из `SKILL.md`, чтобы он оставался компактным.
4. Добавить строку в таблицу скиллов выше и поднять `version` в `plugin.json` и `marketplace.json`.

Отдельный плагин (а не новый скилл внутри `flip-analytics`) имеет смысл заводить, когда появится набор скиллов для другой области: создать `plugins/<имя>/` с собственным `.claude-plugin/plugin.json` и дописать его в `plugins` в `.claude-plugin/marketplace.json`.
