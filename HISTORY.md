# HISTORY — доработки форка

Форк [`grbr2010/dsh-data-agent`](https://github.com/grbr2010/dsh-data-agent) от
[`omdsh-dev/dsh-data-agent`](https://github.com/omdsh-dev/dsh-data-agent) **v0.2.0**
(upstream-коммит `3016f67`, 2026-09-24). Ниже — все отличия от upstream и их причины.

## 2026-09-26 — Oracle-фиксы и настраиваемый язык обогащения

### Исправления Oracle-адаптера

Портированы из внутрисборочного патча `patch-data-agent-oracle.mjs` (правки
скомпилированных бандлов) в исходники — патч-скрипт больше не нужен.

| # | Фикс | Причина |
|---|------|---------|
| 1 | `src/clients.ts`: в plain/introspection-префикс sqlplus добавлены `SET LINESIZE 32767`, `SET WRAP OFF`, `SET RECSEP OFF`, `SET MARKUP CSV ON DELIMITER '\|' QUOTE OFF` | Дефолтный LINESIZE 80 переносил каждую строку словаря данных — парсер терял строки, скан «успешно» находил 0 таблиц. Даже с LINESIZE 32767 строки обрезались: sqlplus паддит колонки до *определённой* ширины, а при AL32UTF8 CHAR-семантике это до 4 позиций на символ (`all_tab_comments` VARCHAR2(4000 CHAR) → 16000 позиций) — строка шире любого LINESIZE теряет хвостовые поля. MARKUP CSV убирает паддинг полностью |
| 2 | `src/clients.ts`: `NLS_LANG=AMERICAN_AMERICA.AL32UTF8` в env клиента (с паролем и без — wallet/OS-аутентификация) | Без пина клиентского charset не-ASCII комментарии (русский) искажаются или превращаются в `?????` |
| 3 | `src/catalog-adapters.ts`: `NVL(REPLACE(REPLACE(...comments,CHR(13),' '),CHR(10),' '),'')` для табличных и колоночных комментариев словаря | Комментарии с CR/LF разрезали одну логическую строку вывода на несколько физических — парсер строк отбрасывал их целиком |
| 4 | `src/catalog-adapters.ts`: index-строки фильтруются `EXISTS (... all_tab_columns ...)` | Колонки function-based индексов видны в `all_ind_columns` под скрытыми системными именами (`SYS_NC...$`), которых нет в `all_tab_columns` (единственный источник колонок скана) — валидация роняла весь прогон ошибкой `Catalog relation has unknown asset reference` |
| 5 | `src/clients.ts` (`buildClientStdin`): дописывание `;` к незавершённым SQL-выражениям в plain-режиме | sqlplus читает SELECT без `;`/`/` до EOF, ничего не исполняет и выходит 0 с пустым выводом (SHOW/DESCRIBE не трогаются — им терминатор не нужен) |

### Новая настройка `enrichmentLanguage`

- Тип: `'zh' \| 'ru' | 'en'`, дефолт `'zh'` (поведение upstream не меняется).
- Управляет языком **выхода** AI-кандидатов бизнес-смысла при сканировании каталога.
- Включение в `cordis.patch.yml` профиля:
  ```yaml
  - id: data-agent
    config:
      enrichmentLanguage: ru
  ```
- Промпты `ru`/`en` — единый английский шаблон инструкций (`englishCatalogMeaningPrompt`),
  локализованы только требуемый язык выхода и примеры запрещённых формулировок
  (правило 5). Промпт `zh` оставлен дословно как в upstream.

### Тесты

- `tests/clients.spec.ts`: обновлены 2 теста, кодировавшие старое поведение
  (пустой env Oracle, отсутствие терминатора), добавлены ассерты новых SET-строк.
- `tests/catalog-ai.spec.ts`: новый тест `selects the system prompt per enrichment
  language` — каждый язык даёт свой промпт, `zh` остаётся дефолтом, все три различны.

### Документация

- `README.md` — русская версия стала основной; китайская перенесена в
  [`README.zh.md`](README.zh.md), английская — [`README.en.md`](README.en.md);
  языковое меню обновлено во всех трёх файлах.

## Планы

- Перевести сборку образа DSH на этот форк (`github:grbr2010/dsh-data-agent`),
  удалить `patch-data-agent-oracle.mjs` из Dockerfile.
- Предложить фиксы 1–5 upstream отдельным PR с корректным описанием.
