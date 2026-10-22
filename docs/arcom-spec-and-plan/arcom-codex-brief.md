# arcom — техническое задание для Codex

## Цель и ограничения

Создать современный, быстрый, mobile-first сайт-витрину **arcom** для продажи преимущественно б/у смартфонов, ноутбуков, ПК и аксессуаров. Источник новых карточек — публикации в Telegram-канале владельца. ИИ определяет, является ли публикация предложением о продаже, извлекает поля и при достаточной уверенности создаёт объявление; спорные случаи отправляются на проверку владельцу. Покупатели **не** регистрируются, **не** оформляют покупки и **не** оплачивают товары на сайте: только звонок или переход в Telegram продавца. Нужна закрытая админка одного владельца.

Канонический API-контракт: `arcom-openapi.yaml` (OpenAPI 3.0.3). Не придумывать новые публичные контракты вместо определённых в YAML. При обоснованном изменении API сначала обновлять спецификацию, затем тесты и реализацию.

## Стек

- **Frontend:** TypeScript + React + Tailwind CSS. Предпочтительно **Next.js App Router** для SSR/SEO страниц товаров; если нужен строго Vite, предусмотреть prerender/SSR для карточек и Open Graph. Компоненты: shadcn/ui, Lucide, аккуратные motion-эффекты.
- **Backend:** Java 21+ + Spring Boot 3.x (или совместимая стабильная версия), Spring Web, Validation, Spring Security, Spring Data JPA, Flyway.
- **Database:** PostgreSQL, расширение `pg_trgm` (неполное совпадение слов в названии/модели/бренде). Для MVP не запускать ElasticSearch.
- **Media:** S3-совместимое хранилище. При разработке — MinIO, в production — отдельный бакет с публичными оптимизированными изображениями. EXIF/геолокацию удалять.
- **Telegram realtime:** Bot API + webhook Spring Boot. В проде HTTPS, `secret_token`, сравнение chat ID, хранение `update_id`.
- **ИИ:** серверный адаптер GigaChat REST API, structured output (json_schema). Не обращаться из браузера напрямую к GigaChat.
- **Telegram history:** отдельная необязательная Python-утилита/worker с Telethon для одного первоначального backfill. Основной сервис оставлять на Java.
- **Deploy:** Docker Compose, HTTPS через Caddy или Nginx + ACME, контейнеры `web`, `api`, `postgres`, `worker` (если нужен) + подключённое S3.

## UI/UX

Эстетика: premium consumer electronics, узнаваемый минималистичный wordmark **arcom** (строчными буквами); светлая тема с графитовым текстом и одним выразительным синим акцентом, тёмная тема — опционально. Высокое качество типографики, крупные реальные фотографии товаров, достаточно воздуха; никаких агрессивных градиентов/автоматических слайдеров. Анимации ненавязчивые, поддержка `prefers-reduced-motion`.

- `/` — лендинг: hero, поисковая строка, CTA «Смотреть каталог», категории (смартфоны/ноутбуки/аксессуары), свежие объявления, преимущества магазина, способы связи, контакты.
- `/catalog` — каталог: поиск с debounce + состояние фильтров в URL, бренды, тип товара, цена, состояние, память/RAM, сортировка, количество результатов, скелетоны, пустые состояния, адаптивная сетка, мобильный bottom sheet фильтров.
- `/catalog/[slug]` — страница товара: фотографии с галереей и zoom, цена в рублях, статус (в наличии/бронь/продан), подтверждённые характеристики, оригинальное описание, CTA «Написать в Telegram» и «Позвонить», похожие товары. Никогда не показывать кнопку «Купить».
- `/admin/login` — вход владельца.
- `/admin` — dashboard количества объявлений, импорта и нуждающихся в проверке (агрегирование возможно на клиенте на первом этапе).
- `/admin/listings` и `/admin/listings/[id]` — редактирование, загрузка фотографий, публикация, архивирование и статус продажи.
- `/admin/imports` и `/admin/imports/[jobId]` — оригинальный пост рядом с извлечёнными полями, warnings и кнопками «Одобрить», «Игнорировать», «Повторить».
- `/admin/settings` — телефон, ссылки на Telegram/Avito, город, график, hero тексты.
- Страницы 404, error boundary, loading; семантическая HTML-разметка, keyboard navigation, alt у изображений.
- SEO: server-rendered HTML для товаров (или pre-render), title/description/canonical, `sitemap.xml`, `robots.txt`, Open Graph с обложкой, JSON-LD Product для карточек при корректно указанных полях.

