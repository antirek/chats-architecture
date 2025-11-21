# Анализ архитектуры OpenIM Server

## Обзор проекта

OpenIM Server - это серверная часть системы мгновенного обмена сообщениями, построенная на микросервисной архитектуре. Проект написан на Go и поддерживает масштабирование для работы с большим количеством пользователей и сообщений.

## Основные компоненты системы

### 1. API Gateway (`openim-api`)
- **Назначение**: REST API для внешних систем и административных операций
- **Основные функции**:
  - Отправка сообщений через HTTP API
  - Управление пользователями, группами, друзьями
  - Административные операции

### 2. Message Gateway (`openim-msggateway`)
- **Назначение**: WebSocket шлюз для реального времени
- **Основные функции**:
  - Управление WebSocket соединениями
  - Прием сообщений от клиентов через WebSocket
  - Доставка сообщений клиентам в реальном времени
  - Управление онлайн статусом пользователей

### 3. Message RPC (`openim-rpc-msg`)
- **Назначение**: RPC сервис для обработки сообщений
- **Основные функции**:
  - Валидация сообщений
  - Обработка webhook'ов (before/after send)
  - Отправка сообщений в очередь для дальнейшей обработки
  - Поддержка разных типов чатов (single, group, notification)

### 4. Message Transfer (`openim-msgtransfer`)
- **Назначение**: Передача и сохранение сообщений
- **Основные функции**:
  - Чтение сообщений из очереди (Redis/Kafka)
  - Сохранение в Redis кэш
  - Отправка в очередь для MongoDB
  - Отправка в очередь для push уведомлений
  - Обработка последовательностей (seq) сообщений

### 5. Push Service (`openim-push`)
- **Назначение**: Доставка push уведомлений
- **Основные функции**:
  - Определение онлайн/офлайн статуса пользователей
  - Онлайн push через WebSocket (через msggateway)
  - Офлайн push через внешние сервисы (FCM, JPush, Getui и др.)

### 6. Другие RPC сервисы
- `openim-rpc-user`: Управление пользователями
- `openim-rpc-group`: Управление группами
- `openim-rpc-conversation`: Управление беседами
- `openim-rpc-friend`: Управление друзьями
- `openim-rpc-auth`: Аутентификация

## Поток сообщения от отправителя к получателю

### Общая схема потока

```mermaid
graph TB
    Client1[Клиент 1<br/>Отправитель]
    Client2[Клиент 2<br/>Получатель]
    
    subgraph "Слой доступа"
        API[openim-api<br/>REST API]
        WS[openim-msggateway<br/>WebSocket Gateway]
    end
    
    subgraph "Слой обработки"
        MSG_RPC[openim-rpc-msg<br/>Message RPC]
    end
    
    subgraph "Очереди сообщений"
        MQ1[Redis/Kafka<br/>Message Queue]
        MQ2[Redis/Kafka<br/>MongoDB Queue]
        MQ3[Redis/Kafka<br/>Push Queue]
    end
    
    subgraph "Слой передачи"
        TRANSFER[openim-msgtransfer<br/>Message Transfer]
    end
    
    subgraph "Слой доставки"
        PUSH[openim-push<br/>Push Service]
        MSG_GW2[openim-msggateway<br/>WebSocket Gateway]
    end
    
    subgraph "Хранилища"
        REDIS[(Redis<br/>Cache)]
        MONGO[(MongoDB<br/>Persistence)]
    end
    
    Client1 -->|1. Отправка сообщения| WS
    Client1 -->|Альтернатива: HTTP| API
    API -->|2. RPC вызов| MSG_RPC
    WS -->|2. RPC вызов| MSG_RPC
    MSG_RPC -->|3. Отправка в очередь| MQ1
    MQ1 -->|4. Чтение из очереди| TRANSFER
    TRANSFER -->|5. Сохранение в кэш| REDIS
    TRANSFER -->|6. Отправка в очередь| MQ2
    TRANSFER -->|7. Отправка в очередь| MQ3
    MQ2 -->|8. Сохранение в БД| MONGO
    MQ3 -->|9. Обработка push| PUSH
    PUSH -->|10. Онлайн push| MSG_GW2
    MSG_GW2 -->|11. Доставка через WS| Client2
    PUSH -->|12. Офлайн push| Client2
    
    style Client1 fill:#e1f5ff
    style Client2 fill:#e1f5ff
    style MSG_RPC fill:#fff4e1
    style TRANSFER fill:#fff4e1
    style PUSH fill:#fff4e1
    style REDIS fill:#ffe1f5
    style MONGO fill:#ffe1f5
```

### Детальный поток отправки сообщения

