# Задание 1. Анализ и планирование

### 1. Описание функциональности монолитного приложения

**Управление отоплением:**

- Пользователи могут удалённо включать/выключать отопление в своих домах
- Пользователи могут удалённо устанавливать температуру отопления в своих домах
- Система поддерживает возможность включения/выключения отопления

**Мониторинг температуры:**

- Пользователи могут просматривать текущую температуру в своих домах через веб-интерфейс
- Система поддерживает запрос/получение данных о температуре с установленных датчиков и сохранение их в БД

### 2. Анализ архитектуры монолитного приложения

- Язык программирования: Go
- База данных: PostgreSQL
- Архитектура: Монолитная, все компоненты системы (обработка запросов, бизнес-логика, работа с данными) находятся в рамках одного приложения.
- Взаимодействие: Синхронное, запросы обрабатываются последовательно.
- Коммуникации: Синхронные HTTP-запросы; внутренняя логика и доступ к данным — в одном процессе
- Развёртывание: Релизы требуют перезапуска всего приложения
- Масштабирование: Только вертикальное
- Наблюдаемость: базовые логи

### 3. Определение доменов и границы контекстов

- Управление пользователями и домами - Регистрация и управление пользователями (настройка прав), добавление сущностей дом, устройство.
- Управление устройствами - Непосредственное управление устройством - отправка команд на устройство.
- Мониторинг устройств и телеметрия - Настройка Healthcheck, получение данных с устройств, получение информации о состоянии устройства.

### **4. Проблемы монолитного решения**

- Невозможно легко подключить новые устройства/датчики - Подключение возможно только через выезд специалиста.
- Ограниченная масштабируемость: Невозможно масштабировать только нагруженные части системы.
- Коммуникации: Нет брокера сообщений или асинхронного обмена данными - при росте количества датчиков/пользователей в системе будут накапливаться очереди исполнения, отклик системы в целом будет увеличиваться. Падение приложения приведет к потере всех данных в очереди исполнения.
- Развёртывание: Обновление и добавление функционала требует полной остановки приложения.
- Совместная работа над проектом затруднена из-за общей кодовой базы. Добавление нового функционала потребует изменения кодовой базы. Сложно распараллелить процесс добавления нового функционала и потребует синхронизации команд. Это увеличивает объем тестирования, риск влзникновения ошибок и увеличивает сроки разработки и поставки обновлений.

### 5. Визуализация контекста системы — диаграмма С4

[C4 Context Diagram](./Schemes/C4_context_Old.png)

<details>
  <summary>PlantUML Source Code:</summary>

```plantuml
@startuml
title Warmhouse Context Diagram

!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

Person(user, "Пользователь", "Пользователь использующий систему")
Person(admin, "Администратор", "Администратор выполняющий управление системой")
System(WarmHouseSystem, "WarmHouseSystem", "Система управления отоплением")

System_Ext(sensor, "Датчик", "API регулировки на устройстве")

Rel(user, WarmHouseSystem, "Включение/выключение отопления")
Rel(user, WarmHouseSystem, "Установка температуры")
Rel(user, WarmHouseSystem, "Просмотр температуры")
Rel(admin, WarmHouseSystem, "Регистрация датчиков")

Rel(admin, WarmHouseSystem, "Мониторинг датчиков")
Rel(admin, WarmHouseSystem, "Администрирование системы")

Rel(WarmHouseSystem, sensor, "Команды управления датчиком")

@enduml
```
</details>

# Задание 2. Проектирование микросервисной архитектуры

**Диаграмма контейнеров (Containers)**

[C4 Container Diagram](./Schemes/C4_container.png)

<details>
  <summary>PlantUML Source Code:</summary>

