# Данные отчета

Справочник к `SKILL.md`: что приходит из `get_report_data` и как читать числа.

## Черновик отчета: что возвращает get_report_data

```
{
  "title": "",                       // заполнить
  "language": "ru",
  "since": "2026-07-01" | null,
  "until": "2026-07-31" | null,
  "blocks": ["overview", "sov", ...],           // фактически собранные блоки
  "meta": { "company": "...", "theme": "...", "product": "...",
            "generatedAt": "ISO", "periodLabel": "...", "sectionCount": 3 },
  "sections": [ { "index": 1, "title": "...",
                  "blocks": [ { "key": "overview", "title": "Ключевые показатели",
                                "data": { ...цифры... },
                                "conclusion": { "keyFindings": [], "analysis": "" } } ] } ],  // заполнить
  "summary": { "overall": "", "sections": [], "recommendations": [] }                         // заполнить
}
```

Заполняются ровно три вещи: `title`, `conclusion` у каждого блока и `summary`. Остальное — данные, их не трогать.

Числа приходят УЖЕ ОТФОРМАТИРОВАННЫМИ, ровно как они напечатаны в отчете:
- доли — строки вида `"85.7%"`, не дроби; при недоступном значении `"—"`;
- изменения к прошлому периоду — строки вида `"+6.3 п.п."` / `"−20.9 п.п."` (для счетчиков просто `"+14"`);
- счетчики — целые числа;
- названия движков — витринные (`ChatGPT`, `Perplexity`, `Google AIO`, `Алиса`).

Отсюда правило: числа в текст переносить как есть, не пересчитывать и не «уточнять» знаки после запятой. Свои проценты появляться не должны — читатель отчета видит те же строки.

Важное следствие: в черновик попадает только то, что видит читатель. Внутренних полей (размер выборки по каждой точке графика, объем ссылок по домену, `isProject`/`isCompetitor`, id промптов) там НЕТ. Если вывод требует таких данных — их берут отдельными инструментами (`run_cube_query`, `query_prompt_metric_snapshot`), а не выдумывают.

Ключи блоков и форма `data` (ключ — это то, что стоит в `blocks` и в `insights[].block`):

Раздел «Конкуренты»:
- `overview` (1.1 Ключевые показатели): `{ brandVisibility: %, shareOfVoice: %, brandMentions: n, answersAnalyzed: n, changeVsPreviousPeriod?: { brandVisibility, shareOfVoice, brandMentions } }` — блок изменений присутствует, только если было с чем сравнивать.
- `sov` (1.2 Share of Voice): `{ brands: [ { brand, shareOfVoice: % } ] }`.
- `bmr` (1.3 Brand Visibility): `{ brand, brandVisibility: %, answersAnalyzed: n }`.
- `dynamics` (1.4 Динамика бренда): `{ points: [ { date, brandVisibility: %, shareOfVoice: % } ] }`. Гранулярность бакетов выбирается автоматически по длине периода; размера выборки в точке тут НЕТ.
- `leaderboard` (1.5 Лидерборд упоминаний): `{ brands: [ { brand, total: n, directBrandMentions: n, linkMentions: n } ] }`, где `total = directBrandMentions + linkMentions`.
- `providerActivity` (1.6 Активность по нейросетям): `{ byAssistant: [ { assistant, answersWithAnyTrackedBrand: n, ourBrand: n, competitors: n } ] }` — пересечение `ourBrand` и `competitors` допустимо.
- `mentionExclusivity` (1.7 Эксклюзивность упоминаний): `{ onlyOurBrand: { answers: n, share: % }, sharedWithCompetitors: { answers: n, share: % }, onlyCompetitors: { answers: n, share: % } }`.
- `competitorExclusivity` (1.8 Эксклюзивность по конкурентам): `{ competitors: [ { competitor, totalAnswers: n, aloneOnly: n, withOtherCompetitors: n, withOurBrand: n } ] }`.
- `comparison` (1.9 Сравнение с конкурентами): `{ brands: [ { brand, answers: n, mentions: n, linkMentions: n } ] }`, где `answers` — число разных ответов с брендом (база SoV), `mentions` — все упоминания (прямые плюс ссылочные).

Раздел «Ссылки»:
- `topSources` (2.1 Топ-20 источников): `{ sources: [ { domain, answers: n } ] }` — только по ответам, где упомянут бренд.
- `sourcesByProvider` (2.2 Источники по нейросетям): `{ byAssistant: [ { assistant, sources: [ { domain, answers: n } ] } ] }`.
- `topUrls` (2.3 Топ-10 URL): `{ byAssistant: [ { assistant, urls: [ { url, answers: n } ] } ] }` — только в разрезе движков, общего списка тут нет; привязки к упоминанию бренда у этого блока тоже нет.

Раздел «Запросы»:
- `queryOverview` (3.1 Соответствие ожидаемому): `{ expectedResponseMatch: %, scoredResponses: n, promptsWithExpected: n }`.
- `queryDynamics` (3.2 Динамика соответствия): `{ points: [ { date, match: % } ] }`.
- `queryTopPrompts` (3.3 Топ-10 запросов): `{ prompts: [ { query, match: %, scoredResponses: n } ] }`.

Особенности, которые надо знать и НЕ принимать за баг:
- Раздел «Запросы» отсутствует целиком, если ни у одного промпта в периоде не задан эталонный ответ.
- База раздела «Запросы» шире остальных: соответствие считается по ВСЕМ симуляциям, включая промпты с `excludeFromProjectMetrics=true`, тогда как BV и SoV такие промпты исключают. Поэтому `scoredResponses` законно не совпадает с `answersAnalyzed` — это разные базы по замыслу; при сопоставлении просто оговаривать разную базу.
- Блоки считаются независимо: если `blocks` сужен, недостающих блоков в черновике просто нет — на них не ссылаться.
- Точный перечень блоков и форм `data` продублирован в описании самого инструмента `get_report_data`. Если он разойдется с текстом выше — верить описанию инструмента, оно генерируется из кода.

---

## Предыдущий период и динамика

- `overview.changeVsPreviousPeriod` заполняется АВТОМАТИЧЕСКИ и только если передан `since`: предыдущее окно — такой же длины, непосредственно перед `since`. Значения — готовые строки (`"−20.9 п.п."`), переносить в текст как есть. Без `since` (за все время) этого поля в `data` просто нет.
- Изменений по остальным блокам система не дает. Чтобы говорить о динамике лидерборда, движков, источников, запросов — вызвать `get_report_data` ВТОРОЙ раз с предыдущим окном (`since_prev = since - (until - since)`, `until_prev = since`) и сравнить руками. Числа второго вызова использовать только для сравнения; сохраняется текущий период.
- Внутрипериодная динамика доступна без второго вызова: `dynamics.points` и `queryDynamics.points`.
- Если второго периода нет или в нем пусто — писать «динамика недоступна: нет данных за предыдущий период», а не додумывать.

---