```mermaid
sequenceDiagram
    participant C1 as Клиент 1<br/>(Отправитель)
    participant WS as msggateway<br/>(WebSocket)
    participant API as openim-api<br/>(REST API)
    participant MSG as openim-rpc-msg<br/>(Message RPC)
    participant MQ1 as Message Queue<br/>(Redis/Kafka)
    participant TRANSFER as msgtransfer<br/>(Transfer Service)
    participant REDIS as Redis Cache
    participant MQ2 as MongoDB Queue
    participant MQ3 as Push Queue
    participant PUSH as push<br/>(Push Service)
    participant MSG_GW as msggateway<br/>(WebSocket Gateway)
    participant C2 as Клиент 2<br/>(Получатель)
    participant MONGO as MongoDB
    participant OFFLINE as Offline Push<br/>(FCM/JPush)
    
    Note over C1,API: Вариант 1: Отправка через WebSocket
    C1->>WS: 1. WebSocket сообщение
    WS->>MSG: 2. SendMsg RPC
    
    Note over C1,API: Вариант 2: Отправка через HTTP API
    C1->>API: 1. HTTP POST /msg/send_msg
    API->>MSG: 2. SendMsg RPC
    
    Note over MSG: 3. Валидация и обработка
    MSG->>MSG: 3.1. Проверка прав доступа
    MSG->>MSG: 3.2. Webhook before send
    MSG->>MSG: 3.3. Модификация сообщения
    MSG->>MQ1: 4. MsgToMQ (отправка в очередь)
    
    Note over TRANSFER: 5. Обработка сообщения
    MQ1->>TRANSFER: 5.1. Чтение из очереди
    TRANSFER->>TRANSFER: 5.2. Категоризация сообщений<br/>(storage/not storage)
    TRANSFER->>REDIS: 5.3. BatchInsertChat2Cache<br/>(сохранение в кэш)
    TRANSFER->>MQ2: 5.4. MsgToMongoMQ<br/>(очередь для MongoDB)
    TRANSFER->>MQ3: 5.5. MsgToPushMQ<br/>(очередь для push)
    
    Note over MONGO: 6. Сохранение в БД (асинхронно)
    MQ2->>MONGO: 6.1. BatchInsertChat2DB<br/>(сохранение в MongoDB)
    
    Note over PUSH: 7. Доставка сообщения
    MQ3->>PUSH: 7.1. Чтение из push очереди
    PUSH->>PUSH: 7.2. Определение онлайн статуса
    
    alt Пользователь онлайн
        PUSH->>MSG_GW: 7.3. SuperGroupOnlineBatchPushOneMsg
        MSG_GW->>MSG_GW: 7.4. Поиск WebSocket соединений
        MSG_GW->>C2: 7.5. Push через WebSocket
    else Пользователь офлайн
        PUSH->>OFFLINE: 7.6. Offline Push<br/>(FCM/JPush/Getui)
        OFFLINE->>C2: 7.7. Push уведомление
    end
```

### Архитектура компонентов

```mermaid
graph TB
    subgraph "Клиентский слой"
        WEB[Web Client]
        MOBILE[Mobile Client]
        DESKTOP[Desktop Client]
    end
    
    subgraph "Gateway слой"
        API_GW[openim-api<br/>REST API Gateway]
        WS_GW[openim-msggateway<br/>WebSocket Gateway]
    end
    
    subgraph "RPC сервисы"
        MSG_RPC[openim-rpc-msg<br/>Message Service]
        USER_RPC[openim-rpc-user<br/>User Service]
        GROUP_RPC[openim-rpc-group<br/>Group Service]
        CONV_RPC[openim-rpc-conversation<br/>Conversation Service]
        FRIEND_RPC[openim-rpc-friend<br/>Friend Service]
        AUTH_RPC[openim-rpc-auth<br/>Auth Service]
    end
    
    subgraph "Обработка сообщений"
        TRANSFER[openim-msgtransfer<br/>Message Transfer]
        PUSH[openim-push<br/>Push Service]
    end
    
    subgraph "Очереди"
        MSG_MQ[Message Queue<br/>Redis/Kafka]
        MONGO_MQ[MongoDB Queue<br/>Redis/Kafka]
        PUSH_MQ[Push Queue<br/>Redis/Kafka]
    end
    
    subgraph "Хранилища"
        REDIS[(Redis<br/>Cache & Queue)]
        MONGO[(MongoDB<br/>Messages)]
        MINIO[(MinIO/S3<br/>Files)]
    end
    
    subgraph "Внешние сервисы"
        FCM[FCM]
        JPUSH[JPush]
        GETUI[Getui]
    end
    
    WEB --> API_GW
    WEB --> WS_GW
    MOBILE --> WS_GW
    DESKTOP --> WS_GW
    
    API_GW --> MSG_RPC
    API_GW --> USER_RPC
    API_GW --> GROUP_RPC
    API_GW --> CONV_RPC
    API_GW --> FRIEND_RPC
    API_GW --> AUTH_RPC
    
    WS_GW --> MSG_RPC
    WS_GW --> USER_RPC
    
    MSG_RPC --> MSG_MQ
    MSG_RPC --> REDIS
    
    MSG_MQ --> TRANSFER
    TRANSFER --> REDIS
    TRANSFER --> MONGO_MQ
    TRANSFER --> PUSH_MQ
    
    MONGO_MQ --> MONGO
    PUSH_MQ --> PUSH
    
    PUSH --> WS_GW
    PUSH --> FCM
    PUSH --> JPUSH
    PUSH --> GETUI
    
    USER_RPC --> REDIS
    GROUP_RPC --> REDIS
    CONV_RPC --> REDIS
    FRIEND_RPC --> REDIS
    
    MSG_RPC --> MINIO
    
    style MSG_RPC fill:#fff4e1
    style TRANSFER fill:#fff4e1
    style PUSH fill:#fff4e1
    style REDIS fill:#ffe1f5
    style MONGO fill:#ffe1f5
```

