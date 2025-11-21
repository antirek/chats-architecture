# Анализ архитектуры Chaskiq

## Обзор проекта

Chaskiq - это платформа для обмена сообщениями (messaging platform) для маркетинга, поддержки и продаж. Проект построен на Ruby on Rails с React.js фронтендом, использует PostgreSQL для хранения данных, Redis для кэширования и очередей, и ActionCable для WebSocket соединений.

## Ключевые понятия

### Основные сущности

#### App (Приложение)
- Основная сущность, представляющая отдельное приложение/организацию в системе
- Содержит настройки, предпочтения, интеграции
- Имеет множество пользователей, агентов, разговоров
- Метод `start_conversation` создает новый разговор

#### AppUser (Пользователь приложения)
- Представляет клиента/пользователя, который общается через мессенджер
- Может быть трех типов: `Visitor`, `Lead`, `AppUser`
- Имеет свойства (properties), метрики, визиты
- Связан с разговорами через `main_participant`
- Использует Redis для счетчиков новых сообщений

#### Agent (Агент)
- Представляет сотрудника/оператора, который отвечает на сообщения
- Может быть человеком или ботом (`bot: true`)
- Имеет роли и права доступа к приложениям
- Аутентификация через Devise

#### Conversation (Разговор)
- Основная единица общения между пользователем и агентом
- Имеет состояние: `opened`, `closed`
- Связан с `main_participant` (AppUser) и `assignee` (Agent)
- Содержит множество сообщений (`ConversationPart`)
- Может иметь теги, приоритет
- Использует AASM для управления состоянием

#### ConversationPart (Сообщение)
- Отдельное сообщение в разговоре
- Может быть разных типов через полиморфную связь `messageable`:
  - `ConversationPartContent` - текстовое сообщение
  - `ConversationPartBlock` - сообщение с блоками/контролами
  - `ConversationPartEvent` - системное событие
- Имеет автора через полиморфную связь `authorable` (AppUser или Agent)
- Может быть приватной заметкой (`private_note`)
- Отслеживает статус прочтения (`read_at`)

#### Message (Базовый класс для кампаний)
- Базовый класс для различных типов сообщений: Campaign, UserAutoMessage, Tour, Banner, BotTask
- Использует STI (Single Table Inheritance) через таблицу `campaigns`
- Имеет сегменты для таргетинга пользователей

#### ConversationChannel (Канал разговора)
- Связывает разговор с внешними каналами (WhatsApp, Twitter, Slack и т.д.)
- Используется для отправки сообщений через интеграции
- Имеет `provider` (название интеграции) и `provider_channel_id`

#### AppPackage / AppPackageIntegration
- Плагины/интеграции для расширения функциональности
- Позволяют добавлять новые каналы связи и функциональность
- Имеют API для обработки входящих и исходящих сообщений

### Каналы связи (ActionCable)

#### MessengerEventsChannel
- WebSocket канал для пользователей (AppUser)
- Подписка: `messenger_events:{app_key}-{session_id}`
- Используется для отправки событий пользователям в реальном времени
- Обрабатывает события от пользователей (отправка сообщений, трекинг и т.д.)

#### AgentChannel
- WebSocket канал для агентов
- Подписка: `events:{app_key}-{agent_id}`
- Используется для уведомления агентов о новых сообщениях и событиях

#### EventsChannel
- Общий канал для приложения
- Подписка: `events:{app_key}`
- Используется для широковещательных событий в рамках приложения

### Jobs (Фоновые задачи)

#### ApiChannelNotificatorJob
- Уведомляет внешние каналы (интеграции) о новом сообщении
- Вызывается после создания `ConversationPart`
- Проходит по всем `conversation_channels` и отправляет уведомления

#### EmailChatNotifierJob
- Отправляет email уведомления о новых сообщениях
- Выполняется с задержкой 20 секунд
- Не отправляется для автоматических сообщений и приватных заметок

#### HookMessageReceiverJob
- Обрабатывает входящие сообщения от внешних интеграций
- Вызывается через API hooks

