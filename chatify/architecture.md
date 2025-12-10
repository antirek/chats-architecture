# Архитектура Chatify

## Обзор проекта

**Chatify** — это Laravel-пакет для реализации системы обмена сообщениями один-на-один в реальном времени. Пакет использует Pusher для real-time коммуникации и предоставляет полнофункциональный мессенджер с поддержкой файлов, эмодзи, статусов активности и других функций.

## Ключевые понятия

### Основные компоненты

1. **ChatifyMessenger** — главный класс-фасад, предоставляющий API для работы с сообщениями, пользователями и Pusher
2. **MessagesController** — контроллер для обработки HTTP-запросов (веб и API)
3. **ChMessage** — модель Eloquent для сообщений
4. **ChFavorite** — модель для избранных контактов
5. **Pusher** — внешний SaaS-сервис для real-time коммуникации через WebSocket

### Pusher — что это?

**Pusher** — это внешний облачный сервис (SaaS), предоставляющий инфраструктуру для real-time коммуникации через WebSocket. Это не компонент Laravel, а отдельный сервис, который требует:

1. **Регистрации** на сайте [pusher.com](https://pusher.com)
2. **Создания приложения** в панели управления Pusher
3. **Получения учетных данных**:
   - `PUSHER_APP_KEY` — публичный ключ приложения
   - `PUSHER_APP_SECRET` — секретный ключ (используется на сервере)
   - `PUSHER_APP_ID` — ID приложения
   - `PUSHER_APP_CLUSTER` — регион кластера (например, `mt1`, `eu`, `ap-southeast-1`)

4. **Настройки в `.env` файле**:
   ```env
   PUSHER_APP_KEY=your_key
   PUSHER_APP_SECRET=your_secret
   PUSHER_APP_ID=your_app_id
   PUSHER_APP_CLUSTER=mt1
   ```

**Альтернативы Pusher:**
- Можно использовать собственный WebSocket-сервер (например, Laravel Echo Server, Soketi)
- Или другие сервисы: Ably, PubNub, Firebase Realtime Database

**Почему Pusher?**
- Простая интеграция
- Масштабируемость
- Надежность
- Поддержка приватных и presence-каналов
- Бесплатный тарифный план для разработки

### База данных

- **ch_messages** — таблица сообщений
  - `id` (UUID) — уникальный идентификатор
  - `from_id` — ID отправителя
  - `to_id` — ID получателя
  - `body` — текст сообщения (до 5000 символов)
  - `attachment` — JSON с информацией о вложении
  - `seen` — флаг прочтения (boolean)
  - `created_at`, `updated_at` — временные метки

- **ch_favorites** — таблица избранных контактов
  - `id` (UUID) — уникальный идентификатор
  - `user_id` — ID пользователя
  - `favorite_id` — ID избранного контакта
  - `created_at`, `updated_at` — временные метки

### Маршруты

**Web-маршруты** (префикс: `/chatify`):
- `GET /` — главная страница мессенджера
- `POST /sendMessage` — отправка сообщения
- `POST /fetchMessages` — получение сообщений
- `POST /makeSeen` — отметка сообщений как прочитанных
- `POST /chat/auth` — аутентификация Pusher
- `GET /getContacts` — получение списка контактов
- `POST /star` — добавление/удаление из избранного
- `GET /search` — поиск пользователей
- `POST /deleteConversation` — удаление беседы
- `POST /deleteMessage` — удаление сообщения
- `GET /download/{fileName}` — скачивание вложений

**API-маршруты** (префикс: `/chatify/api`):
- Аналогичные маршруты, но возвращающие JSON вместо HTML

### Pusher каналы

- `private-chatify.{user_id}` — приватный канал для каждого пользователя
- `presence-activeStatus` — presence-канал для отслеживания статуса активности

### События Pusher

- `messaging` — событие нового сообщения
- `pusher:member_added` — пользователь подключился
- `pusher:member_removed` — пользователь отключился

## Общая архитектура системы

```mermaid
graph TB
    subgraph "Клиент (Browser)"
        UI[Пользовательский интерфейс]
        JS[JavaScript код<br/>code.js, utils.js]
        PusherClient[Pusher Client]
    end
    
    subgraph "Laravel Application"
        Routes[Маршруты<br/>web.php, api.php]
        Controller[MessagesController]
        Chatify[ChatifyMessenger<br/>Facade]
        Models[Models<br/>ChMessage, ChFavorite]
        Storage[File Storage]
    end
    
    subgraph "Внешние сервисы"
        PusherServer[Pusher Server SaaS]
        Database[(База данных)]
    end
    
    UI --> JS
    JS --> PusherClient
    JS -->|HTTP/AJAX| Routes
    Routes --> Controller
    Controller --> Chatify
    Controller --> Models
    Chatify --> PusherServer
    Chatify --> Storage
    Models --> Database
    PusherClient <-->|WebSocket| PusherServer
```

## Поток сообщения от отправителя к получателю

```mermaid
sequenceDiagram
    participant Sender as Отправитель<br/>(User A)
    participant Browser as Браузер<br/>(JavaScript)
    participant Server as Laravel Server
    participant DB as База данных
    participant Pusher as Pusher Server
    participant Receiver as Получатель<br/>(User B)
    
    Note over Sender,Receiver: Этап 1: Отправка сообщения
    Sender->>Browser: Ввод текста/файла
    Browser->>Browser: Валидация и подготовка FormData
    Browser->>Server: POST /chatify/sendMessage<br/>(id, message, file, _token)
    
    Note over Server: Обработка в MessagesController::send()
    Server->>Server: Проверка файла (размер, расширение)
    Server->>Storage: Сохранение файла (если есть)
    Server->>Chatify: newMessage(data)
    Chatify->>DB: INSERT INTO ch_messages
    DB-->>Chatify: Сообщение сохранено
    Chatify->>Chatify: parseMessage(message)
    Chatify->>Chatify: messageCard(data)
    
    Note over Server,Pusher: Этап 2: Отправка через Pusher
    alt Если получатель != отправитель
        Chatify->>Pusher: push("private-chatify.{to_id}", "messaging", data)
        Pusher->>Receiver: WebSocket событие "messaging"
    end
    
    Note over Server,Browser: Этап 3: Ответ отправителю
    Server-->>Browser: JSON {status, message, tempID}
    Browser->>Browser: Замена временного сообщения<br/>на реальное
    
    Note over Receiver: Этап 4: Получение сообщения
    Receiver->>Browser: Получение события через Pusher
    Browser->>Browser: Добавление сообщения в UI
    Browser->>Browser: Воспроизведение звука (если включен)
    
    Note over Receiver,Server: Этап 5: Отметка как прочитанное
    Receiver->>Browser: Просмотр сообщения
    Browser->>Server: POST /chatify/makeSeen<br/>(id: user_id)
    Server->>DB: UPDATE ch_messages SET seen=1<br/>WHERE from_id=user_id AND to_id=auth_id
    DB-->>Server: Обновлено
    Server-->>Browser: JSON {status: 1}
```

## Структура базы данных

```mermaid
erDiagram
    USERS ||--o{ CH_MESSAGES : "from_id"
    USERS ||--o{ CH_MESSAGES : "to_id"
    USERS ||--o{ CH_FAVORITES : "user_id"
    USERS ||--o{ CH_FAVORITES : "favorite_id"
    
    USERS {
        bigint id PK
        string name
        string email
        string avatar
        boolean active_status
        boolean dark_mode
        string messenger_color
    }
    
    CH_MESSAGES {
        uuid id PK
        bigint from_id FK
        bigint to_id FK
        string body
        string attachment
        boolean seen
        timestamp created_at
        timestamp updated_at
    }
    
    CH_FAVORITES {
        uuid id PK
        bigint user_id FK
        bigint favorite_id FK
        timestamp created_at
        timestamp updated_at
    }
```

## Компоненты системы

```mermaid
graph LR
    subgraph "Frontend Layer"
        A[Blade Templates]
        B[JavaScript/jQuery]
        C[CSS Styles]
        D[Pusher JS SDK]
    end
    
    subgraph "Application Layer"
        E[ChatifyServiceProvider]
        F[Routes]
        G[MessagesController]
        H[ChatifyMessenger]
    end
    
    subgraph "Data Layer"
        I[ChMessage Model]
        J[ChFavorite Model]
        K[User Model]
        L[Database]
    end
    
    subgraph "Infrastructure"
        M[File Storage]
        N[Pusher Service]
    end
    
    A --> B
    B --> D
    B --> F
    E --> F
    F --> G
    G --> H
    G --> I
    G --> J
    H --> N
    H --> M
    I --> L
    J --> L
    K --> L
```

## Детальный поток обработки сообщения

### 1. Инициализация Pusher на клиенте

```mermaid
sequenceDiagram
    participant Client as Клиент
    participant Server as Laravel Server
    participant Pusher as Pusher Server
    
    Client->>Server: Загрузка страницы /chatify
    Server-->>Client: HTML + JavaScript
    Client->>Client: Инициализация Pusher<br/>с ключами из config
    Client->>Server: POST /chatify/chat/auth<br/>(channel_name, socket_id)
    Server->>Server: pusherAuth() проверка
    Server-->>Client: Авторизация канала
    Client->>Pusher: Подписка на private-chatify.{user_id}
    Pusher-->>Client: Подписка успешна
    Client->>Pusher: Подписка на presence-activeStatus
```

### 2. Отправка сообщения с вложением

```mermaid
flowchart TD
    Start([Пользователь отправляет сообщение]) --> Validate{Валидация<br/>на клиенте}
    Validate -->|Ошибка| Error1[Показать ошибку]
    Validate -->|OK| Prepare[Подготовка FormData]
    Prepare --> Upload{Есть файл?}
    Upload -->|Да| CheckFile[Проверка файла:<br/>- Размер<br/>- Расширение]
    CheckFile -->|Ошибка| Error2[Ошибка валидации]
    CheckFile -->|OK| SaveFile[Сохранение файла<br/>в Storage]
    Upload -->|Нет| SendRequest
    SaveFile --> SendRequest[POST /sendMessage]
    SendRequest --> ProcessServer[Обработка на сервере]
    ProcessServer --> SaveDB[(Сохранение в БД)]
    SaveDB --> TriggerPusher[Отправка через Pusher]
    TriggerPusher --> Response[Ответ клиенту]
    Response --> UpdateUI[Обновление UI]
    UpdateUI --> End([Готово])
    Error1 --> End
    Error2 --> End
```

### 3. Получение и отображение сообщений

```mermaid
sequenceDiagram
    participant Client as Клиент
    participant Server as Laravel Server
    participant DB as База данных
    participant Pusher as Pusher Server
    
    Note over Client: Загрузка истории сообщений
    Client->>Server: POST /chatify/fetchMessages<br/>(id, per_page)
    Server->>DB: SELECT * FROM ch_messages<br/>WHERE (from_id, to_id) OR (to_id, from_id)<br/>ORDER BY created_at DESC<br/>LIMIT per_page
    DB-->>Server: Список сообщений
    Server->>Server: parseMessage() для каждого
    Server->>Server: messageCard() - генерация HTML
    Server-->>Client: JSON {messages: HTML, total, last_page}
    Client->>Client: Вставка HTML в DOM
    
    Note over Client,Pusher: Real-time получение новых сообщений
    Pusher->>Client: Событие "messaging"
    Client->>Client: Добавление сообщения в UI
    Client->>Client: Воспроизведение звука
    Client->>Client: Прокрутка вниз
    Client->>Server: POST /chatify/makeSeen<br/>(id: from_id)
    Server->>DB: UPDATE ch_messages<br/>SET seen=1
```

## Ключевые методы ChatifyMessenger

### Работа с сообщениями

- `newMessage($data)` — создание нового сообщения в БД
- `parseMessage($message)` — парсинг сообщения в массив данных
- `messageCard($data)` — генерация HTML-карточки сообщения
- `fetchMessagesQuery($user_id)` — запрос для получения сообщений
- `makeSeen($user_id)` — отметка сообщений как прочитанных
- `deleteMessage($id)` — удаление сообщения
- `deleteConversation($user_id)` — удаление всей беседы

### Работа с Pusher

- `push($channel, $event, $data)` — отправка события через Pusher
- `pusherAuth($requestUser, $authUser, $channelName, $socket_id)` — аутентификация для приватных каналов

### Работа с пользователями

- `getContactItem($user)` — получение HTML элемента контакта
- `getUserWithAvatar($user)` — получение пользователя с аватаром
- `getLastMessageQuery($user_id)` — последнее сообщение с пользователем
- `countUnseenMessages($user_id)` — подсчет непрочитанных сообщений

### Работа с избранным

- `inFavorite($user_id)` — проверка, в избранном ли пользователь
- `makeInFavorite($user_id, $action)` — добавление/удаление из избранного

### Работа с файлами

- `getAttachmentUrl($attachment_name)` — получение URL вложения
- `getUserAvatarUrl($user_avatar_name)` — получение URL аватара
- `getSharedPhotos($user_id)` — получение общих фотографий

## Безопасность

1. **Аутентификация**: Все маршруты защищены middleware `auth`
2. **CSRF защита**: Используются CSRF токены для POST-запросов
3. **Валидация файлов**: Проверка размера и расширения файлов
4. **Санитизация**: HTML-сущности экранируются при сохранении
5. **Pusher авторизация**: Приватные каналы требуют авторизации на сервере
6. **Проверка прав**: Пользователь может удалять только свои сообщения

## Хранилище файлов

- **Аватары**: `storage/app/public/users-avatar/`
- **Вложения**: `storage/app/public/attachments/`
- **Диск**: Настраивается через `CHATIFY_STORAGE_DISK` (по умолчанию `public`)

## Конфигурация

Основные настройки в `config/chatify.php`:

- `pusher` — настройки Pusher (key, secret, app_id, cluster)
  - **Важно**: Требуется регистрация на pusher.com и получение учетных данных
  - Настройки берутся из переменных окружения `.env`
- `attachments` — настройки вложений (размер, расширения, папка)
- `user_avatar` — настройки аватаров
- `colors` — доступные цвета для мессенджера
- `sounds` — настройки звуков
- `routes` — настройки маршрутов

### Настройка Pusher

Для работы Chatify необходимо настроить Pusher:

1. Зарегистрируйтесь на [pusher.com](https://pusher.com)
2. Создайте новое приложение
3. Скопируйте учетные данные в `.env`:
   ```env
   PUSHER_APP_KEY=your_app_key
   PUSHER_APP_SECRET=your_app_secret
   PUSHER_APP_ID=your_app_id
   PUSHER_APP_CLUSTER=mt1
   ```
4. Убедитесь, что в `config/broadcasting.php` настроен драйвер `pusher`

## Расширяемость

Пакет поддерживает кастомизацию через:

1. **Публикация конфигурации**: `php artisan vendor:publish --tag=chatify-config`
2. **Публикация контроллеров**: `php artisan vendor:publish --tag=chatify-controllers`
3. **Публикация представлений**: `php artisan vendor:publish --tag=chatify-views`
4. **Публикация маршрутов**: `php artisan vendor:publish --tag=chatify-routes`
5. **Публикация моделей**: `php artisan vendor:publish --tag=chatify-models`

## Производительность

- **Пагинация**: Сообщения загружаются постранично (по умолчанию 30 на страницу)
- **Индексы БД**: Рекомендуется добавить индексы на `from_id`, `to_id`, `created_at` в таблице `ch_messages`
- **Кэширование**: Аватары и вложения хранятся в файловой системе
- **WebSocket**: Используется для real-time коммуникации без постоянных HTTP-запросов

## Заключение

Chatify представляет собой хорошо структурированный пакет для Laravel, который использует современные технологии (Pusher, WebSocket) для обеспечения real-time коммуникации. Архитектура разделена на четкие слои: представление, контроллеры, бизнес-логика (ChatifyMessenger), модели данных и внешние сервисы. Это обеспечивает хорошую поддерживаемость и расширяемость системы.

