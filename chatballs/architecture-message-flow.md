# Архитектура Chatballs

Self-hosted ИИ-платформа клиентской поддержки: модульный Django-монолит, один `compose.yaml`, мультитенантность через PostgreSQL RLS. Клиент пишет в мессенджер или виджет → сообщение попадает в единый `Message` → операторы видят его в `internal-ui`, ответ уходит тем же транспортом (или появляется в виджете по poll).

Канонические источники: [`.skaro/architecture.md`](../.skaro/architecture.md), ADR-0001, ADR-0002, ADR-0008, ADR-0018, ADR-0028.

---

## Ключевые понятия

| Понятие | Что это |
|--------|---------|
| **Organization** | Граница изоляции данных. Всё клиентское принадлежит организации. `organization_id` от клиента никогда не основание доступа. |
| **HumanUser / Membership** | Глобальная учётка + участие в организации (`OWNER` / `ADMIN` / `EMPLOYEE`). Один пользователь — несколько организаций. |
| **Агент** | Продуктовый объект «кто отвечает». В БД это пара `Channel` ↔ `AIAgent` 1:1: канал — якорь группы/подключений, агент — инструкции, модель, знания. В UI «канал» отдельно не показывается. |
| **Интеграция (подключение)** | Транспорт или LLM-провайдер (`Integration`). Мессенджеры: Telegram, MAX, VK, WEB, EMAIL. Не владеют диалогами и знаниями. |
| **Contact** | Клиент организации (имя, телефон, email…). |
| **ConnectionIdentity** | Связка контакта с внешним id в конкретном подключении (`external_user_id`). |
| **Conversation** | Одно обращение/диалог. Три оси состояния (ниже). Новое сообщение после `CLOSED` создаёт **новый** диалог, а не открывает старый. |
| **Lifecycle** | `OPEN` / `CLOSED` / `SPAM` — жив ли диалог как обращение. |
| **ControlMode** | `AI` / `HUMAN` / `PAUSED` — кто ведёт переписку. Новый диалог стартует в `AI`. |
| **ExpectedResponder** | `CUSTOMER` / `AI` / `OPERATOR` / `NOBODY` — от кого ждут следующую реплику. |
| **Message** | Единица ленты. `author_type`: `CONTACT` / `AI` / `OPERATOR` / `SYSTEM`. |
| **ConversationParticipant** | Не «участник чата», а временное членство **сотрудника** в диалоге (специалист join/leave). |
| **ingest_inbound** | Единая точка приёма входящих: виджет и poller мессенджеров. |
| **Inbox / Outbox** | Идемпотентный приём и отложенная доставка событий (`events`). Приём только пишет; ход AI — отдельно на `worker-events`. |
| **Realtime (invalidate-only)** | WebSocket операторам несёт «обновись», не тело сообщения. Данные — повторным REST. |
| **Claim** | Атомарный захват диалога сотрудником в режиме `HUMAN`; писать клиенту может назначенный. |

---

## Карта системы

```mermaid
flowchart TB
  subgraph clients [Клиенты]
    Site[Сайт с виджетом]
    Messengers[Telegram MAX VK Email]
    Ops[Сотрудник internal-ui]
  end

  subgraph edge [Публичная граница]
    Caddy[Caddy gateway]
    FE[frontend nginx]
  end

  subgraph app [Backend surfaces]
    BA["backend-app: tenant API, WS, webchat"]
    BP[backend-platform]
    BAdm[backend-admin loopback]
  end

  subgraph workers [Фон]
    Poller["worker poller: опрос мессенджеров"]
    Events["worker-events: AI, notify, calls"]
  end

  subgraph data [Данные]
    PG[(PostgreSQL + pgvector)]
    Redis["Redis: cache + channel layer"]
  end

  Site -->|HTTP chat-widget.js| Caddy
  Ops -->|HTTPS| Caddy
  Messengers -.->|API провайдера| Poller
  Caddy --> FE
  FE -->|api and ws| BA
  FE -->|статика| Ops
  FE -->|виджет| Site
  BA --> PG
  BA --> Redis
  Poller --> PG
  Events --> PG
  Events --> Redis
  BA -.->|group_send| Redis
  BP --> PG
```