## Архитектура системы

```mermaid
graph TB
    subgraph "Frontend"
        React[React.js Frontend]
        Messenger[Web Messenger Widget]
    end
    
    subgraph "Backend - Rails"
        GraphQL[GraphQL API]
        Controllers[REST Controllers]
        Models[ActiveRecord Models]
        Services[Services]
    end
    
    subgraph "Real-time"
        ActionCable[ActionCable]
        MessengerChannel[MessengerEventsChannel]
        AgentChannel[AgentChannel]
        EventsChannel[EventsChannel]
    end
    
    subgraph "Background Jobs"
        Sidekiq[Sidekiq]
        EmailJob[EmailChatNotifierJob]
        ChannelJob[ApiChannelNotificatorJob]
        HookJob[HookMessageReceiverJob]
    end
    
    subgraph "Data Storage"
        PostgreSQL[(PostgreSQL)]
        Redis[(Redis)]
    end
    
    subgraph "External"
        Integrations[App Packages<br/>WhatsApp, Twitter, etc]
        EmailProvider[Email Provider]
    end
    
    React --> GraphQL
    React --> ActionCable
    Messenger --> GraphQL
    Messenger --> ActionCable
    
    GraphQL --> Models
    Controllers --> Models
    Models --> PostgreSQL
    Models --> Redis
    
    ActionCable --> MessengerChannel
    ActionCable --> AgentChannel
    ActionCable --> EventsChannel
    ActionCable --> Redis
    
    Models --> Sidekiq
    Sidekiq --> EmailJob
    Sidekiq --> ChannelJob
    Sidekiq --> HookJob
    
    ChannelJob --> Integrations
    EmailJob --> EmailProvider
    Integrations --> HookJob
    HookJob --> Models
```

## Поток сообщения от одного участника другому

### Сценарий 1: Пользователь отправляет сообщение агенту

```mermaid
sequenceDiagram
    participant User as AppUser<br/>(Пользователь)
    participant Frontend as React Frontend
    participant GraphQL as GraphQL API
    participant Conversation as Conversation Model
    participant Part as ConversationPart
    participant Cable as ActionCable
    participant Agent as Agent
    participant Job as Background Jobs
    participant Channel as External Channels

    User->>Frontend: Вводит сообщение
    Frontend->>GraphQL: mutation insertComment
    GraphQL->>Conversation: add_message(options)
    Conversation->>Part: create ConversationPart
    Part->>Part: assign_and_notify (after_create)
    Part->>Part: increment_message_stats
    Part->>Part: handle_conversation_touches
    Part->>Part: assign_agent_by_rules
    Part->>Part: enqueue_email_notification
    Part->>Part: notify_to_channels
    Part->>Cable: broadcast to MessengerEventsChannel
    Part->>Cable: broadcast to EventsChannel
    Part->>Job: enqueue ApiChannelNotificatorJob
    Part->>Job: enqueue EmailChatNotifierJob (delayed)
    
    Cable->>User: WebSocket: conversations:conversation_part
    Cable->>Agent: WebSocket: conversation_part (новое сообщение)
    
    Job->>Channel: notify_message_on_available_channels
    Channel->>Integrations: Отправка в WhatsApp/Twitter/etc
    
    Job->>EmailProvider: Отправка email уведомления (через 20 сек)
```

### Сценарий 2: Агент отправляет сообщение пользователю

