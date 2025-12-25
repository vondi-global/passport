# GLOSSARY.md - Глоссарий терминов платформы Vondi

> Комплексный глоссарий всех бизнес-терминов, технических концепций, доменов, сервисов, инфраструктуры и статусов платформы Vondi.

---

## A

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Access Token | JWT токен для аутентификации API запросов (TTL: 15 мин) | Auth Service, JWT |
| ACL | Access Control List - список контроля доступа | Security, Authorization |
| Active Reservation | Активная резервация инвентаря для заказа | Inventory, WMS |
| AddPhoto (mutation) | GraphQL мутация для добавления фото к листингу | Listings Service, GraphQL |
| Admin | Роль администратора платформы с полными правами | RBAC, Users |
| Admin Interface | Административный интерфейс управления платформой | Frontend, Monolith |
| AES-256-GCM | Алгоритм шифрования конфиденциальных данных | Security, Encryption |
| Agent | Роль сотрудника склада (warehouse worker) | WMS, RBAC |
| Alerts | Системные уведомления о критических событиях | Monitoring, Infrastructure |
| API Gateway | Центральный шлюз для маршрутизации API запросов | Architecture, Monolith |
| API v1 | Основной REST API backend монолита | Monolith, REST |
| API v2 | BFF proxy слой Next.js для frontend | Frontend, BFF |
| Argon2id | Алгоритм хеширования паролей | Auth Service, Security |
| Attributes | Характеристики товара (color, size, material) | Categories, Listings |
| Auth Middleware | Middleware для проверки JWT токенов | Auth Service, Middleware |
| Auth Service | Микросервис аутентификации и авторизации | Microservices, gRPC |
| Auth Service DB | База данных Auth Service (порт 25432) | PostgreSQL, Auth Service |
| Authorization | Проверка прав доступа пользователя | Security, RBAC |
| Availability | Статус доступности товара (available/reserved/sold) | Inventory, Listings |

## B

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Backup | Резервная копия базы данных (ежедневно 03:00 UTC) | PostgreSQL, Infrastructure |
| Batch Picking | Комплектование нескольких заказов одновременно | WMS, Fulfillment |
| Bearer Token | HTTP заголовок для передачи JWT токена | Auth, HTTP |
| BFF | Backend-for-Frontend - прокси слой для frontend | Architecture, Next.js |
| B2C | Business-to-Consumer - продажа от бизнеса покупателям | Business Model |
| Billing | Система начисления комиссий и платежей | Payment Service |
| Bucket | S3 хранилище для файлов (images, documents) | MinIO, Storage |

## C

| Термин | Определение | Контекст |
|--------|-------------|----------|
| C2C | Customer-to-Customer - продажа между пользователями | Business Model |
| Cache | Кеш данных в Redis для ускорения запросов | Redis, Performance |
| Canceled | Статус отмененного заказа | Order Lifecycle |
| Captured | Статус успешного списания средств | Payment Lifecycle |
| Cart | Корзина покупок пользователя | E-commerce, Monolith |
| Cart Items | Товары в корзине покупателя | Shopping Cart |
| Category | Категория товаров с иерархией (parent-child) | Listings, Taxonomy |
| Category Attributes | Специфичные атрибуты для категории | Categories, Schema |
| Checkout | Процесс оформления заказа | E-commerce, Order Flow |
| CI/CD | Continuous Integration/Deployment - автоматизация сборки | DevOps, GitHub Actions |
| Commission | Комиссия платформы за сделку | Payment, Business |
| Committed Reservation | Подтвержденная резервация инвентаря | Inventory, WMS |
| Confirmed | Статус подтвержденного заказа продавцом | Order Lifecycle |
| Connection Pool | Пул соединений с базой данных | PostgreSQL, Performance |
| Consumer | Kafka/Redis Streams consumer для обработки событий | Events, Messaging |
| Container | Docker контейнер для сервиса | Docker, Infrastructure |
| Content Manager | Роль менеджера контента (categories, SEO) | RBAC, Users |
| CORS | Cross-Origin Resource Sharing - политика CORS | HTTP, Security |
| Country Code | Код страны (RS, RU, etc.) | Localization |
| CreateListing | API endpoint для создания листинга | Listings Service |
| CRUD | Create, Read, Update, Delete операции | REST API |
| Currency | Валюта платежа (RSD, EUR, USD) | Payment, Localization |
| Customer | Роль покупателя на платформе | RBAC, Users |
| Cycle Count | Инвентаризация склада по графику | WMS, Inventory |