```plantuml
@startuml
title Warmhouse Container Diagram

!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

Person(user, "Пользователь", "Пользователь использующий систему")
Person(admin, "Администратор", "Администратор выполняющий управление системой")

'Внешние устройства умного дома
System_Ext(sensor1, "Устройство типа 1", "API на устройстве")
System_Ext(sensor2, "Устройство типа 2", "API на устройстве")
System_Ext(sensor3, "Устройство типа 3", "API на устройстве")

Container_Boundary(WarmHouseSystem, "WarmHouseSystem") {
    Container(WebUI, "WebUI", "UI для управления системой")
    Container(APIGate, "API", "API взаимодействия")
    Container(IOTGate, "IOT Gate", "Gate для устройств")

    'Функционал работы с учетными данными Пользователей, Домов, Устройств
    Container_Boundary(AccountMGM, "Account Management"){
        Container(AccountCore, "Account System", "Система управления аккаунтами")
        ContainerDb(AccountDb, "Account Database", "PostgreSQL", "База данных о пользователях, домах и зарегистрированных устройствах")
    }

    'Функционал работы с пользовательскими сценариями умного дома
    Container_Boundary(ScenarioMGM, "Scenario Engine"){
        Container(ScenarioEngine, "ScenarioEngine", "Система создания и управления пользовательскими сценариями")
        ContainerDb(ScenarioDb, "Scenario Database", "PostgreSQL", "База данных пользовательских сценариев")
    }

    'Функционал работы с устройствами умного дома
    Container_Boundary(DeviceMGM, "Device Management") {
        Container(Management, "Device Management", "Система конфигурации и управления")
        Container(Telemetry, "Device Telemetry", "Система мониторинга и телеметрии")
        ContainerDb(DeviceDb, "Device Database", "PostgreSQL", "База конфигурации устройств и телеметрии")
    }
    Container(MessageBroker, "Message Broker", "RabbitMQ",  "Брокер сообщений")
}

'Взаимодействие пользователя и Web UI системы
Rel(user, WebUI, "Регистрация пользователя\дома\устройства")
Rel(user, WebUI, "Команды управления устройством")
Rel(user, WebUI, "Мониторинг и просмотр телеметрии устройства")
Rel(user, WebUI, "Настройка сценариев умного дома")

'Взаимодействие администратора и Web UI системы
Rel(admin, WebUI, "Мониторинг датчиков")
Rel(admin, WebUI, "Администрирование системы")

'Передача запросов пользователя и администратора для маршрутизации через API-Gate
Rel(WebUI, APIGate, "Запросы")

'Маршрутизация запросов через API-Gate:
'1. Маршрутизация запросов работы с учетными данными Пользователей, Домов, Устройств
Rel(APIGate, AccountCore, "Регистрация пользователя\дома\устройства")
Rel(AccountCore, AccountDb, "CRUD")

'2. Маршрутизация запросов работы с пользовательскими сценариями умного дома
Rel(APIGate, ScenarioEngine, "Настройка сценариев умного дома")
Rel(ScenarioEngine, ScenarioDb, "CRUD")
Rel(ScenarioEngine, Management, "Команды управления устройством")

'3. Маршрутизация запросов работы с устройствами умного дома
Rel(APIGate, Management, "Команды управления устройством")
Rel(Management, DeviceDb, "Команды управления устройством")
Rel(Management, MessageBroker, "Команды управления устройством")
Rel(APIGate, Telemetry, "Мониторинг и просмотр телеметрии устройства")
Rel(MessageBroker, Telemetry, "Данные и телеметрия")
Rel(Telemetry, DeviceDb, "Данные и телеметрия")

'Передача запросов работы с устройствами умного дома для маршрутизации через IOT-Gate и получение данных и телеметрии с устройств умного дома
Rel(MessageBroker, IOTGate, "Команды управления устройством")
Rel(IOTGate, MessageBroker, "Данные и телеметрия")

'Маршрутизация запросов работы с устройствами умного дома
Rel(IOTGate, sensor1, "Команды управления устройством")
Rel(IOTGate, sensor2, "Команды управления устройством")
Rel(IOTGate, sensor3, "Команды управления устройством")

'Передача данных и телеметрии с устройств умного дома
Rel(sensor1, IOTGate, "Данные и телеметрия")
Rel(sensor2, IOTGate, "Данные и телеметрия")
Rel(sensor3, IOTGate, "Данные и телеметрия")
@enduml
```
</details>



**Диаграмма компонентов (Components)**

Добавьте диаграмму для каждого из выделенных микросервисов.

