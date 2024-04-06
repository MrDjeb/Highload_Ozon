
                                          Highload Ozon


---
<details>

<summary>Содержание</summary>

#### 1. [Тема, целевая аудитория](#1)

#### 2. [Расчет нагрузки](#2)

#### 3. [Глобальная балансировка нагрузки](#3)
  
#### 4. [Локальная балансировка нагрузки](#4)
  
#### 5. [Логическая схема БД](#5)

#### 6. [Физическая схема БД](#6)
  
* * #### [Источники](#sources)

</details>

---

<h2 id="1">1. Тема, аудитория, функционал </h2>

### Тема

__Ozon__ — выход крупнейшего E-commerce России на рынок КНР.  
### Целевая аудитория  [[1](#sources)]
- Рынок КНР.
- Возрастная категория: 18-45 лет; 40% женщин, 60% мужчин.
- Уникальных пользователей:
	-  в месяц 70 млн. (MAU)
	-  в день 15 млн. (DAU)



### MVP
> - Каталог;
> - Поиск товаров в желаемой категории.

> - Корзина (просмотр, изменение);
> - Оформления заказа.

> - Cтраница товара;
> - Отзывы и рейтинг.

> - Список заказов;
> - Обновление статуса заказа в личном кабинете;
> - Трекинг заказа от службы доставки (обновление данных о статусе);
> - Рассылка Push-уведомлений об изменении статуса заказа.

### Логистика и фулфилмент
В рамках __MVP__ ограничимся двумя видами фулфилмента __FBS__ и __rFBS__, откажемся от __FBO__, т.е. от хранения товара на собственных складах.

  __FBS__ (_Fulfillment by Seller_) — 
cеллер хранит товар на собственном складе,
самомостоятельно собирает заказ и передаёте заказ в службу доставки.

__rFBS__ (_real FBS_) — схема работы, при которой продавец сам отвечает за хранение и доставку товара.
 


