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

```mermaid

graph TD
    classDef frontend fill:#e6f3ff,stroke:#007bff,stroke-width:2px;
    classDef gateway fill:#fff3cd,stroke:#ffc107,stroke-width:2px;
    classDef service fill:#d4edda,stroke:#28a745,stroke-width:2px;
    classDef data fill:#f8d7da,stroke:#dc3545,stroke-width:2px;
    classDef external fill:#e2e3e5,stroke:#6c757d,stroke-width:2px,stroke-dasharray: 5 5;
    classDef future fill:#d1ecf1,stroke:#17a2b8,stroke-width:2px,stroke-dasharray: 5 5;
    %% Пользователи и фронтенды
    User[Пользователь]
    subgraph Frontends [Клиентские приложения]
        direction LR
        TG[Telegram Mini App]
        VK[VK Mini App]
        Web[Web-кабинет]
    end
    User --> TG
    User --> VK
    User --> Web

    %% API-шлюз
    API_GW[API-шлюз / BFF]

    TG -->|HTTPS / REST| API_GW
    VK -->|HTTPS / REST| API_GW
    Web -->|HTTPS / REST| API_GW
    class TG,VK,Web frontend;
    class API_GW gateway;

    %% Микросервисы
    subgraph Backend["Микросервисы на FastAPI"]
        direction TB
        Auth_SVC[Сервис Авторизации]
        User_SVC[Сервис Профилей Пользователей]
        Game_SVC[Игра-симулятор]
        Matching_SVC[Сервис Мэтчинга]
        Document_SVC[Модуль Документов]
        RobotJudge_SVC[Робот-судья с ИИ]
    end
    class Auth_SVC,User_SVC,Game_SVC,Matching_SVC,Document_SVC service;
    class RobotJudge_SVC future;

    API_GW --> Auth_SVC
    API_GW --> User_SVC
    API_GW --> Game_SVC
    API_GW --> Matching_SVC
    API_GW --> Document_SVC
    API_GW -.-> RobotJudge_SVC

    %% Взаимодействие между сервисами
    Auth_SVC -.->|JWT-токены| User_SVC
    Document_SVC -->|Асинхронные задачи| Matching_SVC

    %% Инфраструктура и данные
    subgraph Data_Storage [Хранилище данных]
        PostgreSQL[(PostgreSQL)]
        S3[(Object Storage<br/>S3 / MinIO)]
        MessageBroker[<br>Message Broker<br/>RabbitMQ / Redis]
    end
    class PostgreSQL,S3,MessageBroker data;

    User_SVC --> PostgreSQL
    Game_SVC --> PostgreSQL
    Matching_SVC --> PostgreSQL
    Document_SVC --> PostgreSQL
    Document_SVC --> S3
    Document_SVC --> MessageBroker

    %% Внешние интеграции
    subgraph External_APIs [Внешние сервисы]
        ESIA[ЕСИА API]
        Socials[API Соцсетей]
        Rafinad[Rafinad.AI API]
        FNS[ФНС API]
    end
    class ESIA,Socials,Rafinad,FNS external;

    Auth_SVC -->|OAuth 2.0 / OpenID Connect| ESIA
    Auth_SVC -->|OAuth 2.0| Socials
    Document_SVC -->|HTTPS / gRPC| Rafinad
    Document_SVC -->|SOAP / REST| FNS
```

>>Коротко объяснить, как решите проблему разных фронтов (Web, TG, VK).




>Задание 2. План запуска MVP
>Дано:
>>В проекте много сложных мест: разные требования у TG/VK Mini Apps,
>>внешние API (ФНС, ЕСИА), OCR и работа с документами.
>Нужно:
>>Назвать 3 главных риска, которые вы видите для проекта на этапе MVP.
>>Кратко предложить, как их минимизировать (например: использовать
>>заглушки для API, начать с одной платформы, применить готовый OCR).
>