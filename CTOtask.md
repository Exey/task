>Задание 1. Архитектура экосистемы
>Дано:
>>Экосистема состоит из нескольких частей:
>>- Игра-симулятор (лидогенерация и обучение пользователей).
>>- Cервис для мэтчинга пользователей.
>>- Модуль подготовки и подачи документов в суд (с интеграцией Rafinad.AI).
>>- Робот-судья (AI-ассистент, планируется в будущем).
>
>>Доступ к системе должен быть через:
>>- Telegram Mini App
>>- VK Mini App
>>- Web-кабинет
>>Backend на Python (FastAPI), предполагается микросервисная архитектура.
>
>>Нужно:
>>Схематично изобразить архитектуру (модули/сервисы и их взаимодействие).
>>Показать, как объединить все части в единую инфраструктуру:
>>- Авторизация (ЕСИА/соцсети).
>>- Базы данных и хранение документов.
>>- API-шлюз / коммуникация между сервисами.

>>Коротко объяснить, как решите проблему разных фронтов (Web, TG, VK).

### С помощью паттерна BFF 🟡

## Архитектура системы (цвета по слоям)

- **🔵 Клиенты** — Web, Telegram Mini App, VK Mini App  
- **🟡 BFF (API Gateway)** — единая точка входа, маршрутизация, JWT-валидация, rate limiting  
- **🟢 Микросервисы** — слабосвязанные, stateless, общаются через REST (синхронно) и Message Broker (асинхронно)  
  - `auth-svc` — OAuth 2.0 + OIDC с внешними провайдерами → выдача JWT  
  - `user-profile-svc` — управление профилем, ролями и прогрессом пользователя
  - `ocr-svc` — приём файлов, публикация задач в очередь, фоновый OCR  
  - `matching-svc` — подбор пар пользователей (истец/ответчик) по правилам и событиям
  - `game-svc` — игра симуляция судебных процессов для обучения и лидогенерации
  - `robot-judge-svc` —  AI-ассистент для анализа кейсов, оценки перспектив дела и генерации рекомендаций
- **🔴 Хранилище**  
  - **PostgreSQL** — единый кластер, изолированные схемы на сервис (пользователи, документы, мэтчи)  
  - **S3 / MinIO** — бинарные файлы (сканы, PDF)  
  - **Message Broker** (RabbitMQ / Redis) — асинхронные задачи  

```mermaid
graph TD
    classDef frontend stroke:#007bff,stroke-width:3px;
    classDef gateway stroke:#ffc107,stroke-width:3px;
    classDef service stroke:#28a745,stroke-width:3px;
    classDef data stroke:#dc3545,stroke-width:3px;
    classDef future stroke:#28a745,stroke-width:3px,stroke-dasharray: 5 5;

    %% Пользователи и фронты
    User[Пользователь]
    subgraph Frontends [Слой Клиентские приложения]
        direction LR
        TG[Telegram Mini App]
        VK[VK Mini App]
        Web[Web-кабинет]
    end
    User --> TG
    User --> VK
    User --> Web

    %% API-шлюз
    API_GW[BFF / API-шлюз всех фронтов]
    TG -->|HTTPS / REST| API_GW
    VK -->|HTTPS / REST| API_GW
    Web -->|HTTPS / REST| API_GW
    class TG,VK,Web frontend;
    class API_GW gateway;

    %% Микросервисы
    subgraph Backend["Слой Микросервисы на FastAPI"]
        direction TB
        Auth_SVC[Сервис Авторизации<br>auth-svc]
        UserProfile_SVC[Сервис Профилей Пользователей<br>user-profile-svс]
        Game_SVC[Сервис Игра-симулятор<br>game-svc]
        Matching_SVC[Сервис Мэтчинга<br>matching-svc]
        OCR_SVC[Сервис Документов<br>ocr-svc]
        RobotJudge_SVC[Сервис ИИ-Робот-судья<br>robot-judge-svc]
    end
    class Auth_SVC,UserProfile_SVC,Game_SVC,Matching_SVC,OCR_SVC service;
    class RobotJudge_SVC future;

    API_GW --> Auth_SVC
    API_GW --> UserProfile_SVC
    API_GW --> Game_SVC
    API_GW --> Matching_SVC
    API_GW --> OCR_SVC
    API_GW -.-> RobotJudge_SVC

    %% Взаимодействие между сервисами
    Auth_SVC -.->|JWT-токены| UserProfile_SVC
    OCR_SVC -->|Асинхронные задачи| Matching_SVC

    %% Хранилище данных
    subgraph Data_Storage [Слой Хранилище данных]
        PostgreSQL[(PostgreSQL)]
        S3[(S3 / MinIO)]
        MessageBroker[Message Broker<br>RabbitMQ / Redis]
    end
    class PostgreSQL,S3,MessageBroker data;

    UserProfile_SVC --> PostgreSQL
    Game_SVC --> PostgreSQL
    Matching_SVC --> PostgreSQL
    OCR_SVC --> PostgreSQL
    OCR_SVC --> S3
    OCR_SVC --> MessageBroker
```