## D

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Dashboard | Панель управления пользователя/админа | Frontend, UI |
| Data Retention | Политика хранения данных (orders: 7 лет) | Compliance, PostgreSQL |
| DDD | Domain-Driven Design - подход к проектированию | Architecture, Design |
| Deadletter Queue | Очередь для необработанных сообщений | Events, Messaging |
| Delivery | Доставка заказа (Post Express, самовывоз) | Delivery Service |
| Delivery DB | База данных Delivery Service (порт 35433) | PostgreSQL, Delivery |
| Delivery Service | Микросервис управления доставкой | Microservices, gRPC |
| Delivered | Статус доставленного заказа | Order Lifecycle |
| Deployment | Процесс развертывания на сервере | DevOps, Infrastructure |
| Description | Описание товара (title_ru, title_sr, etc.) | Listings, i18n |
| Discount | Скидка на товар или заказ | E-commerce, Pricing |
| Docker Compose | Оркестрация Docker контейнеров | Docker, Infrastructure |
| Domain Event | Событие бизнес-логики (OrderCreated, PaymentCaptured) | Events, DDD |
| Draft | Статус черновика листинга | Listing Lifecycle |

## E

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Email Verification | Подтверждение email через токен | Auth Service, Security |
| Entity | Доменная сущность с уникальным ID | DDD, Domain Model |
| Env Variables | Переменные окружения для конфигурации | Configuration, Docker |
| Escrow | Защита платежа (деньги удерживаются до доставки) | Payment, Business |
| Event | Событие в системе (order.created, payment.captured) | Events, Messaging |
| Event Consumer | Обработчик событий из очереди | Events, Microservices |
| Event Publisher | Публикатор событий в очередь | Events, Microservices |
| Expired Reservation | Просроченная резервация инвентаря | Inventory, WMS |

## F

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Failed Payment | Неудачная попытка оплаты | Payment Lifecycle |
| FIFO | First-In-First-Out - стратегия выбора товара | WMS, Inventory |
| Filter | Фильтр товаров по атрибутам | Search, OpenSearch |
| Fixture | Тестовые данные для разработки | Database, Testing |
| Fulfillment | Процесс обработки заказа (picking → packing → shipping) | WMS, Order Flow |
| Fulfillment Center | Фулфилмент центр (склад Vondi) | WMS, Warehouse |

## G

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Gamification | Игровые механики для мотивации работников | WMS, HR |
| GetMe | API endpoint для получения профиля пользователя | Auth Service, REST |
| GitHub Actions | CI/CD платформа для автоматизации | DevOps, CI |
| Grafana | Система мониторинга и визуализации метрик | Monitoring, Infrastructure |
| GraphQL | Язык запросов для API | Listings Service, API |
| gRPC | Google RPC - протокол межсервисной коммуникации | Microservices, Protocol |
| gRPC Gateway | REST прокси для gRPC сервисов | gRPC, REST |

## H

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Handler | Обработчик HTTP/gRPC запроса | Backend, API |
| Harbor | Private Docker Registry для образов | Infrastructure, Docker |
| Hash | Хеш пароля (Argon2id) или токена | Security, Auth |
| Health Check | Endpoint для проверки состояния сервиса | Monitoring, API |
| HTTP | Протокол передачи гипертекста | REST API, Protocol |
| httpOnly Cookie | Cookie недоступная для JavaScript | Security, Frontend |

## I