```mermaid
sequenceDiagram
    participant Agent as Agent
    participant Frontend as Agent Dashboard
    participant GraphQL as GraphQL API
    participant Conversation as Conversation Model
    participant Part as ConversationPart
    participant Cable as ActionCable
    participant User as AppUser
    participant Job as Background Jobs
    participant Channel as External Channels

    Agent->>Frontend: Вводит сообщение
    Frontend->>GraphQL: mutation insertComment
    GraphQL->>Conversation: add_message(options)
    Conversation->>Part: create ConversationPart
    Part->>Part: assign_and_notify (after_create)
    Part->>Part: increment_outgoing_messages_stat
    Part->>Part: handle_conversation_touches
    Note over Part: Обновляет first_agent_reply<br/>если это первое сообщение
    Part->>Part: notify_to_channels
    Part->>Cable: broadcast to MessengerEventsChannel
    Part->>Cable: broadcast to EventsChannel
    Part->>Job: enqueue ApiChannelNotificatorJob
    
    Cable->>User: WebSocket: conversations:conversation_part
    Cable->>User: WebSocket: conversations:unreads (счетчик)
    Cable->>Agent: WebSocket: conversation_part (подтверждение)
    
    Job->>Channel: notify_message_on_available_channels
    Channel->>Integrations: Отправка в WhatsApp/Twitter/etc
```

### Сценарий 3: Входящее сообщение от внешнего канала

```mermaid
sequenceDiagram
    participant External as External Channel<br/>(WhatsApp/Twitter)
    participant Hook as API Hook Controller
    participant Job as HookMessageReceiverJob
    participant Package as AppPackage
    participant Conversation as Conversation Model
    participant Part as ConversationPart
    participant Cable as ActionCable
    participant User as AppUser
    participant Agent as Agent

    External->>Hook: POST /api/v1/hooks/receiver/:id
    Hook->>Job: perform_later(id, params)
    Job->>Package: message_api_klass.process_event
    Package->>Package: Обработка события
    Package->>Conversation: find_or_create_by
    Package->>Conversation: add_message(from: external_user)
    Conversation->>Part: create ConversationPart
    Part->>Part: assign_and_notify
    Part->>Cable: broadcast to MessengerEventsChannel
    Part->>Cable: broadcast to EventsChannel
    
    Cable->>User: WebSocket: conversations:conversation_part
    Cable->>Agent: WebSocket: conversation_part (новое сообщение)
```

## Детальная схема обработки сообщения

```mermaid
flowchart TD
    Start([Сообщение создано]) --> CreatePart[Создание ConversationPart]
    CreatePart --> AfterCreate[after_create callback]
    
    AfterCreate --> ControlsPing[controls_ping_apis<br/>для блоков]
    AfterCreate --> IncrementStats[increment_message_stats]
    AfterCreate --> HandleTouches[handle_conversation_touches]
    AfterCreate --> CheckBot{От бота?}
    
    CheckBot -->|Нет| UpdateLatest[update_latest_user_visible_comment_at]
    CheckBot -->|Да| AssignRules
    
    UpdateLatest --> AssignRules{assignee<br/>пуст?}
    AssignRules -->|Да| CheckRules[check_rules<br/>assignment_rules]
    AssignRules -->|Нет| EnqueueEmail
    
    CheckRules --> AssignAgent[assign_user<br/>если правило сработало]
    AssignAgent --> EnqueueEmail
    
    EnqueueEmail --> CheckConstraints{send_constraints?}
    CheckConstraints -->|Нет| DelayEmail[EmailChatNotifierJob<br/>через 20 сек]
    CheckConstraints -->|Да| NotifyChannels
    
    DelayEmail --> NotifyChannels
    NotifyChannels --> NotifyAgents[notify_agents<br/>EventsChannel.broadcast]
    NotifyChannels --> NotifyUsers[notify_app_users<br/>MessengerEventsChannel.broadcast]
    NotifyChannels --> EnqueueChannel[ApiChannelNotificatorJob<br/>для внешних каналов]
    
    NotifyAgents --> End([Завершено])
    NotifyUsers --> End
    EnqueueChannel --> ExternalNotify[Уведомление внешних каналов]
    ExternalNotify --> End
```

## Структура данных

### ConversationPart (сообщение)

