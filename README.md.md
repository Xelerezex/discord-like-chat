
# Highload проект : высоконагруженный чат с разбивкой на Сервера/Гильдии 
  
---
# 1. Тема и целевая аудитория:

**Тип сервиса :**  B2C чат для больших сообществ.  
**Ключевая модель:** server-based "Гильдия/Сервер -> Канал -> Роль/Разрешение  -> Событие". 
**Реальные аналоги:**  
- Discord (глобальная пользовательская база ~200M+ MAU \[[1](https://discord.com/company)\]).
- Slack (глобальная пользовательская база ~6.4M MAU \[[2](https://slack.com/)\]).
- Microsoft Teams - похожая структура "рабочее пространство ->  каналы", но B2B. (глобальная пользовательская база ~320M MAU \[[3](https://office365itpros.com/2023/10/26/teams-number-of-users-320-million/)\])  
  
**Целевая аудитория Discord**:   
- **MAU(месячные активные пользователи)**: ~200,000,000+ / мес (глобально) \[[4](https://backlinko.com/discord-users)\].
- **DAU(дневные активные пользователи)**: ~26,000,000+ / день (глобально) \[[5](https://www.demandsage.com/discord-statistics/)\].
- Регионы: Северная/Южная Америка + Европа.

**Целевая аудитория Discord в России (на 2024)**:
- **MAU**: ~40,000,000+ / мес человек \[[6](https://www.kommersant.ru/doc/7165779)\] - \[[7](https://worldpopulationreview.com/country-rankings/discord-users-by-country)\] 
- **DAU**: ~4,000,000+ / день человек (10% от MAU)
# 2. MVP  - 7 основных функций:

1. **Регистрация/авторизация:** регистрация, вход, управление сессиями (JWT-токены и refresh-token).
2. **Гильдии/Серверы:** создание сервера (гильдии) и вход по инвайту. Список серверов пользователя.
3. **Каналы:** список каналов в сервере, фильтр по правам доступа (роль пользователя).
4. **История сообщений:** постраничная пагинация сообщений канала (GET `/channels/{id}/messages`).
5. **Отправка сообщений:** POST `/channels/{id}/messages` + мгновенная доставка по WebSocket.
6. **Статусы прочтения/упоминания:** отслеживание read/unread и упоминаний (@user) для каждого пользователя.
7. **Роли/Модерация:** создание ролей, настройка прав, бан/мут модераторов, поиск по чатам.

Каждая функция вдохновлена практиками Discord и Slack. 
Например, Discord хранит триллионы сообщений \[[8](https://discord.com/blog/maxjourney-pushing-discords-limits-with-a-million-plus-online-users-in-a-single-server)\] и применяет роли/каналы с ACL.
  
---  
---
# 3. Расчёт нагрузки:
## 3.1 Продуктовые метрики и допущения:

- **DAU = 10,000,000**  
- Открытия каналов (загрузка истории): `channels_open = 10 / user / day`  
- Запросы списков (guilds/channels): `channels_list = 5 / user / day`  
- [Доля пишущих сейчас (Discord benchmark of 30% communicators as a healthy goal)](https://discord.com/community/understanding-server-insights): `share_communicators = 0.30`, `DAU_communicators = 3,000,000`  
- Сообщений на пишущего (communicator): `messeges_per_communicator = 15 / day`  
  
### Продуктовые метрики:

| Метрика                   |          Значение |
| ------------------------- | ----------------: |
| MAU                       |  50,000,000 / мес |
| DAU                       | 10,000,000 / день |
| share_communicators       |              0.30 |
| DAU_communicators         |  3,000,000 / день |
| channels_open             |   10 / user / day |
| channels_list             |    5 / user / day |
| messeges_per_communicator |          15 / day |
  
## 3.2 RPS(запросов в секунду) по API:

Формулы:  
`request/day = DAU * actions_per_user_per_day`  
`RPS_average = request/day / 86400`  
`RPS_peak = RPS_average × 10` (пиковый коэффициент = 10)  
  
### RPS (average/peak):

| Операция                                           | request/day | RPS_average | RPS_peak |
| -------------------------------------------------- | ----------: | ----------: | -------: |
| GET /channels/{id}/messages (history)              | 100,000,000 |       1,157 |   11,570 |
| GET /me/guilds + GET /guilds/{id}/channels (lists) |  50,000,000 |         579 |    5,790 |
| POST /channels/{id}/messages (send)                |  45,000,000 |         521 |    5,210 |

## 3.3 Внутренний hot path (плотная нагрузка):

Допущение для рассылки (fan-out): `F = 20` доставок  на `1` сообщение.  

На **одно отправленное сообщение** извне - внутри системы превращается в пачку действий:
1. Записать в БД/лог (1 раз).
2. Отправить по WebSocket всем онлайн, кто видит канал (вот это и называется fan-out).
3. Обновить unread/mentions (read-states) для этих получателей (часто тоже “на каждого получателя”).

| Внутренние операции                                                |  events/day | RPS_average | RPS_peak |
| ------------------------------------------------------------------ | ----------: | ----------: | -------: |
| WebSocket deliveries = `messages/day * F`                          | 900,000,000 |      10,417 |  104,200 |
| Read-state (обновления счётчиков прочитки) = `~ messages/day *  F` | 900,000,000 |      10,417 |  104,200 |
|                                                                    |             |             |          |
  
---
# Источники:

1. Discord Inc., _"About Discord"_ (корпоративная страница, 2025): https://discord.com/company
2. Salesforce Inc., "Millions of people love to work in Slack" (корпоративная страница, 2026): https://slack.com/
3. Office 365. Teams Grows to 320 Million Monthly Active Users: https://office365itpros.com/2023/10/26/teams-number-of-users-320-million/.
4. Backlinko. Discord User and Funding Statistics: How Many People Use Discord: https://backlinko.com/discord-users
5. Demandsage. Discord Statistics 2026 (Users, Revenue & Market Share): https://www.demandsage.com/discord-statistics/
6. Коммерсантъ. С Discord сыграли в частичную блокировку: https://www.kommersant.ru/doc/7165779
7. World Population Review. Discord Users by Country 2026: https://worldpopulationreview.com/country-rankings/discord-users-by-country
8. Discord Inc., Maxjourney: Pushing Discord’s Limits with a Million+ Online Users in a Single Server: https://discord.com/blog/maxjourney-pushing-discords-limits-with-a-million-plus-online-users-in-a-single-server

9. https://discord.com/blog/why-discord-is-switching-from-go-to-rust
10. https://discord.com/blog/how-discord-stores-trillions-of-messages
11. https://support.discord.com/hc/en-us/articles/206141927-How-is-the-permission-hierarchy-structured
12. https://discord.com/community/understanding-server-insights