## Детальный анализ потока сообщения

### Этап 1: Отправка сообщения клиентом

**Через WebSocket:**
1. Клиент устанавливает WebSocket соединение с `openim-msggateway`
2. Клиент отправляет сообщение через WebSocket с типом `WSSendMsg` (1003)
3. `msggateway` обрабатывает сообщение в `message_handler.go:SendMessage`
4. Вызывается RPC `msg.SendMsg`

**Через HTTP API:**
1. Клиент отправляет POST запрос на `/msg/send_msg`
2. `openim-api` обрабатывает в `msg.go:SendMessage`
3. Проверяются права доступа (только app manager)
4. Вызывается RPC `msg.SendMsg`

### Этап 2: Обработка в Message RPC

**Файл:** `internal/rpc/msg/send.go`

1. **Валидация доступа** (`authverify.CheckAccess`)
2. **Определение типа чата**:
   - `SingleChatType` - личный чат
   - `ReadGroupChatType` - групповой чат
   - `NotificationChatType` - уведомления
3. **Webhook before send** (если настроен)
4. **Модификация сообщения** (webhook before modify)
5. **Отправка в очередь** через `MsgDatabase.MsgToMQ`

**Ключевой код:**
```go
// internal/rpc/msg/send.go:81
err = m.MsgDatabase.MsgToMQ(ctx, 
    conversationutil.GenConversationUniqueKeyForGroup(req.MsgData.GroupID), 
    req.MsgData)
```

### Этап 3: Обработка в Message Transfer

**Файл:** `internal/msgtransfer/online_history_msg_handler.go`

1. **Чтение из очереди** - `HandlerRedisMessage`
2. **Батчинг** - сообщения группируются по ключу (conversationID)
3. **Категоризация**:
   - Сообщения для хранения (`IsHistory`)
   - Сообщения только для push (`!IsHistory`)
   - Уведомления
4. **Сохранение в Redis**:
   - `BatchInsertChat2Cache` - сохранение в кэш
   - Установка последовательностей (seq)
5. **Отправка в MongoDB очередь** - `MsgToMongoMQ`
6. **Отправка в Push очередь** - `MsgToPushMQ`

**Ключевой код:**
```go
// internal/msgtransfer/online_history_msg_handler.go:267
lastSeq, isNewConversation, userSeqMap, err := 
    och.msgTransferDatabase.BatchInsertChat2Cache(ctx, conversationID, storageMessageList)
```

### Этап 4: Сохранение в MongoDB

**Файл:** `internal/msgtransfer/online_msg_to_mongo_handler.go`

1. **Чтение из MongoDB очереди**
2. **Батч вставка** - `BatchInsertChat2DB`
3. **Webhook after save** (если настроен)

### Этап 5: Доставка через Push Service

**Файл:** `internal/push/push_handler.go`

1. **Чтение из Push очереди** - `HandleMs2PsChat`
2. **Определение типа доставки**:
   - Групповой чат → `Push2Group`
   - Личный чат → `Push2User`
3. **Проверка онлайн статуса**:
   - `GetUsersOnlineStatus` - проверка через Redis кэш
4. **Онлайн доставка**:
   - `GetConnsAndOnlinePush` - получение WebSocket соединений
   - `SuperGroupOnlineBatchPushOneMsg` - отправка через msggateway
5. **Офлайн доставка** (если онлайн доставка не удалась):
   - Фильтрация пользователей (не беспокоить и т.д.)
   - Отправка через внешние сервисы (FCM, JPush, Getui)