```mermaid
erDiagram
    ConversationPart ||--o{ ConversationPartChannelSource : has
    ConversationPart }o--|| Conversation : belongs_to
    ConversationPart }o--|| Message : "message_source"
    ConversationPart }o--o| ConversationPartContent : "messageable (polymorphic)"
    ConversationPart }o--o| ConversationPartBlock : "messageable (polymorphic)"
    ConversationPart }o--o| ConversationPartEvent : "messageable (polymorphic)"
    ConversationPart }o--|| AppUser : "authorable (polymorphic)"
    ConversationPart }o--|| Agent : "authorable (polymorphic)"
    
    Conversation ||--o{ ConversationPart : has_many
    Conversation }o--|| AppUser : "main_participant"
    Conversation }o--o| Agent : "assignee"
    Conversation ||--o{ ConversationChannel : has_many
    
    ConversationChannel }o--|| AppPackageIntegration : "через provider"
    
    AppUser ||--o{ Conversation : "main_participant"
    Agent ||--o{ Conversation : "assignee"
```

## Ключевые компоненты обработки

### 1. Создание сообщения

**Метод:** `Conversation#add_message(options)`

**Процесс:**
1. Создает новый `ConversationPart`
2. Устанавливает автора (`authorable`)
3. Создает содержимое сообщения (`ConversationPartContent`)
4. Сохраняет в транзакции
5. Вызывает `notify_to_channels`

### 2. Уведомления через WebSocket

**Каналы:**
- `MessengerEventsChannel` - для пользователей
  - Ключ: `messenger_events:{app_key}-{session_id}`
  - События: `conversations:conversation_part`, `conversations:unreads`, `conversations:update_state`
  
- `EventsChannel` - для агентов
  - Ключ: `events:{app_key}` или `events:{app_key}-{agent_id}`
  - События: `conversation_part`, `conversations:update_state`

### 3. Интеграции с внешними каналами

**Процесс:**
1. `ConversationPart` создается
2. `ApiChannelNotificatorJob` ставится в очередь
3. Job находит все `conversation_channels` для разговора
4. Для каждого канала вызывается `notify_part`
5. `AppPackageIntegration.message_api_klass.notify_message` отправляет сообщение

### 4. Автоматическое назначение агента

**Правила назначения (Assignment Rules):**
- Проверяются после создания сообщения от пользователя
- Анализируется текст сообщения
- Если правило срабатывает, назначается соответствующий агент
- Создается событие `conversation_assigned`

### 5. Боты и триггеры

**ActionTrigger:**
- Обрабатывает ответы пользователей на сообщения ботов
- Определяет следующий шаг в диалоге
- Создает новые сообщения от бота
- Регистрирует метрики взаимодействия

## Технологический стек

- **Backend:** Ruby on Rails 7.2.1
- **Frontend:** React.js
- **API:** GraphQL (graphql-ruby)
- **WebSocket:** ActionCable (AnyCable)
- **Database:** PostgreSQL
- **Cache/Queue:** Redis
- **Background Jobs:** Sidekiq
- **Search:** Searchkick (Elasticsearch/OpenSearch)
- **Authentication:** Devise, Doorkeeper (OAuth)

## Особенности реализации

1. **Полиморфные связи:** `ConversationPart` использует полиморфные связи для `authorable` (AppUser/Agent) и `messageable` (Content/Block/Event)

2. **State Machine:** `Conversation` использует AASM для управления состоянием (opened/closed)

3. **Redis Objects:** Используется для счетчиков новых сообщений и блокировок

4. **Event Sourcing:** События разговоров логируются через `Event` модель

5. **Плагинная архитектура:** `AppPackage` позволяет расширять функциональность через интеграции

6. **Многоязычность:** Используется Globalize для переводов

7. **Аудит:** Изменения записываются через `Audit` модель

## Интеграция с Telegram

### Обзор

Telegram интеграция работает через систему плагинов Chaskiq. Плагин Telegram загружается из AppStore (appstore.chaskiq.io) или может быть разработан локально. Интеграция использует Telegram Bot API для получения и отправки сообщений.

### Архитектура интеграции