## Модель данных PostgreSQL

Названия примерные, контракт API приоритетен.

1. `admin_users(id, username, password_hash, created_at)` — одна учётная запись, пароли BCrypt/Argon2; никогда не хранить plaintext.
2. `site_settings(id, name, phone, telegram_url, telegram_channel_url, avito_url, ... )` — одиночная запись.
3. `listings(id UUID PK, slug UNIQUE, title, description, category, brand, model, price_rub INTEGER, condition, availability, publication_status, source, specs JSONB, published_at, created_at, updated_at, import_job_id?, raw_source_text?, protected_fields TEXT[], ai_confidence?)`.
4. `listing_images(id UUID PK, listing_id FK, object_key, public_url, thumbnail_url, sort_order, width, height, alt)`.
5. `telegram_updates(update_id BIGINT UNIQUE, payload JSONB, received_at, processed_at)` — дедупликация доставки вебхуков.
6. `telegram_messages(id UUID PK, chat_id BIGINT, message_id BIGINT, media_group_id?, message_date, edited_at, original_text, payload JSONB, listing_id?, UNIQUE(chat_id, message_id))` — источник истины об исходных постах. Альбом из нескольких Telegram message IDs связывать общей media_group_id / id карточки.
7. `import_jobs(id UUID PK, telegram_message_id FK?, status, candidate JSONB, ai JSONB, listing_id?, retry_count, error_code, error_message, created_at, processed_at)`; для альбома можно ссылаться на `telegram_media_groups` или на основной `telegram_message`.
8. `backfill_jobs(id UUID PK, status, scanned, imported, skipped, created_at, completed_at, error_message)` — только если нужна история.

Индексы: `listings(publication_status, availability, published_at DESC)`, `listings(category, price_rub)`, `listings(brand)`, `listings USING GIN (lower(title) gin_trgm_ops)` (указать корректный SQL для PostgreSQL), `telegram_updates(update_id) UNIQUE`, `telegram_messages(chat_id,message_id) UNIQUE`, `listing_images(listing_id,sort_order)`. Избегать N+1 запросов и отдавать срезы без тяжёлых медиа в списках.

## Telegram ingestion — обязательно