---

## Модули backend (границы ответственности)

```mermaid
flowchart LR
  subgraph identity_tenancy [Идентичность]
    identity
    tenancy
  end

  subgraph agent_stack [Агент и вход]
    channels
    ai
    integrations
    webchat
  end

  subgraph comms [Коммуникации]
    conversations
    calls
    notifications
    ratings
  end

  subgraph infra [Инфра доменов]
    events
    realtime[realtime.py]
    presence[presence.py]
  end

  integrations -->|connection to channel| channels
  channels -->|1 to 1| ai
  webchat -->|WEB integration| integrations
  conversations -->|ingest and send| integrations
  conversations --> events
  conversations --> realtime
  events -->|AI turn| conversations
  calls -->|child session| conversations
```

Приложения: `apps/backend`, `apps/internal-ui`, `apps/web-chat`. Пакеты: `packages/contracts`, `packages/shared`, `packages/ui`.

---

## Прохождение сообщения: общая схема

Один персистентный центр — `Message` на `Conversation`. Асимметрия доставки:

| Получатель | Как узнаёт о новом сообщении |
|------------|------------------------------|
| Другие операторы | WS invalidate → REST refetch |
| Посетитель веб-чата | Poll `GET /api/v1/webchat/messages/?since=` |
| Клиент в мессенджере | Синхронный/адаптерный `transports.send_*` |

Webhook’ов мессенджеров нет: входящие с Telegram/MAX/VK/Email — через **poller**.

```mermaid
flowchart TB
  subgraph inbound [Входящие]
    WC[web-chat POST webchat messages]
    POLL[poller poll_updates]
  end

  ING[ingest_inbound]
  MSG[(Message CONTACT)]
  SIG[signals notify_conversation_changed]
  WS[Redis Channels to ConversationEventsConsumer]
  UI[internal-ui refetch REST]

  WC --> ING
  POLL --> ING
  ING --> MSG
  MSG --> SIG
  SIG --> WS --> UI

  ING -->|control_mode AI| OUT[outbox ai_turn_requested]
  OUT --> EW[worker-events]
  EW --> AIM[(Message AI)]
  AIM --> SEND[transports.send_reply]

  OP[internal-ui POST messages]
  OP --> POM[post_operator_message]
  POM --> OPM[(Message OPERATOR)]
  OPM --> SEND
  OPM --> SIG

  SEND -->|WEB| NOOP[web_noop already in DB]
  SEND -->|TG MAX VK Email| API[API провайдера]
  NOOP --> POLLUI[виджет polls history]
```

---

## Sequence: клиент → оператор (веб-чат)

```mermaid
sequenceDiagram
  autonumber
  participant V as Visitor web-chat
  participant API as WebchatMessagesView
  participant IN as ingest_inbound
  participant DB as PostgreSQL
  participant RT as realtime.publish
  participant WS as ConversationEventsConsumer
  participant UI as internal-ui

  V->>API: POST webchat messages
  API->>IN: InboundMessage
  IN->>DB: Contact Identity Conversation Message CONTACT
  IN-->>DB: optional request_ai_turn then outbox
  Note over DB: post_save Message
  DB->>RT: conversation.changed + inbox.changed
  RT->>WS: Redis group_send
  WS->>UI: JSON invalidate
  UI->>API: GET conversations messages
  API-->>UI: message list with new message
```

Мессенджер отличается только входом: `poll_all_messengers` → тот же `ingest_inbound`.

---

## Sequence: оператор → клиент