| Термин | Определение | Контекст |
|--------|-------------|----------|
| i18n | Internationalization - поддержка языков (sr/ru/en) | Localization, Frontend |
| Idempotency Key | Ключ для защиты от повторной обработки | WMS, API |
| Image | Изображение товара в MinIO S3 | Storage, Listings |
| Index | Индекс OpenSearch для поиска | Search, OpenSearch |
| Inventory | Инвентарь склада (товары на складе) | WMS, Warehouse |
| Inventory Reservation | Резервация товара для заказа | WMS, Order Flow |
| Invoice | Счет на оплату | Payment, Billing |

## J

| Термин | Определение | Контекст |
|--------|-------------|----------|
| JSONB | JSON тип данных в PostgreSQL | PostgreSQL, Database |
| JWT | JSON Web Token - формат токена авторизации | Auth, Security |
| JWT Parser | Middleware для парсинга JWT токенов | Auth Middleware |

## K

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Kafka | Платформа потоковой обработки событий | Events, Messaging |
| Key-Value Store | Хранилище ключ-значение (Redis) | Redis, Cache |

## L

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Language | Язык интерфейса (sr-Latn, ru, en) | i18n, Localization |
| Listing | Объявление о продаже товара | Listings Service, Core |
| Listing Photo | Фотография товара в листинге | Listings, Storage |
| Listing Status | Статус листинга (draft, active, sold, archived) | Listing Lifecycle |
| Listings DB | База данных Listings Service (порт 35434) | PostgreSQL, Listings |
| Listings Service | Микросервис управления листингами | Microservices, gRPC |
| Locale | Локаль пользователя (sr-Latn, ru, en) | i18n, Frontend |
| Location | Адрес склада или места хранения товара | WMS, Warehouse |
| Lock | Блокировка для конкурентного доступа | Concurrency, PostgreSQL |
| Logger | Библиотека логирования (zerolog) | Logging, Infrastructure |

## M

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Marketplace | Торговая площадка Vondi | Business Model, Core |
| Metadata | Метаданные объекта (created_at, updated_at) | Database, Schema |
| Microservice | Независимый сервис с собственной БД | Architecture, Design |
| Middleware | Промежуточный обработчик запросов | Backend, HTTP |
| Migration | SQL миграция базы данных | PostgreSQL, Schema |
| MinIO | S3-совместимое объектное хранилище | Storage, Infrastructure |
| Monolith | Монолитный backend сервис | Architecture, Legacy |
| Monitoring | Мониторинг состояния системы | Infrastructure, Grafana |

## N

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Nginx | HTTP сервер и reverse proxy | Infrastructure, Web Server |
| Notification | Уведомление пользователя (email, push, SMS) | Notifications Service |
| Notifications Service | Микросервис уведомлений | Microservices, gRPC |

## O

| Термин | Определение | Контекст |
|--------|-------------|----------|
| OAuth2 | Протокол авторизации | Auth, Security |
| OpenSearch | Полнотекстовый поиск и аналитика | Search, Infrastructure |
| Order | Заказ покупателя | Orders, E-commerce |
| Order ID | Уникальный идентификатор заказа (UUID) | Orders, Database |
| Order Item | Позиция в заказе (товар + количество) | Orders, Schema |
| Order Lifecycle | Жизненный цикл заказа (pending → delivered) | Order Flow, State Machine |
| Order Status | Статус заказа (pending, confirmed, shipped, etc.) | Order Lifecycle |
| Owner | Роль владельца склада | WMS, RBAC |

