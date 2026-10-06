---
name: kinopoisk-imdb-migrate
description: "Migrate a user's movie ratings and 'Буду смотреть' watchlist from Kinopoisk to IMDb: scrape the Kinopoisk pages with a browser-console script, build a CSV, match titles to IMDb IDs via Letterboxd import/export and the IMDb suggestion endpoint, then bulk-import ratings (Tampermonkey userscript) and the watchlist (console script) in chunks. Use when the user wants to move or back up their Kinopoisk history to IMDb or Letterboxd. Triggers: «перенести оценки с Кинопоиска», «Кинопоиск в IMDb», «экспорт Кинопоиска», «перенос watchlist», «Буду смотреть в IMDb», «Kinopoisk to IMDb», «Letterboxd импорт»."
---

# Перенос оценок и «Буду смотреть»: Кинопоиск → IMDb

Инструкция для ИИ-агента. Гайд для людей: [README](https://github.com/Eniggman/kinopoisk-imdb-migrate#readme). Ниже описано, что реально делают скрипты. Там, где README с ними расходится, это отмечено.

## Принципы

- Все действия в аккаунтах Кинопоиска, Letterboxd и IMDb выполняет **человек в своём залогиненном браузере**. Не спрашивай пароли и cookies, не пытайся их получить.
- **Спроси подтверждение** перед: импортом в Letterboxd, массовой простановкой оценок и добавлением в Watchlist на IMDb (откатывать придётся вручную), установкой стороннего userscript, правкой скриптов репозитория и перезаписью JSON.
- Сначала всегда тестовый чанк из 5 фильмов, потом остальное.

## Шаг 0. Окружение

```bash
git clone https://github.com/Eniggman/kinopoisk-imdb-migrate && cd kinopoisk-imdb-migrate
python --version            # нужен 3.10+
pip install pandas          # нужен только Scripts/rebuild_watched.py
mkdir -p data/import_chunks # make_chunks.py сам папку не создаёт
```
Все скрипты запускаются **из корня репозитория** (пути относительные).

**Windows-пути.** `Scripts/convert_to_letterboxd.py`, `Scripts/prepare_watchlist.py` и `Scripts/check_ids.py` используют пути вида `r'data\...'`. На Linux/macOS они упадут с `FileNotFoundError`. С согласия человека замени их на `'data/...'` (на Windows это тоже работает).

VPN: если Кинопоиск отдаёт 403 или капчу, README советует VPN с обфускацией на порту 443. Его настраивает человек, не агент.

## Шаг 1. Выгрузка с Кинопоиска (делает человек)

1. Открыть https://www.kinopoisk.ru/mykp/movies/, внизу выбрать «Показывать по 200» и прокрутить страницу до конца.
2. F12 → Console → вставить весь `Scripts/extract_movies.js` → Enter. Скрипт только читает страницу (DOM) и ничего не отправляет.
3. Строки вида `Название | Оценка: 8 | Дата: 29.09.2025, 14:40` сохранить в `data/watched.txt`. Если страниц несколько, повторить на каждой.
4. То же на странице «Буду смотреть». Результат сохранить в `data/watch_later.txt` и привести к формату **одно название на строку**: убрать ` | Оценка: … | Дата: …` и служебные строки (`Собрано уникальных…`, `Если фильмов меньше…`). `prepare_watchlist.py` считает названием каждую непустую строку, кроме начинающихся с `=`. Файл-пример в репо оформлен Markdown-таблицей, но этот формат скрипту не подходит.

## Шаг 2. CSV-база `data/кинопоиск_база.csv` (делает агент, скрипта нет)

Преобразуй `data/watched.txt` в CSV (UTF-8, разделитель `;`):
```
Название (RUS);Название (ENG);Оценка;Дата просмотра
Мементо (2000);Memento;9;12.03.2026, 19:55
```
- Год должен быть в скобках в русском названии: из него `convert_to_letterboxd.py` берёт `Year`.
- Английское название подставь, если уверен. Если не уверен, оставь поле пустым (тогда используется русское) и составь для человека список сомнительных.
- Оценку можно писать как `9` или `**9**` (звёздочки скрипты убирают). Дата: `ДД.ММ.ГГГГ, ЧЧ:ММ`.
- Строки с `Оценка: Нет оценки` покажи человеку и спроси, пропустить их или оценить.
- Проверь, что нет дубликатов.

## Шаг 3. `watched.md`

```bash
python Scripts/rebuild_watched.py
```
Создаёт `watched.md` **в корне репо** (а не в `data/`, как нарисовано в README) с таблицей `№ | **Название (Год)** [English] | Оценка | Дата`. Статистику, графики и топ-5, обещанные в README, текущий скрипт не делает. Шаг нужен для `fix_order.py`.

## Шаг 4. IMDb ID через Letterboxd

```bash
python Scripts/convert_to_letterboxd.py   # -> data/letterboxd_import.csv (Title, Year, Rating10, WatchedDate)
```
Человек (после твоего подтверждения): letterboxd.com → Settings → Import & Export → загрузить `data/letterboxd_import.csv` и проверить сопоставления. Затем «Export your data», распаковать ZIP в `letterboxd_export/` в корне репо (нужен файл `letterboxd_export/ratings.csv`).

```bash
python Scripts/extract_from_letterboxd.py   # 2–5 мин, 10 потоков -> data/imdb_import_clean.json
```
Формат результата: `[{"name": "...", "rating": 1..10, "imdb": "tt..."}]`. Строки `No IMDb ID found…` и `Error fetching…` в выводе — это пропуски.

## Шаг 5. Пропуски и проверка ID

- README предлагает для этого `prepare_watchlist.py`, но на деле он обрабатывает только `data/watch_later.txt` (шаг 8) и в JSON оценок ничего не дописывает. Пропущенные оценки добавь в `data/imdb_import_clean.json` вручную. ID можно искать тем же эндпоинтом, что используют скрипты: `https://v3.sg.media-imdb.com/suggestion/x/<название>.json`. Каждый найденный ID покажи человеку.
- Проверка:
  ```bash
  python Scripts/check_ids.py
  ```
  Скрипт печатает случайные ~13% фильмов и их названия на IMDb, но сам не решает, где ошибка. Сравни пары сам и составь список несовпадений.

## Шаг 6. Порядок и чанки

```bash
cp data/imdb_import_clean.json data/imdb_import_clean.backup.json   # fix_order перезаписывает файл
python Scripts/fix_order.py      # переворачивает порядок watched.md: последняя строка станет первой
python Scripts/make_chunks.py    # data/import_chunks/part_1_5_items.json, part_2_10…, part_3_20…, part_4_40…, остаток
```
CSV и `watched.md` должны идти от новых просмотров к старым (как на Кинопоиске), тогда на IMDb хронология совпадёт. `fix_order.py` печатает `Creating stub for missing…` (такие фильмы в JSON **не** добавляются) и `Leftovers in JSON` (они дописываются в конец). Покажи человеку оба списка.

## Шаг 7. Импорт оценок на IMDb (делает человек)

1. Предупреди: импортёр — **сторонний** userscript с GreasyFork: https://greasyfork.org/en/scripts/573527-imdb-ratings-importer-json. Он работает в залогиненной сессии IMDb. Ставить его или нет, решает человек.
2. Tampermonkey: в настройках расширений браузера включить режим разработчика и нажать Install у скрипта.
3. https://www.imdb.com/ratings/ (под своим логином) → кнопка «Import Ratings (JSON)» → сначала `part_1_5_items.json`.
4. Человек проверяет 5 оценок на сайте, затем загружает остальные чанки по порядку. Строка `FORBIDDEN: This title is not yet open for voting` — это нормально.

## Шаг 8. Watchlist

```bash
python Scripts/prepare_watchlist.py   # -> data/imdb_watchlist_import.json + data/watchlist_chunks/watchlist_part_N.json (по 40)
```
- Записи с `"imdb": "NOT_FOUND"` исправь (словарь `manual_fixes` в начале скрипта, затем перезапуск) или удали. Их нельзя отправлять в импорт.
- В `Scripts/imdb_watchlist_importer.js` замени массив `HARDCODED_MOVIES` на содержимое чанка, оставив поля `name` и `imdb`.
- Человек: https://www.imdb.com/watchlist/ под логином → F12 → Console → вставить скрипт → кнопка **START DUAL-MODE IMPORT**. Вкладку не закрывать и не переключаться, ждать `🎯 MISSION ACCOMPLISHED!`.
- Лог: `[GQL SUCCESS]` и `[CLASSIC SUCCESS]` означают успех, `[FATAL FAIL] … Status 404` — ошибку. Красные `BLOCKED_BY_ADBLOCKER` в консоли не мешают.
- Если GraphQL не сработал, скрипт пробует POST `/watchlist/{id}/add`. README называет этот эндпоинт мёртвым (404), поэтому массовые `[FATAL FAIL] … Status 404` означают, что не сработали оба метода: переходи к альтернативе ниже.

### Альтернатива: `Scripts/add_watchlist_batch.js` (создаёт пользователь)

Этого файла **намеренно нет в репозитории**: пользователь создаёт его сам. Проверь, есть ли он у человека: `ls Scripts/add_watchlist_batch.js` (Windows: `Test-Path Scripts\add_watchlist_batch.js`).
- Если файл есть: прочитай его и объясни человеку, что он делает, до запуска. Запускает человек сам в консоли (F12) на своей странице https://www.imdb.com/watchlist/ под логином, по тем же правилам, что и основной импортёр (вкладка активна, чанками).
- Если файла нет: помоги человеку создать его, опираясь только на то, что сказано в README. Это скрипт для консоли браузера (F12) для «быстрого добавления в Watchlist через iframe», альтернатива на шаге 8. Он добавляет фильмы методом «iframe + настоящий клик», при котором браузер сам прикрепляет все куки и токены. Других деталей README не задаёт, поэтому каждое решение (формат входного списка, селекторы кнопок, паузы) согласуй с человеком и не выдавай за «официальную» версию из репо. Готовый код покажи человеку и получи «да» до запуска; первый прогон — на тестовом чанке из 5 фильмов.

## Проверка успеха

- Число строк в CSV ≈ числу записей в `imdb_import_clean.json` ≈ числу оценок на https://www.imdb.com/ratings/. Все расхождения перечисли.
- Число фильмов в Watchlist IMDb равно числу записей с валидным `tt`-ID.

## Частые ошибки

| Симптом | Решение |
|---|---|
| `FileNotFoundError: data\…` | Windows-путь на Linux/macOS: см. шаг 0 |
| `FileNotFoundError: data/import_chunks/…` | `mkdir -p data/import_chunks` |
| `KeyError: 'Название (RUS)'` | В CSV неверный заголовок или разделитель не `;` |
| Нет `watched.md` для `fix_order.py` | Сначала выполни `python Scripts/rebuild_watched.py` |
| Кинопоиск отдаёт капчу или 403 | Нужен VPN (решает человек), страницу открывать вручную |
| Все `FATAL FAIL` в Watchlist | Человек не залогинен или IMDb режет запросы: подождать 10 мин и взять чанк поменьше. Если всё равно 404, используй `Scripts/add_watchlist_batch.js` (см. шаг 8) |