Схема __FBS__ от Ozon  [[11](#sources)]
![alt text](images/image-8.png)

---

<h2 id="2">2. Расчет нагрузки</h2>

### Продуктовые метрики

* Месячная аудитория маркетплейса 70 млн. пользователей [[1](#sources)]
* Дневная аудитория маркетплейса 15 млн. пользователей
- 500к заказов в сутки (80% от [[1](#sources)])
- 50к заказов в минуту в дни распродаж
- 200к активных продавцов [[12](#sources)]

Открытых источников информации нет. Данные о *среднем количестве действий пользователя по типам в день* получены в ходе опроса знакомых, пользующихся площадкой Ozon. 

#### Сценарий посещения сайта одним пользователем в течение дня

| Действие | Кол-во | В среднем |
| ---- | ---- | ---- |
| Авторизация | [0, 1]  | 0.5 |
| Просмотр отдельного товара,<br>каталога | [5, 55] | 30 |
| Добавление в корзину | [1, 5] | 3 |
| Отзыв и оценка | [0, 1] | 0.5 |
| Заказ товара | [0, 1] | 0.5 |
| Просмотр списка заказов | [1, 3] | 2 |

Средние количество действий за визит 37.
<br>
`37*DAU = 555 млн.`<br>
`555млн/(24*60*60) = 6500 RPS` - от пользователей

500к создаются в течение дня. Пусть в среднем жизненый цикл заказа 5 дней и его статус обновляется раз в сутки. Тогда действуюших заказов в среднем 2500к. 
`2500 RPS` - от логистики


### Объем изображений товаров
Ассортимент Ozon составил 250 млн товаров. Каждый товар содержит в среднем 4 фотографии, при этом каждое изображение весит в среднем 50 КБ. Отсюда получаем `50 * 4 * 250млн = 46,57 TБ`.

### Объем изображений в отзывах

Пусть 15% товаров имеют порядка 100 отзывов. И 1% отзывов из списка имеют [2, 7] фотографий. Тогда `250млн*0.15*(100*0.01*5) = 187.5млн` - изображений приходится на долю отзывов. Объём `187.5млн * 50 КБ = 8,73 ТБ`

---

## 3. Глобальная балансировка нагрузки <a name="3"></a>

Плотность населения КНР
![alt text](images/image-1.png)

---

## 4. Локальная балансировка нагрузки <a name="4"></a>

Возможные решения:
* *Weight round robin*
* *Consistent Hash*
* *Least Connected*
* *PeakEWMA*
#### Выберем *_PeakEWMA_*
Рассчитываем скользящее среднее
времени длительности запросов и,
исходя из этого, выбираем бэкенд, на
который вышлем нагрузку.
Данный алгоритм использует концепцию экспоненциально взвешенных скользящих средних для определения «пиковой» нагрузки серверов.
Присваивая веса недавним измерениям трафика, он точно фиксирует текущую нагрузку сервера и динамически корректирует свой выбор для входящих запросов.

Эксперимент с тремя алгоритмами: round robin, least loaded, and peak exponentially-weighted moving average (“peak EWMA”)
![alt text](images/image-2.png)

*PeakEWMA* показал наибольшее лучшии метрики при тестах в Ozon.
![alt text](images/image-3.png)

---

## 5. Логическая схема БД <a name="5"></a>

```mermaid
erDiagram
  PROFILE ||--o{ ORDER_INFO : includes
  PROFILE ||--o{ ADDRESS : includes
  PROFILE ||--o{ CART : includes

  ORDER_INFO ||--o{ ORDER_ITEM : includes

  PRODUCT ||--o{ ORDER_ITEM: includes
  PRODUCT ||--o{ SHOPPING_CART_ITEM : includes
  CATEGORY ||--|{ PRODUCT : includes
  STATUS ||--|{ ORDER_INFO : includes
  CART ||--o{ SHOPPING_CART_ITEM : includes

  COMMENT ||--o{ PROFILE : includes
  COMMENT ||--o{ PRODUCT : includes


  PROFILE {
    uuid id PK
    text login UK
    text description
    text imgsrc
	  text phone
    text passwordhash
  }

  PRODUCT {
    uuid id PK
    text name UK
    text description
    int price
    text imgsrc
    numeric rating
    uuid category FK
	int	count_comments
  }

  COMMENT {
    uuid id PK
    uuid product_id FK
    uuid profile_id FK
	text comment
	int rating
  }


  ORDER_INFO {
    uuid id PK
    uuid profile_id FK
    uuid promocode_id FK
    int status_id FK
	uuid address_id FK
    timestampz creation_at
    timestampz delivery_at
  }

  STATUS {
    serial id PK
    text name UK
  }

  ORDER_ITEM {
    uuid id PK
    uuid order_info_id FK
    uuid product_id FK
    int quantity
    int price
  }

  ADDRESS {
    uuid id PK
    uuid profile_id FK
    text city
    text street
    text house
    text flat
    bool is_current
  }

  CATEGORY {
    serial id PK
    text name UK
    int parent FK
  }

  CART {
    uuid id PK
    uuid profile_id FK
    bool is_current
  }
  
  SHOPPING_CART_ITEM {
    uuid id PK
    uuid cart_id FK
    uuid product_id FK
    int quantity
  }
```

 

| Type          | Byte size |
| ------------- | --------- |
| SERIAL        | 4         |
| UUID          | 16        |
| INT           | 4         |
| NUMERIC(3, 2) | 8         |
| timestampz    | 8         |
| boolean       | 1         |


| Table              | Row size [byte]                   | Number of row | Total |
| ------------------ | --------------------------------- | ------------- | ----- |
| PROFILE            | 16 + 32 + 128 + 64 + 19 +64 = 323 |               |       |
| PRODUCT            |                                   |               |       |
| COMMENT            |                                   |               |       |
| ORDER_INFO         |                                   |               |       |
| STATUS             |                                   |               |       |
| ORDER_ITEM         |                                   |               |       |
| ADDRESS            |                                   |               |       |
| CATEGORY           |                                   |               |       |
| CART               |                                   |               |       |
| SHOPPING_CART_ITEM |                                   |               |       |

---

## 6. Логическая схема БД <a name="6"></a>

Используем СУБД PostgreSQL и патерн Database Per Service.
#### Преимущества PostgreSQL:
* Поддержка БД неограниченного размера
* Мощные и надёжные механизмы транзакций и репликации
* Легко масштабировать 

### Потоковая репликация

#### Плюсы
* Работает из коробки.
* Годами обкатанная технология.
* Низкое потребление ресурсов, так как никакой логики
 при репликации нет, изменения выполняются в том же
порядке, что и на мастере.
* Простота конфигурации, настроил и забыл, простое
побайтовое копирование через WAL.

#### Минусы
* Реплицируется весь кластер целиком.
* Реплицируются все операции, включая ошибки.
* Изменения применяются в один поток.
* Работа только в рамках одной мажорной версии.
* Слейвы только read only.

![alt text](images/image-5.png)

Структура кластера:
Один мастер, одна синхронная релпика и несколько асинхронных.

Патерн работы с данными.
Пишем в мастер.
Если супер актуальные данные и нужно минимизировать отставания, то читаем с синхронной реплики. В ином случае с асинхронных.
Избегаем обильного чтения с мастера и по возможности с синхронной реплики.
Базовое правило — в мастер пишем, читаем только из слейвов. Сихроная реплица выполняет функцию фейловера, она всегда находиться в другом ДЦ.

### Партиционирование

#### Решаемые проблемы:
Если много удаляем/апдейтим записи в базе, то vacuum может работать довольно долго.
Операции insert/update перестраивают индексы, идет перебалансировка деревьев.

Засчёт партиционирования кол-во индексов будет намного меньше, vacuum работает быстро на маленьких таблицах, следовательно все проблемы больших таблиц будут в N раз меньше.

Кол-во партиций делаем не более 100-200.

### Шардинг

Шардинг и решардинг сложная процедура, и использование готовых инструментов для шардинга в случае непридвиденных ситуаций приводит к разбирательству с чёрным ящиком. Сделаем свой инструмент.

#### Получение физического адреса данных и решардинг.
Shard Key, к примеру, user_id. Берём остаток от деления по модулю на 1024 от user_id и в зависимости от интервала куда попадает резуьтат 0..127, 128..255, 256..383 и т.д. выбираем физисеский адрес. Следоват решардинг делаем в рамках одного кластера и для этого делим предыдущие интервалы попалам, получаем: 0..63, 64..127, 128..191 и т.д.

![alt text](images/image-4.png)

В итоге, из опыта Ozon, для таблицы _CART_ чтобы держать 20K RPS можно сделать 32 шарда: 1 синхронная реплика и 2 асинхронных реплики, 100 партиций: cart_0, cart_1, cart_2...

#### Асинхронное межсервисное взаимодействие. Сбор измененных данных с паттерном Outbox на Apache Kafka. 

Этот паттерн решенает проблемыу потери событий.
![alt text](images/image-6.png)
__Коньюмер-группа: Cервис отслеживания заказа__

![alt text](images/image-7.png)
__Внутреннее устройство__

Используем Батчинг + сжатие

#### Сетевая файловая система (CEPH)

---

## Источники <a name="sources"></a>
1. https://ozon.tech/
2. https://habr.com/ru/companies/ozontech/articles/667600/
3. https://www.youtube.com/watch?v=kIZ_4PNvkro
4. https://habr.com/ru/companies/ozontech/articles/749328/
5. https://linkerd.io/2016/03/16/beyond-round-robin-load-balancing-for-latency/
6. https://tenchat.ru/media/1400080-privet-bezuprechniy-balans-ili-kombinatsiya-peakewma-i-p2c-ot-twitter
7. https://super-video-tube.ru/video/7A7Cq9w0G9Y/ozon-tech-community-go-meetup/
8. https://speakerdeck.com/ozontech/dmitrii-loghovskii-kak-zastavit-vashu-bazu-dannykh-dierzhat-20k-rps-varianty-masshtabirovaniia-i-ikh-minusy
9. https://bigdataschool.ru/blog/transactional-outbox-pattern-on-neo4j-and-kafka.html
10. https://speakerdeck.com/ozontech/viktor-korieisha-camyie-rasprostraniennyie-oshibki-pri-rabotie-s-apache-kafka
11. https://seller-edu.ozon.ru/how-to-start/onboarding/step-4-choose-work-mode 
12. https://speakerdeck.com/ozontech/boris-kuzovatkin-put-posylki-kliuchievyie-tiekhnologhii-i-proiekty-v-loghistikie