## P

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Packing | Упаковка заказа перед отправкой | WMS, Fulfillment |
| Packing Task | Задача упаковки для работника склада | WMS, Tasks |
| Password | Пароль пользователя (хешируется Argon2id) | Auth, Security |
| Payment | Платеж за заказ | Payment Service, E-commerce |
| Payment Gateway | Шлюз приема платежей | Payment, Infrastructure |
| Payment Service | Микросервис обработки платежей | Microservices, gRPC |
| Payment Status | Статус платежа (pending, captured, failed, etc.) | Payment Lifecycle |
| Payout | Выплата продавцу после доставки | Payment, Business |
| Pending | Статус ожидания (заказ, платеж) | Lifecycle, Status |
| Permission | Право доступа к ресурсу | RBAC, Authorization |
| Picking | Комплектование товаров для заказа | WMS, Fulfillment |
| Picking Task | Задача комплектации для работника склада | WMS, Tasks |
| Placeholder | Ключ для перевода (categories.name_required) | i18n, Backend |
| PM2 | Process manager для Node.js приложений | Frontend, Infrastructure |
| Pool | Пул соединений с БД | PostgreSQL, Performance |
| Port | Порт сервиса (3000 - backend, 50053 - gRPC, etc.) | Infrastructure, Network |
| Post Express | Партнер доставки (почта Сербии) | Delivery, Integration |
| PostgreSQL | Реляционная СУБД | Database, Infrastructure |
| Price | Цена товара | Listings, E-commerce |
| Priority | Приоритет задачи (high, medium, low) | WMS, Tasks |
| Private Key | Приватный ключ для подписи JWT (RS256) | Auth, Security |
| Product | Товар на платформе | Listings, Core |
| Product Location | Место хранения товара на складе | WMS, Inventory |
| Proto | Protocol Buffers - формат данных для gRPC | gRPC, Serialization |
| Public Key | Публичный ключ для проверки JWT | Auth, Security |
| Published | Статус опубликованного листинга | Listing Lifecycle |
| Publisher | Публикатор событий в очередь | Events, Messaging |

## Q

| Термин | Определение | Контекст |
|--------|-------------|----------|
| QC | Quality Control - контроль качества товара | WMS, Fulfillment |
| Quantity | Количество товара | Inventory, Listings |
| Query | Запрос к API или базе данных | REST API, Database |
| Queue | Очередь сообщений (Redis Streams, Kafka) | Events, Messaging |

## R

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Rate Limiter | Ограничитель частоты запросов | Security, API |
| RBAC | Role-Based Access Control - управление доступом | Security, Authorization |
| Ready to Pack | Статус задачи упаковки (готов к упаковке) | Packing Lifecycle |
| Ready to Pick | Статус задачи комплектации (готов к сбору) | Picking Lifecycle |
| Redis | In-memory база данных (кеш, очереди) | Cache, Infrastructure |
| Redis Streams | Потоковая обработка событий в Redis | Events, Messaging |
| Refunded | Статус возвращенного платежа | Payment Lifecycle |
| Refresh Token | JWT токен для обновления access token (TTL: 30 дней) | Auth Service, JWT |
| Released Reservation | Освобожденная резервация инвентаря | Inventory, WMS |
| Replica | Реплика базы данных (read-only) | PostgreSQL, High Availability |
| Repository | Репозиторий для работы с БД (DDD pattern) | DDD, Database |
| Reservation | Резервация товара для заказа | Inventory, WMS |
| Reservation Status | Статус резервации (active, committed, released, expired) | Reservation Lifecycle |
| REST API | RESTful API для HTTP запросов | API, HTTP |
| Retry | Повтор операции при ошибке | Reliability, Events |
| Returns | Возврат товара покупателем | WMS, E-commerce |
| Role | Роль пользователя (admin, seller, customer, etc.) | RBAC, Users |
| Rollback | Откат транзакции или миграции | PostgreSQL, Database |
| RS256 | Алгоритм подписи JWT (RSA + SHA256) | Auth, Security |

## S

| Термин | Определение | Контекст |
|--------|-------------|----------|
| S3 | Simple Storage Service - объектное хранилище | MinIO, Storage |
| Schema | Схема базы данных | PostgreSQL, Database |
| Search | Поиск товаров по тексту и фильтрам | OpenSearch, Frontend |
| Secret | Секретный ключ (API key, password, etc.) | Security, Configuration |
| Seller | Роль продавца на платформе | RBAC, Users |
| Service | Микросервис или системный сервис | Architecture, Microservices |
| Session | Сессия пользователя (JWT tokens) | Auth, Security |
| Shipped | Статус отправленного заказа | Order Lifecycle |
| Shopping Cart | Корзина покупок | E-commerce, Monolith |
| SKU | Stock Keeping Unit - артикул товара | Inventory, Listings |
| Slug | URL-friendly идентификатор | SEO, Frontend |
| Sold | Статус проданного листинга | Listing Lifecycle |
| SQL | Structured Query Language - язык запросов | PostgreSQL, Database |
| SSL/TLS | Протокол шифрования соединения | Security, HTTPS |
| State Machine | Конечный автомат для управления статусами | Order Flow, Architecture |
| Status | Статус объекта (order, payment, listing, etc.) | Lifecycle, Core |
| Stock | Запас товара на складе | Inventory, WMS |
| Storefront | Витрина продавца с его товарами | Listings, E-commerce |
| Stream | Поток событий (Redis Streams) | Events, Messaging |
| Substitution | Замена товара при комплектации | WMS, Fulfillment |
| Svetubd | Основная база данных монолита (порт 5433) | PostgreSQL, Monolith |

