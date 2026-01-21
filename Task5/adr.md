### **Название задачи:**
Проектирование эволюции MVP-системы мониторинга свиноводческих ферм в SaaS-платформу

### **Автор:**
Бурганов И.И.

### **Дата:**
12 января 2026 г.

---

### **Функциональные требования**

| № | Действующие лица или системы | Use Case | Описание |
| :-: | :- | :- | :- |
| 1 | Клиент (агрохолдинг) | Управление фермами и пользователями | Настройка аккаунта, приглашение операторов, назначение ролей |
| 2 | Оператор фермы | Мониторинг и получение оповещений | Работа с локальными агентами, просмотр событий |
| 3 | Ветеринар | Анализ состояния поголовья | Получение отчётов и уведомлений о здоровье животных |
| 4 | Администратор платформы | Управление биллингом и подписками | Назначение тарифов, просмотр метрик использования |
| 5 | Система | Изоляция данных между клиентами | Полная разделённость данных, конфигураций, пользователей |
| 6 | Система | Интеграция с платёжными системами | Поддержка российских платёжных провайдеров |
| 7 | Разработчик клиента | Интеграция через API | Использование документации и dev-портала для подключения |

---

### **Нефункциональные требования**

| № | Требование                                                   |
|:-:|:-------------------------------------------------------------|
| 1 | Полная изоляция данных между клиентами                       |
| 2 | Поддержка гибких тарифных планов                             |
| 3 | Горизонтальная масштабируемость по числу клиентов и ферм     |
| 4 | Сбор метрик использования для биллинга и аналитики           |
| 5 | Поддержка российских платёжных систем (СБП, ЮKassa, Tinkoff) |
| 6 | Dev-портал с OpenAPI, примерами, SDK                         |

---

### **Решение**

На основе **основного решения**, выбранного в Задании 4, проектируем **SaaS-платформу**.

#### Задача 1. Проектирование мультитенантной архитектуры

Рассмотрены три подхода к изоляции данных:

| Вариант | Описание                                                             | Плюсы                                                                                               | Минусы                                                                  |
| :- |:---------------------------------------------------------------------|:----------------------------------------------------------------------------------------------------|:------------------------------------------------------------------------|
| **1. Изоляция на уровне схем БД** | Один экземпляр PostgreSQL, отдельная схема (`tenant_123`) на клиента | - Простота развёртывания<br> - Общая инфраструктура                                                 | - Сложность резервного копирования<br> - Риск утечки через SQL-инъекции |
| **2. Изоляция на уровне отдельных БД** | Отдельная БД (PostgreSQL) на клиента                                 | - Полная изоляция<br> - Гибкость резервного копирования<br> - Соответствие регуляторным требованиям | ️ - Увеличение накладных расходов на управление                         |
| **3. Изоляция на уровне инстансов** | Отдельный Kubernetes-namespace + отдельные сервисы на клиента        | - Максимальная изоляция<br> - Полный контроль SLA                                                   | - Очень высокий TCO<br> - Сложность управления                          |

**Выбран вариант 2 — изоляция на уровне отдельных БД.**  
Обоснование:
- Соответствует требованиям **российского законодательства** (ФЗ-152, изоляция персональных/хозяйственных данных).
- Позволяет легко реализовать **разные SLA** (например, «Премиум» клиенты — SSD-хранилище, «Стандарт» — HDD).
- Поддерживает **гибкий биллинг** (стоимость зависит от объёма данных и количества ферм).

##### Диаграмма C2 — Multi-tenancy (вариант 2)
```plantuml
@startuml
title AgroPig SaaS — Multi-tenancy (отдельные БД)

!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

Person(customer_admin, "Админ клиента", "Управляет фермами и пользователями")
Person(operator, "Оператор фермы", "Мониторинг")
System_Ext(edge_farms, "Фермы клиента", "Edge-агенты")

Container_Boundary(platform, "AgroPig SaaS") {
    Container(api_gateway, "API Gateway", "Kong", "Единая точка входа с tenant routing")
    Container(tenant_router, "Tenant Router", "Java/Spring Boot", "Извлекает tenant_id из JWT / URL")
    Container(user_service, "User & Auth Service", "Keycloak + Custom", "RBAC по tenant")
    Container(analytics, "Analytics Service", "Java/Spring Boot", "Работает с tenant-specific DB")
    Container(billing, "Billing Service", "Java/Spring Boot", "Тарификация и биллинг")
    ContainerDb(tenant_db_1, "БД клиента A", "PostgreSQL", "Изоляция на уровне БД")
    ContainerDb(tenant_db_2, "БД клиента B", "PostgreSQL", "Изоляция на уровне БД")
    ContainerDb(shared_minio, "Хранилище видео", "MinIO", "Бакеты изолированы по tenant_id")
}

Rel(customer_admin, api_gateway, "Управление аккаунтом", "HTTPS")
Rel(operator, api_gateway, "Мониторинг", "HTTPS")
Rel(edge_farms, api_gateway, "Отправка телеметрии", "HTTPS/Kafka")
Rel(api_gateway, tenant_router, "Маршрутизация", "internal")
Rel(tenant_router, analytics, "tenant_id", "gRPC")
Rel(tenant_router, user_service, "tenant_id", "gRPC")
Rel(analytics, tenant_db_1, "tenant-specific", "JDBC")
Rel(analytics, tenant_db_2, "tenant-specific", "JDBC")
Rel(edge_farms, shared_minio, "Загрузка видео", "S3 (bucket=tenant-id)")

note right of tenant_router
  Tenant ID извлекается из:
  - subdomain: clientA.agropig.ru
  - JWT claim: tenant_id
  - API header: X-Tenant-ID
end note

@enduml
```

---

#### Задача 2. Система биллинга и монетизации