```mermaid
sequenceDiagram
    participant TG as Telegram Bot API
    participant Webhook as Chaskiq Webhook<br/>/api/v1/hooks/receiver/:id
    participant Controller as ProviderController
    participant Job as HookMessageReceiverJob
    participant API as MessageApis::Telegram::Api
    participant Conversation as Conversation Model
    participant Part as ConversationPart
    participant Cable as ActionCable
    participant Agent as Agent
    participant User as AppUser

    TG->>Webhook: POST webhook (новое сообщение)
    Webhook->>Controller: process_event
    Controller->>Job: enqueue_process_event
    Job->>API: process_event(params, package)
    API->>API: Парсинг Telegram update
    API->>API: Извлечение данных пользователя
    API->>API: Обработка медиа (фото, видео, аудио, стикеры)
    API->>Conversation: find_or_create_by_channel
    API->>User: add_participant (создание/поиск AppUser)
    API->>Conversation: add_message
    Conversation->>Part: create ConversationPart
    Part->>Part: assign_and_notify
    Part->>Cable: broadcast to EventsChannel
    Cable->>Agent: WebSocket: conversation_part
```

### Входящие сообщения (Telegram → Chaskiq)

**Процесс обработки:**

1. **Webhook регистрация:**
   - При установке интеграции вызывается `register_webhook`
   - Telegram Bot API настраивается на отправку webhook'ов на URL: `/api/v1/hooks/receiver/{encoded_id}`
   - `encoded_id` содержит закодированный `app_key` и `integration_id`

2. **Получение сообщения:**
   - Telegram отправляет POST запрос с `update` объектом
   - Контроллер `ProviderController#process_event` получает запрос
   - Создается `HookMessageReceiverJob` для асинхронной обработки

3. **Обработка события (`MessageApis::Telegram::Api#process_event`):**
   ```ruby
   # Парсинг Telegram update
   message = params["message"]
   sender_id = message["from"]["id"]
   chat_id = message["chat"]["id"]
   text = message["text"]
   
   # Поиск или создание пользователя
   user_data = {
     id: sender_id,
     first_name: message["from"]["first_name"],
     last_name: message["from"]["last_name"]
   }
   participant = add_participant(user_data, "telegram")
   
   # Поиск или создание разговора
   conversation = find_conversation_by_channel("telegram", chat_id) ||
                 create_conversation_for_channel(participant, chat_id)
   
   # Обработка медиа
   if message["photo"]
     # Загрузка фото через Telegram API
     file_url = get_telegram_file(message["photo"].last["file_id"])
     # Создание блока изображения
   elsif message["video"] || message["animation"]
     # Обработка видео/GIF
   elsif message["voice"] || message["audio"]
     # Обработка аудио
   elsif message["sticker"]
     # Обработка стикера
   end
   
   # Создание сообщения
   conversation.add_message(
     from: participant,
     message: {
       html_content: text,
       serialized_content: serialize_content(text, media)
     },
     provider: "telegram",
     message_source_id: message["message_id"]
   )
   ```

4. **Создание ConversationChannel:**
   - При создании разговора создается `ConversationChannel`
   - `provider: "telegram"`, `provider_channel_id: chat_id`
   - Это связывает разговор с Telegram чатом

5. **Уведомления:**
   - После создания `ConversationPart` отправляются WebSocket уведомления
   - Агенты получают событие через `EventsChannel`
   - Пользователи получают обновление через `MessengerEventsChannel`

### Исходящие сообщения (Chaskiq → Telegram)

**Процесс отправки:**

1. **Триггер отправки:**
   - Когда агент отправляет сообщение в разговоре
   - `ConversationPart` создается через `Conversation#add_message`
   - После сохранения вызывается `notify_to_channels`

2. **Уведомление каналов:**
   ```ruby
   # ApiChannelNotificatorJob
   conversation.conversation_channels.each do |channel|
     channel.notify_part(conversation: conversation, part: part)
   end
   ```

