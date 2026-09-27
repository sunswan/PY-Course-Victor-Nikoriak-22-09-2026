# Виправлення старого коду — нотатки викладача

> Файл для викладача, у книгу курсу (`docs/`) не входить. Тут — усе, що довелося змінити в коді й
> матеріалах старого курсу `PY-Course-Victor-Nikoriak-23_02` під час перенесення: уроки 34–42,
> бонус-урок pandas, довідники. На сторінках уроків цих списків немає — студенти бачать лише
> правильний код і пояснення «чому так».

## Урок 34. Django: forms, HTML practice

| Де | Було | Стало |
|---|---|---|
| `services.create_note`, `note_create` (`crispy_notes_project`) | `is_pinned` з форми не доходив до сервісу — прапорець губився при створенні | параметр `is_pinned` у сервісі й view; тест `test_is_pinned_is_saved_on_create` |
| `django_bootstrap_project/templates/base.html` | SRI-хеш Bootstrap JS не відповідав версії 5.3.3 — браузер блокував скрипт (меню, модальні вікна) | правильний хеш |
| обидва проєкти | тестів не було | `hello_app/tests.py` |

«Знайди помилку» уроку побудовано на баг `is_pinned`.

## Урок 35. DRF overview + Django vs FastAPI

| Де | Було | Стало |
|---|---|---|
| `services.update_note` (з уроку 34) | `save(update_fields=changed_fields)` без `updated_at` — `auto_now` не оновлювався, редагована нотатка не піднімалась у списку | `update_fields=changed_fields + ['updated_at']`; тест `test_patch_changes_only_sent_fields_and_updated_at` |

«Знайди помилку» уроку побудовано на цьому баг.

## Урок 36. Typing + Pydantic

| Де | Було | Стало |
|---|---|---|
| `news_dashboard/app/scraper.py`, категорія | URL розбирався рядком (`url.replace("https://www.rbc.ua", "")`) — для `auto.rbc.ua` категорією ставав домен | `urlsplit` у `NewsItem.derive_from_url` |

«Знайди помилку» уроку побудовано на цьому баг.

## Урок 37. FastAPI basics + Postman + OpenAPI

| Де | Було | Стало |
|---|---|---|
| `news_dashboard/app/main.py` | `@app.on_event("startup")` — застарілий API FastAPI | `lifespan` |
| `main.py`, `ScrapeRequest.pages` | будь-які URL — сервер завантажить що завгодно (SSRF) | лише сторінки rbc.ua, не більше 20; інше — `422` |
| `main.py`, `lang` | вільний рядок: `?lang=en` — порожній список без помилки | `Literal["uk", "ru"]` → `422` |
| `news_dashboard/app/scraper.py`, `fetch_one` | статус відповіді не перевірявся: сторінку «403» розбирало як стрічку, «0 новин, помилки немає»; `except Exception` ховав будь-які помилки | `raise_for_status()`; ловимо лише мережеві помилки й тайм-аут |
| `scraper.py`, `_parse_page` | друга копія парсера зі своїм словником категорій (і помилкою з піддоменами, урок 36) | `parse_rbc_news` + `NewsItem` з уроку 36 |
| `fastapi_demo/load_test.py` | клієнт `httpx` на 500 з'єднаннях сам гальмував вимір; тайм-аут 30 с < 40 с тесту | клієнт навантаження — `aiohttp` без ліміту з'єднань, тайм-аут 60 с |
| `load_test.py`, docstring | «`uvicorn app.main:app --reload`» — порт 8000, а тест стукає на 8001 | `uvicorn app.main:app --port 8001` |

Вимір `fastapi_demo/load_test.py`: перший запуск показав «❌ ПРОБЛЕМА» для 500 запитів до `/async-correct` (16.56 с замість 2 с). Винен був клієнт тесту: `httpx.AsyncClient` на сотнях з'єднань сам гальмує (500 запитів до `/health`: httpx 5.66 с, aiohttp 0.22 с), а тайм-аут 30 с менший за 40 с тесту. Після заміни клієнта на aiohttp і тайм-ауту 60 с — числа в таблиці уроку.

## Урок 38. FastAPI + SQLAlchemy: повний CRUD