1. Запрос `POST /api/v1/integrations/telegram/webhook` принимает JSON Telegram `Update`. Подписываемся только на `channel_post`, `edited_channel_post`. Секрет — header `X-Telegram-Bot-Api-Secret-Token`, дополнительная сверка numeric `chat.id` из allowlist.
2. Для повторного `update_id` быстро вернуть 200 без повторной обработки. **Не** делать HTTP-вызовы к GigaChat внутри webhook request; записать событие транзакционно и запланировать асинхронную обработку.
3. Telegram media albums приходят несколькими Update, объединять по `(chat_id, media_group_id)`, debounce около 3–5 секунд от последнего элемента (конфигурируемо), не плодить объявления для фото одного товара. Не применять debounce к отдельному посту.
4. Выбирать фото максимального размера из `photo[]`, получать через `getFile`, скачивать, ограничивать размер, проверять содержимое, конвертировать и сохранять в собственный S3. Не хранить Telegram CDN URL как основной URL для сайта.
5. Готовить из текста/подписи точные исходные данные. Если явно рекламный пост, новости, розыгрыш, опрос, поздравление или «куплю», не публиковать. Регулярки могут дешево отсеивать очевидные случаи, сложные — через GigaChat.
6. При вызове GigaChat запросить **строгий JSON** с `classification` (SALE/NOT_SALE/UNCERTAIN), `confidence`, `title`, `category`, `brand`, `model`, `priceRub`, `condition`, `specs`, `warnings`, `missingFields`. Запретить выдумывать цену/память/здоровье батареи/комплектацию. Извлекать **только из исходного поста**, неизвестное возвращать `null`.
7. Сервер валидирует JSON схему, валюту, допустимые категории, цену >0, разумные диапазоны и непротиворечивость полей. `confidence` — эвристика, не гарантия. Обязательно ручное ревью неполных/противоречивых карточек и в начале проекта — пробный период, когда всё поступает на ревью.
8. Полный валидный SALE с достаточной уверенностью (начать с 0.90, менять через env) публиковать; прочее REVIEW_REQUIRED или IGNORED. Нельзя автоматически публиковать NEW для б/у устройства только потому, что модель не увидела слова «б/у»; при отсутствии состояния — ручная проверка.
9. Новое редактирование **того же** сообщения обновляет связанный товар, но сохраняет поля `protected_fields`, если владелец изменил их вручную. Не создавать дубль. При повторном GigaChat для того же content hash не тратить токены.
10. Telegram Bot API не предоставляет надёжный поток удаления старых channel posts; удаление поста не считать автоматической продажей. Статус `SOLD` меняется админом или по однозначным действиям с известной карточкой.
11. Логи и ошибки: статусы QUEUED / PROCESSING / REVIEW_REQUIRED / PUBLISHED / IGNORED / FAILED; сохранение причины и retry с backoff. Не логировать ключи, сессии, коды Telegram, сырые персональные сообщения.

## Автоматизация истории канала

Bot API не предназначен для чтения истории канала, опубликованной до добавления бота. Опционально запустить **Telethon** c собственным `api_id` / `api_hash` из https://my.telegram.org, авторизовать учётную запись с законным доступом к каналу (лучше отдельную разрешённую учётную запись), получить прошлые сообщения через `iter_messages`, а затем передать данные в тот же ingestion pipeline. Держать `.session` файл и credentials **только** на сервере, вне git, защищать как пароль. Соблюдать лимиты Telegram и правила платформы. Никакой обход авторизации Telegram не нужен. Можно начать просто с разового CLI-скрипта, затем добавить административный backfill endpoint.

## GigaChat интеграция

- `GIGACHAT_AUTH_KEY`: Authorization Key из проекта, не краткоживущий access token. OAuth: POST `https://ngw.devices.sberbank.ru:9443/api/v2/oauth` с `Authorization: Basic <KEY>`, `RqUID: <uuid4>`, form `scope=GIGACHAT_API_PERS` (или подходящий для юрлица). Access token кэшировать до истечения срока (~30 минут), затем обновлять. Сам inference: `https://api.giga.chat/v1/chat/completions` или поддерживаемый SDK. Параметры структурированного ответа: `response_format.type=json_schema`, strict validation с учётом выбранной версии endpoint.
- Для новых клиентов с сентября 2026 уточнить доступную схему оплаты/модель в cloud.ru по официальной документации. Название модели вынести в env.
- TLS доверие: установить требуемые сертификаты, **не** отключать SSL verify в production.
- Обработка изображений ИИ опциональна. Сначала извлекать текст поста, а фото лишь хранить. Позже использовать vision при наличии тарифа и подходящей модели для верификации.

## Безопасность

- Публичные методы `GET /site`, `/categories`, `/catalog/facets`, `/listings` и страницы товаров доступны без логина. Только один admin login, никакой регистрации пользователей.
- Spring Security session cookie `JSESSIONID` (HttpOnly, Secure, SameSite=Lax), `GET /admin/auth/csrf` и `X-CSRF-TOKEN` на любые state-changing admin запросы. После login обязательно обновлять session ID. На frontend fetch с `credentials: 'include'`; проксировать `/api` под тем же доменом, чтобы не плодить CORS.
- Строгий rate limit на логин, server-side validation всех форм, безопасная обработка изображений (лимит размера/количества/формата), CSP/headers, HTTPS, privacy requirements.
- CSRF не применяется к Telegram webhook, но webhook защищён сильным отдельным секретом и `chat.id` whitelist.
- Secrets через Docker secrets или env переменные на сервере; gitignore для `.env`, session файлов, S3 keys.
- Хранить резервные копии PostgreSQL и файлов; проверить восстановление. Не раскрывать внутренние stack traces в API.