Введены новые сервисы:

- **Billing Service** — управляет подписками, тарифами, выставляет счёты.
- **Usage Collector** — собирает метрики: число ферм, камер, событий, объём хранилища.
- **Payment Gateway Adapter** — интеграция с российскими платёжными системами.

Поддерживаемые провайдеры:
- **ЮKassa** (Яндекс.Касса)
- **Tinkoff Acquiring**
- **СБП** (Система быстрых платежей)

##### Интеграция с платёжными системами
```plantuml
@startuml
title Billing Integration

!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

Container(billing_svc, "Billing Service", "Java/Spring Boot")
Container(yookassa, "ЮKassa", "External")
Container(tinkoff, "Tinkoff", "External")
Container(sbp, "СБП", "External")

Rel(billing_svc, yookassa, "Создание платежа", "REST API")
Rel(billing_svc, tinkoff, "Создание платежа", "REST API")
Rel(billing_svc, sbp, "Создание QR/перевода", "Open API")

note right
  Все интеграции через адаптеры,
  поддерживающие retry, idempotency,
  webhook verification.
end note

@enduml
```

---

#### Задача 3. Интеграции с клиентами

- **API для клиентов SaaS-платформы**:
    - `/api/v1/farms` — управление фермами
    - `/api/v1/alerts` — получение событий
    - `/api/v1/metrics` — экспорт метрик
    - `/api/v1/equipment` — управление кормушками 
- **Scope функционала (по тарифным планам)**:

| Функционал |          Базовый          |                Стандарт                |                      Премиум                       |
| :- |:-------------------------:|:--------------------------------------:|:--------------------------------------------------:|
| **Количество ферм** |           До 3            |                 До 20                  |                    Неограничено                    |
| **Количество камер на ферму** |           До 4            |                 До 12                  |                       До 32                        |
| **Видеоаналитика в реальном времени** | Только обнаружение гибели |         + беспокойство, драки          |     + задавливание поросят, пересчёт поголовья     |
| **Управление оборудованием** |        Недоступно         |       Кормушки и поилки (on/off)       |         Полный контроль + прогноз расхода          |
| **Мониторинг систем фильтрации воды** |                           |           Основные параметры           |              Расширенная диагностика               |
| **Оповещения** |           Email           |              Email + Push              |            Email + Push + SMS + Webhook            |
| **Хранилище видео (на событие)** |          24 часа          |                 7 дней                 |                      30 дней                       |
| **Аналитика и отчёты** |     Ежедневный отчёт      |           + еженедельные KPI           |          + прогнозирование, custom-отчёты          |
| **Интеграционный API** |                           |  Только чтение (`/alerts`, `/metrics`) |  Полный доступ (чтение + управление оборудованием) |
| **RBAC (роли пользователей)** |     Админ + Оператор      |              + Ветеринар               |                  + Кастомные роли                  |
| **SLA** |            99%            |                 99,5%                  |                       99,95%                       |
| **Поддержка** |       Email (48 ч)        |           Email + чат (24 ч)           |             24/7 + выделенный менеджер             |

- **Dev-портал**: на базе **Redocly** или **Swagger UI**, с примерами.

---

#### Задача 4. Финальная архитектура To-Be

##### C1 — SaaS-платформа
```plantuml
@startuml
title AgroPig SaaS — Контекстная диаграмма

!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

Person(admin, "Админ клиента")
Person(operator, "Оператор фермы")
Person(vet, "Ветеринар")
Person(platform_owner, "Владелец платформы")

System(agropig_saas, "AgroPig SaaS", "Мультитенантная платформа мониторинга")
System_Ext(farms, "Фермы клиентов", "Edge-агенты")

Rel(admin, agropig_saas, "Управление аккаунтом")
Rel(operator, agropig_saas, "Мониторинг")
Rel(vet, agropig_saas, "Получение отчётов")
Rel(farms, agropig_saas, "Телеметрия и управление")
Rel(platform_owner, agropig_saas, "Биллинг, мониторинг")

@enduml
```

##### C2 — Интеграция с компанией «АгроТех Х» как клиентом
```plantuml
@startuml
title Интеграция «АгроТех Х» как клиента SaaS

!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

System_Boundary(agrotech_client, "Клиент: АгроТех Х") {
    Container(farm_agents, "Edge-агенты на 150 фермах", "Java/Spring Boot")
    Container(mobile_app, "Мобильное приложение", "Flutter")
}

System_Boundary(saas_platform, "AgroPig SaaS") {
    Container(api_gw, "API Gateway", "Kong")
    Container(tenant_router, "Tenant Router", "Java")
    ContainerDb(tenant_db_agrotech, "БД АгроТех Х", "PostgreSQL")
    Container(analytics, "Analytics", "Java")
    Container(billing, "Billing", "Java")
}

Rel(farm_agents, api_gw, "Телеметрия", "HTTPS")
Rel(mobile_app, api_gw, "Управление", "HTTPS")
Rel(api_gw, tenant_router, "Маршрутизация", "internal")
Rel(tenant_router, analytics, "tenant=agrotech", "gRPC")
Rel(analytics, tenant_db_agrotech, "Данные", "JDBC")
Rel(billing, tenant_db_agrotech, "Метрики использования", "JDBC")

note right of tenant_db_agrotech
  Tenant ID: agrotech-x
  БД полностью изолирована
  от других клиентов.
end note

@enduml
```

---

### **Итог**

- Выбрана **мультитенантная архитектура с изоляцией на уровне БД**.
- Реализованы **биллинг**, **платёжные интеграции**, **dev-портал**.
- Архитектура **горизонтально масштабируема** и соответствует **российскому законодательству**.
- Компания «АгроТех Х» может быть первым клиентом платформы — без изменений в edge-агентах.