| Де (`production_bot`) | Було | Стало |
|---|---|---|
| `core/database.py`, `get_db` | COMMIT після `yield` — з FastAPI ≥ 0.118 після відповіді: клієнт бачить успіх, навіть якщо COMMIT не вдався | `Depends(get_db, scope="function")`; `fastapi>=0.121`; тест |
| `core/database.py`, `get_db` | анотація `-> AsyncSession`, хоча це генератор (mypy `--strict` — помилка) | `-> AsyncIterator[AsyncSession]` |
| `core/database.py`, `alembic.ini` | адреса лише PostgreSQL, у `alembic.ini` — ще й окремо, з паролем | одна `DATABASE_URL` зі змінної середовища для застосунку й міграцій; без неї — SQLite |
| `repositories/base.py`, `user_repo.py` | `from sqlalchemy import func, select` усередині методів; `datetime` — теж | імпорти вгорі модуля |
| міграція, згенерована autogenerate | `server_default=sa.text('now()')` — лише PostgreSQL | `sa.func.now()` — PostgreSQL і SQLite |
| — (нове в уроці) | SQLite `lower()` знає лише латиницю: пошук «ЗЕЛЕНСЬК» не знаходив «Зеленськ» | `lower()` з Unicode для SQLite (`db.py`); тест пошуку на обох базах |

## Урок 39. Middlewares і кешування (Redis)

| Де | Було | Стало |
|---|---|---|
| `ai_bot/…/rate_limit_repo.py` | `INCR`, потім окремий `EXPIRE` лише при `count == 1`: збій між ними — ключ без TTL, блокування назавжди | транзакція `INCR` + `EXPIRE … NX`; «залишок» без TTL лікується |
| `ai_bot/…/rate_limit.py` | алгоритм названо «Sliding Window Counter», а це фіксоване вікно | назву виправлено; межу вікна показано вимірюванням |
| `production_bot/…/redis.py` | глобальна змінна `_redis_pool`, не підміниш у тестах | клієнт у `app.state` з `lifespan`, `Depends(get_redis)`; `fakeredis://` для тестів |
| `news_dashboard/…/main.py`, `/api/scrape/archive` | у фонову задачу передавали `db` запиту; статус задач — у базі новин | своя сесія бази в задачі; статус — Redis-hash з TTL; помилка → `failed` |
| — (урок 39) | рядки журналу `news_hub` нікуди не виводились: uvicorn налаштовує лише свої логери | `setup_logging()` у `lifespan` |
| — (урок 39) | інвалідація кешу в ендпоінті йшла б до COMMIT | middleware `invalidate_cache` — після COMMIT |

## Урок 40. Автентифікація та security basics

| Де | Було | Стало |
|---|---|---|
| `api.py` (урок 35) + групи зі старого курсу | через API учасник групи міняв (`200`) і видаляв (`204`) чужі нотатки: `get_note_detail` віддає нотатки групи, а перевірка автора була лише в HTML-views | `_get_own_note` для `PATCH`, `DELETE`, `pin` → `403`; тест |
| `settings.py`, `DEFAULT_AUTHENTICATION_CLASSES` | `Session` + `Basic`: пароль у кожному запиті; анонім отримував `403` | `JWT` першим + `Session`: анонім — `401` з `WWW-Authenticate: Bearer` |
| `settings.py` | `SECRET_KEY`, `DEBUG = True`, `ALLOWED_HOSTS` — у коді | зі змінних середовища (значення для навчання — за замовчуванням) |
| — (нове в уроці 40) | вхід для API лише через сесію | `/api/token/`, `/api/token/refresh/`, throttle 5/хв |
| `services.create_note` | у старому курсі не приймав `is_pinned` (виправлено в уроці 34) — конфлікт при перенесенні | `is_pinned` і `group` разом |

## Урок 41. Тестування API (pytest + httpx)

| Де | Було | Хто знайшов | Стало |
|---|---|---|---|
| `models.py` (урок 36) | `published_time` приймав лише «HH:MM»; ISO-час з `<time datetime>` → новину відхилено | тест конвеєра на збереженій сторінці | `field_validator`: з ISO береться час |
| `scraper.py` (урок 37, `news_dashboard`) | `resp.text()` на битому байті — `UnicodeDecodeError` повз `except`, `gather` губив усі сторінки | фейковий HTTP-сервер | `resp.text(errors="replace")` |
| `models.py`, `api.py` (36–37) | `host.endswith("rbc.ua")` пропускав `fakerbc.ua`: сервер завантажував чужий сайт на запит | покриття гілок → тест межових значень | `is_rbc_host` |
| `parser.py` (урок 36) | з beautifulsoup4 4.12 `_classes` падав на тезі без `class` | прогін на мінімальних версіях | порожні значення відкидаються |
| `tests/` (урок 39) | один рівень, HTML у рядках, мережа й `get_db` не тестувались, покриття занижене | — | unit / integration, фікстури, мок і фейк, `aclient`, `.coveragerc` |