## Авторизация

Клиент → **🟡 BFF** → **🟢 auth-svc** → (OAuth) → JWT → клиент.  

Все последующие запросы несут JWT → **🟡 BFF** валидирует → проксирует в нужный **🟢 микросервис** с `user_id`.

## Хранение

- Структурированные данные → **🔴 PostgreSQL**  
- Файлы → **🔴 S3 / MinIO**  
- Фоновые задачи → **🔴 Message Broker (Redis/Rabbit)**

```mermaid
graph TD
    classDef service stroke:#28a745,stroke-width:3px;
    classDef data stroke:#dc3545,stroke-width:3px;
    classDef mock stroke:#f8f9fa,stroke:#adb5bd,stroke-width:2px,stroke-dasharray: 4 4;

    %% БД
    DB[(Единая PostgreSQL<br/>БД: пользователи,<br/>документы, метаданные)]

    %% Авторизация 
    Auth_SVC[Сервис Авторизации]
    Mock_ESIA[Mock-ЕСИА заглушка]
    Mock_Socials[Mock-Соцсети заглушка]

    Auth_SVC -->|OAuth 2.0 + OIDC| Mock_ESIA
    Auth_SVC -->|OAuth 2.0| Mock_Socials
    Mock_ESIA -->|user_id| Auth_SVC
    Mock_Socials -->|user_id| Auth_SVC
    Auth_SVC -->|Чтение/запись| DB

    %% Документы
    Document_SVC[Сервис Документов]
    S3_Storage[(S3 / MinIO)]
    MsgBroker[(Message Broker)]

    Document_SVC -->|Метаданные| DB
    Document_SVC -->|Файлы| S3_Storage
    Document_SVC -->|Публикация задачи| MsgBroker

    Worker[Сервис OCR + AI Worker]
    Mock_Rafinad[Mock-Rafinad.AI заглушка]

    MsgBroker -->|Потребление задачи| Worker
    Worker -->|Результат| Mock_Rafinad
    Mock_Rafinad -->|Ответ| Worker
    Worker -->|Обновление статуса| DB

    class Auth_SVC,Document_SVC,Worker service;
    class DB,S3_Storage,MsgBroker data;
    class Mock_ESIA,Mock_Socials,Mock_Rafinad mock;
```

>Задание 2. План запуска MVP
>Дано:
>>В проекте много сложных мест: разные требования у TG/VK Mini Apps,
>>внешние API (ФНС, ЕСИА), OCR и работа с документами.
>Нужно:
>>Назвать 3 главных риска, которые вы видите для проекта на этапе MVP.
>>Кратко предложить, как их минимизировать (например: использовать
>>заглушки для API, начать с одной платформы, применить готовый OCR).
>

## Ключевые риски 
1. Внешние API: ЕСИА можно только банкам, в исключения очень непросто попасть (Мой опыт в Солар тому подтверждение)
   - Банк партнер необходим 
2. Сервис доков будет очень долго делаться (много корнер кейсов)
   - Да нужно брать готовый OCR
3. Можно словить бан от VK
   - Начать с WEB онли и TG канала
   - Второй итерацией делаем бот TG