## Порядок разработки в Codex

### Этап 1 — scaffolding
- Создать monorepo (`frontend`, `backend`, `tools/telegram-backfill`) и docker-compose для postgres/minio.
- Импортировать `arcom-openapi.yaml`; сформировать API клиент TS через `openapi-typescript` (или Orval), настроить типы, линтер, formatter.
- Создать Spring Boot + Flyway миграции, Testcontainers для интеграционных тестов.

### Этап 2 — реальный каталог без Telegram
- CRUD admin и API каталога строго по OpenAPI.
- Реалистичные seed-товары: несколько б/у телефонов, ноутбук, аксессуары; тестовые фиктивные изображения/данные, без реальных контактных данных.
- Каталог, фильтры, лендинг, детальная страница. Реализовать все состояния загрузки, доступность и responsive.

### Этап 3 — административная панель
- Авторизация владельца, CSRF/session, защищённые маршруты, управление карточками, загрузка/порядок изображений, статус SOLD/RESERVED.
- Формы валидировать с теми же ограничениями, что и API.

### Этап 4 — Telegram
- `setWebhook`, проверка секрета и канала, дедуп, обработка альбомов, загрузка/оптимизация фото, создаваться должны ImportJob.
- Тестировать webhook с заранее сохранёнными fixture JSON от Telegram, без live-токена.

### Этап 5 — GigaChat и модерация
- Интерфейс `ListingAiExtractor` + заглушка для тестов, затем реальный GigaChat адаптер.
- Structured output, валидация, ручное review, retry, отсутствие выдуманных данных, защита ручных правок.
- Сначала 100% ручная проверка на 30–50 реальных сообщениях, затем включить controlled auto-publish.

### Этап 6 — SEO, качество, деплой
- Metadata, sitemap, robots, Lighthouse, адаптивные фотографии, правильная русская локаль.
- Проверить бэкапы, HTTPS, мониторинг, rate limits, health endpoint, запуск по Docker Compose.
- Исторический backfill через Telethon сделать отдельным этапом после стабильного live-импорта.

## Приёмочные критерии

- Посетитель без авторизации может быстро найти телефон по цене/бренду/памяти, увидеть его фотографии и перейти на контакт продавца.
- Покупатель не встречает signup, checkout, shopping cart или платёжные формы.
- Одна Telegram-публикация с 5 фото порождает **ровно одно** объявление с 5 картинками; повтор вебхука не создаёт дубль.
- Рекламный/новостной пост не публикуется как товар.
- Пропущенная или двусмысленная цена **никогда** не выдумывается и автоматически не публикуется.
- Исправленная владельцем цена не затирается последующим edit из Telegram.
- После статуса SOLD товар остаётся доступен по URL (с бейджем «Продано»), но по умолчанию исчезает из выдачи активного каталога.
- API администратора требует сессию, изменяющие маршруты требуют CSRF; webhook требует секрет и разрешённый канал.
- Работают поиск, пагинация, обновления товаров, дедуп, backoff, тесты и восстановление после перезапуска.

## Стартовый промпт Codex

> Прочитай `arcom-codex-brief.md` и `arcom-openapi.yaml`. Реализуй arcom по этапам, без корзины, оплаты и аккаунтов покупателей. Начни с этапа 1 и этапа 2: monorepo, Docker Compose, PostgreSQL/Flyway, backend Spring Boot, типизированный клиент OpenAPI и качественные страницы React/Tailwind. Не придумывай секреты; все интеграции с Telegram и GigaChat пока за интерфейсами-заглушками. После изменений запускай линтеры, сборки и тесты и сообщай конкретные результаты. Соблюдай API-контракт. На следующем этапе реализуем админку и ingestion.