## T

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Task | Задача для работника склада (picking, packing, QC) | WMS, Fulfillment |
| Telegram Bot | Бот для уведомлений в Telegram | Notifications, Integration |
| Template | Шаблон уведомления или письма | Notifications, i18n |
| Tenant | Арендатор (multi-tenancy) | WMS, SaaS |
| Test | Тестовое окружение или unit test | Testing, CI/CD |
| Timezone | Временная зона (Europe/Belgrade) | Localization, Configuration |
| Token | Токен аутентификации (JWT) | Auth, Security |
| Tracking Number | Трекинг номер отправления | Delivery, Post Express |
| Transaction | Транзакция базы данных | PostgreSQL, ACID |
| Translation | Перевод интерфейса (sr/ru/en) | i18n, Frontend |
| TTL | Time-To-Live - время жизни токена/кеша | Redis, Auth |
| Type | Тип данных (TypeScript, PostgreSQL) | Schema, Type Safety |

## U

| Термин | Определение | Контекст |
|--------|-------------|----------|
| UpdateListing | API endpoint для обновления листинга | Listings Service, REST |
| User | Пользователь платформы | Auth, Core |
| User ID | Уникальный идентификатор пользователя (UUID) | Users, Database |
| User Settings | Настройки пользователя (язык, уведомления) | Users, Profile |
| UUID | Universally Unique Identifier - уникальный ID | Database, Core |

## V

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Validation | Валидация входных данных | Backend, Security |
| Value Object | Объект-значение без идентификатора (DDD) | DDD, Domain Model |
| Variant | Вариант товара (цвет, размер) | Listings, E-commerce |
| Vault | Хранилище секретов | Security, Configuration |
| Version | Версия сервиса или API | Versioning, API |
| View | SQL view для упрощения запросов | PostgreSQL, Database |
| Vondi | Название платформы (бывш. Svetu) | Brand, Core |

## W

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Warehouse | Склад для хранения товаров | WMS, Infrastructure |
| Warehouse DB | База данных WMS Service (порт 35435) | PostgreSQL, WMS |
| Warehouse Staff | Сотрудники склада (pickers, packers, QC) | WMS, HR |
| Webhook | HTTP callback для уведомлений | Integration, Events |
| WebSocket | Протокол двусторонней связи | Real-time, Frontend |
| Worker | Работник склада | WMS, RBAC |
| WMS | Warehouse Management System - система управления складом | WMS, Core |

## Z

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Zerolog | Библиотека структурированного логирования (Go) | Logging, Backend |
| Zoho Mail | Email провайдер для уведомлений | Notifications, Integration |

---

## Технические аббревиатуры