```mermaid
sequenceDiagram
  autonumber
  participant UI as internal-ui
  participant MV as ConversationMessageView
  participant POM as post_operator_message
  participant DB as PostgreSQL
  participant TR as transports.send_reply
  participant EXT as Messenger or WEB
  participant V as Visitor web-chat
  participant RT as realtime
  participant Peer as Other operators

  UI->>MV: POST conversations messages
  MV->>POM: claim and PUBLIC_REPLY
  POM->>DB: Message OPERATOR
  POM->>TR: send_reply
  alt WEB
    TR-->>EXT: noop already in DB
    V->>MV: poll GET webchat messages
    MV-->>V: new message
  else Telegram MAX VK Email
    TR->>EXT: send_text voice file
  end
  DB->>RT: conversation.changed
  RT->>Peer: WS invalidate then REST
```

---

## Sequence: AI-ответ (отдельно от приёма)

По ADR-0028 poller/HTTP только записывают; генерация — на `worker-events`.

```mermaid
sequenceDiagram
  participant IN as ingest_inbound
  participant OB as outbox
  participant EW as worker-events
  participant AI as run_requested_turn
  participant DB as PostgreSQL
  participant TR as transports

  IN->>OB: conversation.ai_turn_requested
  EW->>OB: claim with SKIP LOCKED
  EW->>AI: handle_ai_turn_requested
  AI->>DB: store_answer Message AI
  AI->>TR: deliver
  Note over TR: WEB noop, messengers send adapters
```

Пока ход в `PENDING`/`RUNNING`, виджет в poll видит `thinking` (не классический typing оператора).

---

## Доменная модель сообщения (упрощённо)

```mermaid
erDiagram
  Organization ||--o{ Contact : has
  Organization ||--o{ Channel : has
  Channel ||--o| AIAgent : agent
  Channel ||--o{ Integration : connections
  Contact ||--o{ ConnectionIdentity : identities
  Integration ||--o{ ConnectionIdentity : via
  Contact ||--o{ Conversation : has
  Channel ||--o{ Conversation : routes
  Integration ||--o{ Conversation : connection
  Conversation ||--o{ Message : contains
  Conversation ||--o{ ConversationRead : operator_cursors
  Conversation ||--o{ ConversationParticipant : staff_membership

  Conversation {
    string lifecycle
    string control_mode
    string expected_responder
    string external_chat_id
  }
  Message {
    string author_type
    string kind
    string text
    string external_id
    string ai_turn_state
  }
```

---

## Важные детали пути

1. **Единый ingest.** И виджет (`webchat.services.post_message`), и poller сходятся в `conversations/ingest.py::ingest_inbound`.
2. **Исходящие оператора.** `operator_messages.post_operator_message` пишет `Message` и сразу зовёт транспорт (с `domain_operation("PUBLIC_REPLY")` и проверкой `require_human_reply`).
3. **Realtime не несёт контент.** `conversations/realtime.py` шлёт только `inbox.changed` / `conversation.changed`; видимость и тело — в REST.
4. **Веб-чат без WS.** Доставка клиенту — БД + poll; для WEB исходящий транспорт — noop.
5. **Нет messenger webhooks.** Входящие с внешних сетей — цикл `worker --role=poller` (~каждые несколько секунд).
6. **Read receipts** — курсор оператора (`ConversationRead`), наружу клиенту не уходят.
7. **Presence** — hint по открытым WS сотрудников (Redis), не авторизация и не часть message pipeline.

Ключевые файлы:

- `apps/backend/chatballs/conversations/ingest.py`
- `apps/backend/chatballs/conversations/operator_messages.py`
- `apps/backend/chatballs/conversations/message_views.py`
- `apps/backend/chatballs/conversations/poller.py`
- `apps/backend/chatballs/conversations/transports/`
- `apps/backend/chatballs/conversations/signals.py`
- `apps/backend/chatballs/conversations/realtime.py`
- `apps/backend/chatballs/conversations/consumers.py`
- `apps/backend/chatballs/conversations/ai_turn.py`
- `apps/backend/chatballs/conversations/ai_delivery.py`
- `apps/backend/chatballs/realtime.py`
- `apps/backend/chatballs/webchat/services.py`
- `apps/backend/chatballs/events/management/commands/run_worker.py`