[C4 Component Diagram AccountMGM — Система управления аккаунтами](./Schemes/C4_Component_AccountMGM.png)

<details>
  <summary>PlantUML Source Code:</summary>

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title C4_Component_Diagram_AccountMGM — Система управления аккаунтами

Person(user, "Пользователь", "Пользователь использующий систему")
Person(admin, "Администратор", "Администратор выполняющий управление системой")

System_Ext(WebUI, "WebUI", "UI для управления системой")
System_Ext(APIGate, "API", "API взаимодействия")

Container_Boundary(AccountMGM, "Система управления аккаунтами") {
    Component(AccountCore, "Account Core Service", "Бизнес-логика", "CRUD операции с Пользователями, Домами, Устройствами")
    ComponentDb(AccountDb, "Account Database", "PostgreSQL", "Хранение таблиц Пользователей, Домов, Устройств и связей между ними")
    Component(AuthService, "Authorization Service", "Осуществление авторизации, управление токенами, ролями и правами доступа")
    Component(AccountBroker, "Message Broker для событий пользователей", "Message Broker", "Публикация событий с Пользователями, Домами, Устройствами для других сервисов")
    Component(LogService, "Logger Service", "Фоновый процесс", "Логирование действий пользователей для EventLog")
}

Rel(user, WebUI, "Выполнение действий с данными Пользователей, Домов, Устройств")
Rel(admin, WebUI, "Выполнение администрирования")
Rel(WebUI, APIGate, "Отправка запросов")
Rel(APIGate, AccountCore, "Вызов логики")
Rel(AccountCore, AccountDb, " CRUD работа с учетными данными Пользователей, Домов, Устройств")
Rel(AccountCore, AuthService, "Осуществление авторизации, управление токенами, ролями и правами доступа")
Rel(AccountCore, AccountBroker, "Публикация событий с Пользователями, Домами, Устройствами для других сервисов)")
Rel(AccountCore, LogService, "Передача данных об изменениях для EventLog")

@enduml
```
</details>

**Диаграмма кода (Code)**

Добавьте одну диаграмму или несколько.

# Задание 3. Разработка ER-диаграммы

Добавьте сюда ER-диаграмму. Она должна отражать ключевые сущности системы, их атрибуты и тип связей между ними.

# Задание 4. Создание и документирование API

### 1. Тип API

Укажите, какой тип API вы будете использовать для взаимодействия микросервисов. Объясните своё решение.

### 2. Документация API

Здесь приложите ссылки на документацию API для микросервисов, которые вы спроектировали в первой части проектной работы. Для документирования используйте Swagger/OpenAPI или AsyncAPI.

# Задание 5. Работа с docker и docker-compose

Перейдите в apps.

Там находится приложение-монолит для работы с датчиками температуры. В README.md описано как запустить решение.

Вам нужно:

1) сделать простое приложение temperature-api на любом удобном для вас языке программирования, которое при запросе /temperature?location= будет отдавать рандомное значение температуры.

Locations - название комнаты, sensorId - идентификатор названия комнаты

```
// If no location is provided, use a default based on sensor ID
	if location == "" {
		switch sensorID {
		case "1":
			location = "Living Room"
		case "2":
			location = "Bedroom"
		case "3":
			location = "Kitchen"
		default:
			location = "Unknown"
		}
	}

	// If no sensor ID is provided, generate one based on location
	if sensorID == "" {
		switch location {
		case "Living Room":
			sensorID = "1"
		case "Bedroom":
			sensorID = "2"
		case "Kitchen":
			sensorID = "3"
		default:
			sensorID = "0"
		}
	}
```

2) Приложение следует упаковать в Docker и добавить в docker-compose. Порт по умолчанию должен быть 8081
3) Кроме того для smart_home приложения требуется база данных - добавьте в docker-compose файл настройки для запуска postgres с указанием скрипта инициализации ./smart_home/init.sql

Для проверки можно использовать Postman коллекцию smarthome-api.postman_collection.json и вызвать:

- Create Sensor
- Get All Sensors

Должно при каждом вызове отображаться разное значение температуры

Ревьюер будет проверять точно так же.