**Ключевой код:**
```go
// internal/push/push_handler.go:195
onlineUserIDs, offlineUserIDs, err := c.onlineCache.GetUsersOnline(ctx, pushToUserIDs)
```

### Этап 6: Доставка через WebSocket Gateway

**Файл:** `internal/msggateway/hub_server.go`

1. **Получение соединений** - `GetUserAllCons(userID)`
2. **Отправка сообщения** - `pushToUser`
3. **Проверка фона** - если приложение в фоне (iOS), не отправлять
4. **Push через WebSocket** - `client.PushMessage(ctx, msgData)`

## Типы сообщений и их обработка

### 1. Личный чат (SingleChatType)

```mermaid
graph LR
    A[Отправитель] -->|1. SendMsg| MSG[Message RPC]
    MSG -->|2. MsgToMQ| MQ[Message Queue]
    MQ -->|3. Process| TRANSFER[Transfer]
    TRANSFER -->|4. Save| REDIS[Redis Cache]
    TRANSFER -->|5. Push| PUSH[Push Service]
    PUSH -->|6. Online| WS[WebSocket]
    PUSH -->|7. Offline| FCM[FCM/JPush]
    WS -->|8. Deliver| B[Получатель]
    FCM -->|9. Notify| B
```

### 2. Групповой чат (ReadGroupChatType)

```mermaid
graph TB
    A[Отправитель] -->|1. SendMsg| MSG[Message RPC]
    MSG -->|2. MsgToMQ| MQ[Message Queue]
    MQ -->|3. Process| TRANSFER[Transfer]
    TRANSFER -->|4. Save| REDIS[Redis Cache]
    TRANSFER -->|5. Push| PUSH[Push Service]
    PUSH -->|6. Get Members| GROUP[Group RPC]
    GROUP -->|7. Member IDs| PUSH
    PUSH -->|8. Batch Push| WS[WebSocket Gateway]
    WS -->|9. Deliver| B1[Участник 1]
    WS -->|9. Deliver| B2[Участник 2]
    WS -->|9. Deliver| B3[Участник N]
    PUSH -->|10. Offline Push| FCM[FCM/JPush]
    FCM -->|11. Notify| B4[Офлайн участник]
```

### 3. Уведомления (NotificationChatType)

Обрабатываются аналогично личному чату, но с дополнительной логикой для системных уведомлений.

## Хранилища данных

### Redis
- **Кэш сообщений** - быстрый доступ к последним сообщениям
- **Онлайн статус** - информация о подключенных пользователях
- **Последовательности (seq)** - порядковые номера сообщений
- **Очереди** - для асинхронной обработки

### MongoDB
- **История сообщений** - долгосрочное хранение
- **Метаданные** - информация о беседах, пользователях, группах

### MinIO/S3
- **Файлы** - изображения, видео, документы

## Очереди сообщений

Система использует очереди (Redis Streams или Kafka) для:
1. **Асинхронной обработки** - разгрузка RPC сервисов
2. **Масштабирования** - возможность горизонтального масштабирования
3. **Надежности** - гарантия доставки сообщений

**Основные очереди:**
- **Message Queue** - для передачи сообщений в transfer
- **MongoDB Queue** - для сохранения в БД
- **Push Queue** - для доставки push уведомлений

## Webhook'и

Система поддерживает webhook'и на разных этапах:
- **BeforeSendSingleMsg** - перед отправкой личного сообщения
- **AfterSendSingleMsg** - после отправки личного сообщения
- **BeforeSendGroupMsg** - перед отправкой группового сообщения
- **AfterSendGroupMsg** - после отправки группового сообщения
- **BeforeMsgModify** - перед модификацией сообщения
- **AfterMsgSaveDB** - после сохранения в БД
- **BeforeOnlinePush** - перед онлайн push
- **BeforeOfflinePush** - перед офлайн push

## Масштабирование

### Горизонтальное масштабирование
- **msggateway** - можно запускать несколько инстансов
- **msgtransfer** - можно запускать несколько инстансов (consumer groups)
- **push** - можно запускать несколько инстансов
- **RPC сервисы** - можно запускать несколько инстансов

### Вертикальное масштабирование
- Настройка размеров батчей
- Настройка количества воркеров
- Настройка размеров очередей

## Заключение

OpenIM Server использует современную микросервисную архитектуру с разделением ответственности:
- **Gateway слой** - точка входа для клиентов
- **RPC слой** - бизнес-логика
- **Transfer слой** - обработка и сохранение
- **Push слой** - доставка сообщений

Поток сообщения проходит через несколько этапов с использованием очередей для асинхронной обработки, что обеспечивает высокую производительность и надежность системы.