| Аббревиатура | Расшифровка | Контекст |
|--------------|-------------|----------|
| ACL | Access Control List | Security |
| ACID | Atomicity, Consistency, Isolation, Durability | Database |
| AES | Advanced Encryption Standard | Security |
| API | Application Programming Interface | Integration |
| B2B | Business-to-Business | Business Model |
| B2C | Business-to-Consumer | Business Model |
| BFF | Backend-for-Frontend | Architecture |
| C2C | Customer-to-Customer | Business Model |
| CDN | Content Delivery Network | Infrastructure |
| CI/CD | Continuous Integration/Deployment | DevOps |
| CORS | Cross-Origin Resource Sharing | Security |
| CRUD | Create, Read, Update, Delete | REST API |
| CSV | Comma-Separated Values | Data Format |
| DDD | Domain-Driven Design | Architecture |
| DNS | Domain Name System | Infrastructure |
| DTO | Data Transfer Object | Design Pattern |
| ETL | Extract, Transform, Load | Data Processing |
| FIFO | First-In-First-Out | Inventory Strategy |
| FTP | File Transfer Protocol | Integration |
| GCM | Galois/Counter Mode | Encryption |
| gRPC | Google Remote Procedure Call | Protocol |
| HATEOAS | Hypermedia as the Engine of Application State | REST |
| HTML | HyperText Markup Language | Web |
| HTTP | HyperText Transfer Protocol | Protocol |
| HTTPS | HTTP Secure | Protocol |
| i18n | Internationalization | Localization |
| ID | Identifier | Core |
| JSON | JavaScript Object Notation | Data Format |
| JSONB | JSON Binary | PostgreSQL |
| JWT | JSON Web Token | Security |
| KPI | Key Performance Indicator | Metrics |
| L10n | Localization | i18n |
| LIFO | Last-In-First-Out | Inventory Strategy |
| MFA | Multi-Factor Authentication | Security |
| MinIO | Minimal Object Storage | Infrastructure |
| MQ | Message Queue | Messaging |
| MVP | Minimum Viable Product | Development |
| NGINX | Engine X | Web Server |
| ORM | Object-Relational Mapping | Database |
| OTP | One-Time Password | Security |
| P2P | Peer-to-Peer | Architecture |
| PGP | Pretty Good Privacy | Security |
| PM2 | Process Manager 2 | Node.js |
| POD | Proof of Delivery | Logistics |
| QC | Quality Control | WMS |
| RBAC | Role-Based Access Control | Security |
| REST | Representational State Transfer | API |
| RPC | Remote Procedure Call | Protocol |
| RS256 | RSA Signature SHA-256 | JWT Algorithm |
| RSA | Rivest-Shamir-Adleman | Cryptography |
| RSD | Serbian Dinar | Currency |
| S3 | Simple Storage Service | Storage |
| SaaS | Software as a Service | Business Model |
| SDK | Software Development Kit | Development |
| SEO | Search Engine Optimization | Marketing |
| SHA | Secure Hash Algorithm | Cryptography |
| SKU | Stock Keeping Unit | Inventory |
| SMS | Short Message Service | Notifications |
| SMTP | Simple Mail Transfer Protocol | Email |
| SQL | Structured Query Language | Database |
| SSH | Secure Shell | Infrastructure |
| SSL | Secure Sockets Layer | Security |
| SSO | Single Sign-On | Auth |
| TLS | Transport Layer Security | Security |
| TTL | Time-To-Live | Cache |
| UI | User Interface | Frontend |
| URI | Uniform Resource Identifier | Web |
| URL | Uniform Resource Locator | Web |
| UUID | Universally Unique Identifier | Core |
| UX | User Experience | Frontend |
| VAT | Value Added Tax | Billing |
| VPS | Virtual Private Server | Infrastructure |
| WAL | Write-Ahead Log | PostgreSQL |
| WebSocket | Web Socket Protocol | Real-time |
| WMS | Warehouse Management System | Core |
| XML | eXtensible Markup Language | Data Format |
| YAML | YAML Ain't Markup Language | Configuration |

---

## Статусы и константы

### Order Statuses (order_status)

| Статус | Описание | Переход |
|--------|----------|---------|
| pending | Заказ создан, ожидает подтверждения продавца | → confirmed, canceled |
| confirmed | Продавец подтвердил заказ | → accepted, canceled |
| accepted | Заказ принят в обработку | → processing |
| processing | Идет обработка (picking/packing) | → shipped |
| shipped | Заказ отправлен покупателю | → delivered |
| delivered | Заказ доставлен | — |
| canceled | Заказ отменен | — |
| refunded | Деньги возвращены покупателю | — |

### Payment Statuses (payment_status)