## Урок 42. AI-інструменти розробника

| Де | Було | Хто знайшов | Стало |
|---|---|---|---|
| `news_hub/CLAUDE.md` | приклад старого курсу описував MongoDB, `nlp.py`, Streamlit | звірка з кодом | команди перевірки й правила проєкту |
| `news_hub/rss.py` (код агента) | новина без `<pubDate>` валила всю стрічку; `-0000` — як час сервера | тест рецензента | новина без часу; `-0000` = UTC |
| `news_hub/models.py` (код агента) | без `<category>` категорія «2026» | тест рецензента | розділ — перший сегмент шляху для не-rbc доменів |
| `news_hub/requirements.txt` | `zoneinfo` без `tzdata` — у Windows модуль не імпортується | тест рецензента (без бази поясів) | `tzdata>=2024.1` |
| `depression_dashboard` | 10 вад з таблиці вище + пороги, підписи, залежності | `tests/test_review.py`, AppTest | див. таблицю й README |
| [довідник Claude Code](../docs/modules/m4/ai/claude_code.md) | моделі, режими, події hooks, Agent SDK, CI — станом на 2025-05 | звірка з документацією й `claude --help` | виправлено; AI-аудитор помилився в 4 пунктах, їх відкинуто |

### `depression_dashboard` — 10 вад, виправлених у копії курсу

| # | Знахідка | Як доведено | Виправлення в копії |
|---|---|---|---|
| 1 | «топ-ознаки» й «найсильніші предиктори» — за абеткою | тест порядку після `jsonify` | `StrictJSONProvider(sort_keys=False)`, сортування в UI |
| 2 | назви кластерів вшиті в UI: «Sleep Deprived» — кластер, що спить **найбільше** (8,15 год), «Financially Stressed» — кластер з 2 людей | центроїди моделі старого курсу; тест «назва відповідає профілю» | назву дає профіль кластера порівняно з середнім |
| 3 | `/api/predict` приймав похідні ознаки від клієнта: `Risk_Score=-100` змінював прогноз; `Age=-500` → `200`; пропущене поле → `500` з текстом pandas | тести з поганими даними | `StudentProfile` (Pydantic, `extra="forbid"`) → `422`; похідні ознаки — `add_features` на сервері, одна функція для навчання й прогнозу |
| 4 | повзунок CGPA ні на що не впливав: CGPA немає в ознаках моделі | прогноз з CGPA 0,1 і 10 однаковий | повзунок прибрано; невідоме поле → `422` |
| 5 | `POST /api/train` без захисту перезаписував модель | тест: без токена → `403` | `X-Admin-Token` = `ADMIN_TOKEN` (`hmac.compare_digest`); без змінної — вимкнено |
| 6 | витік даних: імпутер і скейлер навчались на всіх рядках до `train_test_split` | тест: медіани імпутера = медіани навчальної вибірки | `Pipeline`, навчений лише на навчальній частині; скейлер для лісу прибрано |
| 7 | pickle на 57 МБ читався з диска на кожен запит; перезапис на місці | тест: 3 прогнози — 0 читань з диска | кеш у пам'яті, атомарна заміна файлу, версія scikit-learn у файлі |
| 8 | `/api/groups` дописував колонку в кешований DataFrame | тест: після `/api/groups` у `/api/summary` є чужа колонка | `.copy()` |
| 9 | невідоме значення у стовпці → `NaN` у відповіді — невалідний JSON | тест зі строгим парсером JSON | `null`, `allow_nan=False` |
| 10 | папка `ui/pages/` — Streamlit автоматично додає 5 порожніх сторінок у меню | Streamlit AppTest | `ui/views/` |