3. **Отправка через Telegram API (`MessageApis::Telegram::Api#send_message`):**
   ```ruby
   def send_message(conversation, part)
     return if part.private_note?
     
     channel = conversation.conversation_channels
                          .find_by(provider: "telegram")
     return unless channel
     
     chat_id = channel.provider_channel_id
     message = part.message
     
     # Отправка текста
     if message.text_content.present?
       response = @conn.post(
         "https://api.telegram.org/bot#{@api_key}/sendMessage",
         {
           chat_id: chat_id,
           text: message.text_content,
           parse_mode: "HTML"
         }
       )
     end
     
     # Отправка медиа из блоков
     blocks = MessageApis::BlockManager.get_blocks(message.serialized_content)
     blocks.each do |block|
       if block["type"] == "ImageBlock"
         send_photo(chat_id, block["attrs"]["url"])
       elsif block["type"] == "FileBlock"
         send_document(chat_id, block["attrs"]["url"])
       end
     end
     
     # Сохранение связи
     message_id = JSON.parse(response.body)["result"]["message_id"]
     part.conversation_part_channel_sources.create(
       provider: "telegram",
       message_source_id: message_id
     )
   end
   ```

### Поддерживаемые типы сообщений

Telegram интеграция поддерживает:

1. **Текстовые сообщения:**
   - Обычный текст
   - Текст с переносами строк
   - HTML форматирование

2. **Медиа:**
   - **Фото:** Обрабатываются через `photo` массив, выбирается наибольшее разрешение
   - **Видео:** Обрабатываются через `video` или `animation` объект
   - **Аудио:** Обрабатываются через `voice` или `audio` объект
   - **Стикеры:** Обрабатываются через `sticker` объект
   - **Документы:** Обрабатываются через `document` объект

3. **Обработка медиа:**
   - Файлы загружаются через Telegram Bot API (`getFile`)
   - Сохраняются в ActiveStorage
   - Создаются соответствующие блоки в `serialized_content`

### Конфигурация интеграции

**Необходимые параметры:**
- `access_token` - токен Telegram бота (получается от @BotFather)
- `user_id` - ID владельца бота (опционально)

**Регистрация webhook:**
```ruby
def register_webhook(app_package, integration)
  webhook_url = integration.hook_url
  @conn.post(
    "https://api.telegram.org/bot#{@api_key}/setWebhook",
    { url: webhook_url }
  )
end
```

### Особенности реализации

1. **Связь сообщений:**
   - `ConversationPartChannelSource` связывает сообщение Chaskiq с Telegram message_id
   - Это позволяет отслеживать статус доставки и избегать дублирования

2. **Обработка ошибок:**
   - Ошибки Telegram API логируются
   - Сообщения с ошибками не создаются в системе

3. **Множественные чаты:**
   - Каждый Telegram чат создает отдельный `ConversationChannel`
   - Разговоры из одного чата группируются в один `Conversation`

4. **Анонимные пользователи:**
   - Пользователи Telegram создаются как анонимные (`Visitor` или `Lead`)
   - Связываются через `ExternalProfile` с `provider: "telegram"`

### Схема потока данных Telegram

```mermaid
graph TB
    subgraph "Telegram"
        Bot[Telegram Bot]
        User[Telegram User]
    end
    
    subgraph "Chaskiq"
        Webhook[Webhook Endpoint]
        Job[HookMessageReceiverJob]
        API[Telegram API Class]
        Conv[Conversation]
        Part[ConversationPart]
        Channel[ConversationChannel]
        AppUser[AppUser]
    end
    
    User -->|Отправляет сообщение| Bot
    Bot -->|Webhook POST| Webhook
    Webhook -->|enqueue| Job
    Job -->|process_event| API
    API -->|find_or_create| Conv
    API -->|add_participant| AppUser
    API -->|add_message| Part
    Part -->|notify| Channel
    
    Channel -->|send_message| API
    API -->|POST sendMessage| Bot
    Bot -->|Доставляет| User
```

## Потенциальные точки оптимизации

1. **N+1 запросы:** При загрузке разговоров с сообщениями
2. **WebSocket broadcast:** Можно оптимизировать фильтрацию событий
3. **Background jobs:** Можно добавить приоритеты для критичных задач
4. **Кэширование:** Можно кэшировать часто запрашиваемые данные (агенты, правила назначения)
5. **Telegram API rate limits:** Нужно учитывать лимиты Telegram (30 сообщений/сек для группы)