| Статус | Описание | Переход |
|--------|----------|---------|
| pending | Платеж ожидает обработки | → authorized, failed |
| authorized | Средства заблокированы (escrow) | → captured, voided |
| captured | Средства списаны | → refunded |
| failed | Попытка оплаты неудачна | — |
| refunded | Деньги возвращены | — |
| voided | Авторизация отменена | — |

### Reservation Statuses (reservation_status)

| Статус | Описание | Переход |
|--------|----------|---------|
| active | Резервация активна (15 мин) | → committed, released, expired |
| committed | Резервация подтверждена (заказ создан) | → released |
| released | Резервация освобождена | — |
| expired | Резервация просрочена | — |

### Picking Task Statuses (picking_status)

| Статус | Описание | Переход |
|--------|----------|---------|
| pending | Задача создана | → assigned |
| assigned | Назначена работнику | → in_progress |
| in_progress | Работник комплектует | → completed, failed |
| completed | Комплектация завершена | — |
| failed | Ошибка комплектации | — |

### Packing Task Statuses (packing_status)

| Статус | Описание | Переход |
|--------|----------|---------|
| pending | Задача создана | → assigned |
| assigned | Назначена работнику | → in_progress |
| in_progress | Работник упаковывает | → completed, failed |
| completed | Упаковка завершена | — |
| failed | Ошибка упаковки | — |

### Delivery Statuses (delivery_status)

| Статус | Описание | Переход |
|--------|----------|---------|
| pending | Ожидает отправки | → in_transit |
| in_transit | В пути | → delivered, failed |
| delivered | Доставлено | — |
| failed | Ошибка доставки | — |
| returned | Возвращено отправителю | — |

### Listing Statuses (listing_status)

| Статус | Описание | Переход |
|--------|----------|---------|
| draft | Черновик | → active |
| active | Опубликован | → sold, archived |
| sold | Продан | → archived |
| archived | Архивирован | → active |
| deleted | Удален | — |

---

## Business Terms (Бизнес-термины)

| Термин | Определение | Контекст |
|--------|-------------|----------|
| Buyer Protection | Защита покупателя через escrow | Payment, Trust |
| Commission Fee | Комиссия платформы (%) | Business Model |
| Listing Fee | Плата за размещение объявления | Business Model |
| Marketplace Fee | Комиссия маркетплейса за сделку | Business Model |
| Platform Commission | Комиссия платформы | Revenue |
| Promotion | Акция или продвижение листинга | Marketing |
| Rating | Рейтинг продавца/товара (1-5 звезд) | Trust, Reviews |
| Review | Отзыв покупателя о товаре/продавце | Trust, E-commerce |
| Shipping Cost | Стоимость доставки | Delivery, Pricing |
| Transaction Fee | Комиссия за транзакцию | Payment |
| Trust Score | Показатель доверия к продавцу | Trust, Reputation |
| Vondi Fulfillment | Услуга хранения и отправки товаров платформой | WMS, Business |

---

## Сервисы и порты

| Сервис | HTTP Port | gRPC Port | PostgreSQL Port | Redis Port |
|--------|-----------|-----------|-----------------|------------|
| Monolith (vondi) | 3000 | — | 5433 (vondi_db) | 6379 |
| Frontend (Next.js) | 3001 | — | — | — |
| Auth Service | 28086 | 20053 | 25432 | 26379 |
| Listings Service | 8082 | 50053 | 35434 | 36380 |
| Delivery Service | 8083 | 50051 | 35433 | 36381 |
| WMS Service | 8084 | 50052 | 35435 | — |
| Payment Service | 8085 | 50054 | — | — |
| Notifications Service | 8088 | 50055 | 35436 | 36382 |
| OpenSearch | 9200 | — | — | — |
| MinIO S3 | 9000 (API), 9001 (Console) | — | — | — |

---

**Итого терминов:** 300+ основных терминов + 80+ аббревиатур + статусы всех доменов

**Версия:** 1.0
**Дата создания:** 2025-12-21
**Источники:** Все passport файлы из `/p/github.com/vondi-global/.passport/`