А ще: пороги ризику в UI (0,3 / 0,5) не збігались з бекендом (0,5 / 0,75); `delta="vs overall"` — підпис без порівняння; `requirements.txt` з `==` під Python 3.11, без колес для 3.13. Усе виправлено й перелічено в [README проєкту](https://github.com/NikoriakViktot/PY-Course-Victor-Nikoriak-22-09-2026/blob/main/module_4/lessons/lesson_42_ai_dev_tools/depression_dashboard/README.md). Streamlit-дашборд після змін пройдено через `streamlit.testing.AppTest` проти живого бекенду: 5 розділів, форма прогнозу, жодного винятку.

## Довідник Claude Code (`CLAUDE_DOC.md` старого курсу)

Старий довідник датовано 2025-05. Ми попросили AI-агента звірити його з документацією. Агент знайшов справжні застарілі місця, але **сам помилився** щонайменше в чотирьох пунктах. Кожне твердження нижче тому перевірено ще раз — за `claude --help` установленої версії і за документацією.

| Старий довідник | Що не так | Тепер |
|---|---|---|
| таблиця моделей з повними назвами | застаріла за рік | аліаси `--model sonnet` / `opus` / `haiku`; поточний список — `/model` |
| ієрархія CLAUDE.md як «порядок пріоритету» (`/.claude/CLAUDE.md` для організації) | файли не перекривають, а доповнюють один одного; шлях організації — інший | розділ 6 |
| режими дозволів: 4 | з'явились `auto` і `dontAsk`, `default` у CLI зветься Manual | таблиця в розділі 11 |
| події hooks: 9 | понад 30 | головні — у розділі 10, решта — посилання |
| hook блокує `^rm -rf` і друкує JSON разом з `exit 2` | з кодом 2 JSON ігнорується, причина — у stderr; регулярний вираз легко обійти | розділ 10: hook — для якості, безпека — `deny` і sandbox |
| «Agent SDK» — приклад `anthropic.Anthropic().messages.create(...)` | це Client SDK (урок 43), а не Agent SDK | `claude-agent-sdk`, `query(...)` |
| CI: `claude --permission-mode bypassPermissions -p` у workflow | Claude Code не встановлено на раннері; повний автопілот на вмісті PR небезпечний | `anthropics/claude-code-action@v1` |
| `sudo apt install claude-code` одним рядком; автооновлення за замовчуванням — `stable` | apt/dnf/apk — лише після підключення підписаного репозиторію; канал за замовчуванням — `latest` | розділ 2 |
| посилання `docs.anthropic.com/en/docs/claude-code/…` | документація переїхала | `code.claude.com/docs/en/…` |
| — | не було: довіра до папки, `ask`-правила, `AGENTS.md`, `.claude/rules/`, skills з `paths` | розділи 6, 8, 11 |

**Помилки AI-аудитора (перевірено й відкинуто).** Агент стверджував, що імпорти `@path` у CLAUDE.md більше не підтримуються, що ключа `autoMemoryEnabled` немає, що поля skill `argument-hint` немає і що прапорців `--json-schema` і `--system-prompt` немає. Усе це є: `@path` і `autoMemoryEnabled` — у документації Memory, `argument-hint` — у документації Skills, прапорці — у `claude --help`. Урок той самий, що з кодом: **відповідь AI перевіряють за першоджерелом**.

## Довідник FastAPI (`lesson_34_fastapi_documentation.md`)

Перенесено як є. У старій шапці сторінки згадувався «перелік виправлень унизу», але самого переліку на сторінці не було.

## Бонус. Pandas: аналіз даних, графіки і Dash

**Дані WFP змінились.** З 2018 року WFP публікує ціни лише по регіональних ринках; «National Average» є тільки за 2014–2017. Ноутбуки старого курсу брали «ціну по Україні» як `admin1.isna()` і на свіжому файлі або падали (`KeyError: "['Bread (wheat)'] not in index"` у ноутбуці 3), або обривали графіки на 2017 році. Додано одну клітинку «🆕 National Average» у ноутбуки 1–3 (просте середнє по ринках); старі клітинки не змінено.

**`dash_API` на поточних бібліотеках:**

| Проблема | Причина | Виправлення |
|---|---|---|
| вкладка «Просторовий аналіз»: помилка 500 | `px.scatter_mapbox` прибрано в Plotly 6+ | `px.scatter_map` (MapLibre, без ключа Mapbox), `map_style=` замість `mapbox_style=` |
| попередження React у консолі | `html.Tr` безпосередньо в `html.Table` | `html.Table(html.Tbody(rows))` |

**Довідник «Аналіз даних»** (`data_analitic.md`) — виправлено лише огородження блоків коду, які ламали рендер діаграм.
