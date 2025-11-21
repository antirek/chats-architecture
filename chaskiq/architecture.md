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

## Потенциальные точки оптимизации

1. **N+1 запросы:** При загрузке разговоров с сообщениями
2. **WebSocket broadcast:** Можно оптимизировать фильтрацию событий
3. **Background jobs:** Можно добавить приоритеты для критичных задач
4. **Кэширование:** Можно кэшировать часто запрашиваемые данные (агенты, правила назначения)

