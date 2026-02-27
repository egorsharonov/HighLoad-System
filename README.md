# Проектирование высоконагруженной системы: PriceCompare

Курсовой проект по дисциплине «Проектирование высоконагруженных систем» (НИУ ВШЭ).

PriceCompare — веб‑сервис сравнения цен по модели idealo: пользователь ищет товар, сравнивает предложения (offers) разных магазинов, смотрит историю цены, добавляет товар в избранное/уведомления и переходит в магазин для покупки (click‑out).

## Содержание

- [1. Тема и целевая аудитория](#1-тема-и-целевая-аудитория)
  - [1.1 Короткое описание сервиса](#11-короткое-описание-сервиса)
  - [1.2 Почему тема подходит под highload](#12-почему-тема-подходит-под-highload)
  - [1.3 Отличительные черты сервиса](#13-отличительные-черты-сервиса)
  - [1.4 Целевая аудитория](#14-целевая-аудитория)
  - [1.5 MVP‑функционал](#15-mvpфункционал)
  - [1.6 Ключевые продуктовые решения](#16-ключевые-продуктовые-решения)
  - [1.7 Термины](#17-термины)
- [2. Расчёт нагрузки](#2-расчёт-нагрузки)
  - [2.1 Продуктовые метрики](#21-продуктовые-метрики)
  - [2.2 Технические метрики](#22-технические-метрики)
- [3. Глобальная балансировка нагрузки](#3-глобальная-балансировка-нагрузки)
  - [3.1 Функциональное разбиение по доменам](#31-функциональное-разбиение-по-доменам)
  - [3.2 Обоснование расположения ДЦ](#32-обоснование-расположения-дц)
  - [3.3 Распределение запросов по ДЦ](#33-распределение-запросов-по-дц)
  - [3.4 DNS‑балансировка](#34-dnsбалансировка)
  - [3.5 Anycast‑балансировка](#35-anycastбалансировка)
  - [3.6 Механизм регулировки трафика между ДЦ](#36-механизм-регулировки-трафика-между-дц)
- [4. Источники](#4-источники)

## 1. Тема и целевая аудитория

### 1.1 Короткое описание сервиса

PriceCompare — сервис сравнения цен (price comparison).

Основной пользовательский сценарий:
1) поиск товара (по строке/категории/фильтрам),
2) карточка товара,
3) сравнение предложений магазинов (цена/доставка/наличие/продавец),
4) история цены (price history),
5) избранное и price alert,
6) переход в магазин (click‑out).

### 1.2 Почему тема подходит под highload

Аналог idealo находится на масштабе десятков миллионов пользователей в месяц и сотен миллионов предложений.

Публично подтверждённые масштабы idealo:
- 78 млн визитов/мес в Германии и 96 млн визитов/мес суммарно в 5 других странах (FR, UK, IT, AT, ES) [2];
- >606 млн предложений (offers) от ~50 000 магазинов [2];
- >4 млн товаров в каталоге [4][5];
- ёмкость внутреннего хранилища под нагрузкой: до 200 000 queries/s и до 60 000 updates/s (кейс AWS) [1].

### 1.3 Отличительные черты сервиса

1) Агрегация предложений продавцов (offer store) и нормализация данных.
2) Высокая динамика цен/наличия и необходимость быстро отражать обновления.
3) Разделение heavy‑трафика (статические ресурсы/изображения) и динамики (поиск/карточки/сравнение/история/события).
4) Click‑out как важный бизнес‑событийный поток для аналитики и атрибуции.

### 1.4 Целевая аудитория

Масштаб целевой аудитории берётся по аналогам idealo (Европа, 6 стран: Германия, Австрия, Великобритания, Испания, Франция, Италия) [3][4].

- MAU: 72 000 000 пользователей/месяц (ориентир по кейсу AWS) [1]
- DAU: 2 500 000 пользователей/день (ориентир по отчётам idealo) [3][4]

### 1.5 MVP‑функционал

1. Поиск и выдача товаров (фильтры/сортировка).
2. Карточка товара.
3. Список предложений (offers) по товару.
4. История цены (price history).
5. Избранное (wishlist/favorites).
6. Price alert (уведомление о снижении цены).
7. Click‑out (переход в магазин).

### 1.6 Ключевые продуктовые решения

- Нейтральное ранжирование: “топ‑позиции не покупаются”, сравнение идёт по цене/условиям.
- Mobile‑first производительность: существенная доля customer journey начинается на мобильных устройствах (58%) [10].
- Разделение статики и динамики: heavy‑контент уходит в CDN, динамика остаётся в API/SSR.
- Click‑out — отдельный поток событий (аналитика, антифрод, партнёрская атрибуция).

### 1.7 Термины

- Product — товар в каталоге.
- Offer — предложение продавца по товару (цена, доставка, наличие, ссылка).
- Click‑out — переход пользователя в магазин по выбранному offer.
- Price history — временной ряд цен по товару (для графика).
- Price alert — правило уведомления о снижении цены.

## 2. Расчёт нагрузки

### 2.1 Продуктовые метрики

Опорные значения (по idealo, публичные источники):

| Метрика | Значение |
|---|---:|
| MAU | 72 000 000 [1] |
| DAU | 2 500 000 [3][4] |
| Визиты/месяц в Германии | 78 000 000 [2] |
| Визиты/месяц суммарно в 5 других странах | 96 000 000 [2] |
| Offers (предложения) | 606 000 000 [2] |
| Shops/Merchants | 50 000 [2] |
| Products (товары) | 4 000 000+ [4][5] |
| Среднее page views per session (benchmark) | 4,48 [7] |
| Доля customer journey, стартующая на mobile | 58% [10] |

Производные метрики:

- Page views/day = DAU * pageviews/session = 2 500 000 * 4,48 = 11 200 000 просмотров страниц/сутки.

Для разбиения по страницам MVP (инженерное допущение):
- Search: 30%
- Product: 40%
- Offers: 20%
- Price history: 10%

Тогда:

| Тип действия | Действий/сутки |
|---|---:|
| Search | 3 360 000 |
| Product page | 4 480 000 |
| Offers list | 2 240 000 |
| Price history | 1 120 000 |

Действия, которые создают write‑нагрузку (инженерные допущения, калиброваны сезонной статистикой):
- Click‑out: 25% сессий → 625 000/сутки
- Wishlist add/remove: 1% product views → 44 800/сутки
- Price alert create/delete: 0,5% product views → 22 400/сутки

### 2.2 Технические метрики

#### 2.2.1 RPS по основным запросам (средний и пиковый)

Формулы:
- RPS_avg = N_day / 86 400
- RPS_peak = 3 * RPS_avg (k_peak = 3 как консервативная оценка суточного пика для одного региона; термин peak‑to‑mean ratio) [9]

| Запрос | N_day | RPS_avg | RPS_peak |
|---|---:|---:|---:|
| Search | 3 360 000 | 38,89 | 116,67 |
| Product | 4 480 000 | 51,85 | 155,56 |
| Offers | 2 240 000 | 25,93 | 77,78 |
| Price history | 1 120 000 | 12,96 | 38,89 |
| Wishlist | 44 800 | 0,52 | 1,56 |
| Price alert | 22 400 | 0,26 | 0,78 |
| Click‑out | 625 000 | 7,23 | 21,70 |

Дополнительно (ориентир внутренней нагрузки offer store по публичному кейсу AWS):
- до 200 000 queries/s (peak) [1]
- до 60 000 updates/s (peak) [1]

#### 2.2.2 RPS для статики/изображений (CDN)

По данным HTTP Archive Web Almanac (медиана inner page): 71 запрос на страницу — 3 HTML, 4 fonts, 8 CSS, 13 images, 23 JS и прочие [8].

| Тип ресурса | Запросов/страница | Запросов/сутки | RPS_avg | RPS_peak |
|---|---:|---:|---:|---:|
| HTML | 3 | 33 600 000 | 388,89 | 1 166,67 |
| JS | 23 | 257 600 000 | 2 981,48 | 8 944,44 |
| CSS | 8 | 89 600 000 | 1 037,04 | 3 111,11 |
| Fonts | 4 | 44 800 000 | 518,52 | 1 555,56 |
| Images | 13 | 145 600 000 | 1 685,19 | 5 055,56 |
| Other | 20 | 224 000 000 | 2 592,59 | 7 777,78 |
| **Итого** | **71** | **795 200 000** | **9 203,70** | **27 611,11** |

#### 2.2.3 Сетевой трафик

По данным HTTP Archive Web Almanac (медиана inner page): 1,8 MB mobile и 2,0 MB desktop [8].

Доля mobile трафика: 60% (ориентир по доле mobile customer journeys 58%) [10].

Объём трафика страниц:
- 11,2 млн page views/day * (0,6*1,8MB + 0,4*2,0MB) ≈ 21,1 TB/day

Разбиение (оценка на основании медианного вклада HTML в вес страницы):
- Origin (HTML): ~0,23 TB/day
- CDN (остальное): ~20,8 TB/day

Средняя полоса:
- CDN BW_avg ≈ 1,93 Gbit/s, пиковая (×3) ≈ 5,79 Gbit/s
- Origin BW_avg ≈ 0,021 Gbit/s, пиковая (×3) ≈ 0,063 Gbit/s

#### 2.2.4 Размер хранения (существенные блоки)

База (твёрдые количества по источникам idealo):
- Offers: 606 млн [2]
- Products: 4 млн+ [4][5]
- Shops: 50 тыс [2]

Инженерные оценки размера элемента:
- Offer: 2 KB
- Product: 4 KB
- Shop: 2 KB
- Price history: 12 KB/товар (2 года, агрегация по дням)

Пользовательские данные:
- 10% от MAU имеют аккаунт (оценка)
- Wishlist: 20 товаров на пользователя
- Alerts: 5 правил на пользователя

| Тип данных | Кол-во | Размер элемента | Хранение |
|---|---:|---:|---:|
| Offers | 606 000 000 [2] | 2 KB | ~1,2 TB |
| Products | 4 000 000 [4][5] | 4 KB | ~16 GB |
| Shops | 50 000 [2] | 2 KB | ~0,1 GB |
| Price history | 4 000 000 [4][5] | 12 KB | ~48 GB |
| User profiles | 7 200 000 | 1 KB | ~7,2 GB |
| Wishlist records | 144 000 000 | 24 B | ~3,5 GB |
| Price alerts | 36 000 000 | 64 B | ~2,3 GB |

Сезонная «массовость» функций wishlist/alerts подтверждается пресс‑материалами idealo: в октябре–ноябре 2025 в Германии было создано >3 млн price alerts и добавлено >5,2 млн товаров в wishlists [6].

## 3. Глобальная балансировка нагрузки

### 3.1 Функциональное разбиение по доменам

| Домен | Назначение | Тип трафика |
|---|---|---|
| www.pricecompare.example | Web/SSR (HTML) | частично кэшируемый |
| api.pricecompare.example | API (search/product/offers/history/wishlist/alert/clickout) | динамический |
| static.pricecompare.example | JS/CSS/fonts | CDN |
| img.pricecompare.example | изображения товаров / логотипы магазинов | CDN |
| events.pricecompare.example | сбор событий (click‑out, аналитика) | write‑поток |
| merchant.pricecompare.example | B2B: загрузка фидов/обновлений продавцов | write‑поток |

### 3.2 Обоснование расположения ДЦ

Аудитория распределена по Европе (6 стран) [3][4]. По визитам idealo публикует разрез: Германия 78 млн/мес и остальные 5 стран 96 млн/мес суммарно [2].

Выбор ДЦ (MVP):
- Frankfurt — основной (центр Европы, минимизация задержек для DE/AT и части EU)
- Dublin — резервный (гео‑резерв, хороший маршрут для UK и западной Европы)

Статика/изображения обслуживаются edge‑узлами CDN (Anycast), поэтому не привязаны к ДЦ [13].

### 3.3 Распределение запросов по ДЦ

Базовая пропорция для динамики (по доле визитов DE vs остальные):
- Frankfurt: 45%
- Dublin: 55%

Расчёт: 78 / (78 + 96) ≈ 45% [2].

CDN‑трафик распределяется по edge автоматически.

Failover:
- при недоступности Frankfurt — 0% / 100% на Dublin
- при восстановлении — плавный возврат (weighted routing)

### 3.4 DNS‑балансировка

Для www/api:
- latency‑based routing (DNS выбирает регион с минимальной задержкой) [11]
- health checks + failover [12]
- низкий TTL для более быстрого переключения

### 3.5 Anycast‑балансировка

Anycast используется на уровне CDN для static/img [13].

Для api в MVP Anycast не обязателен; как опция — AWS Global Accelerator (anycast IP на edge сети) [14].

### 3.6 Механизм регулировки трафика между ДЦ

- автоматический failover по health check [12]
- weighted routing для ручного/автоматического изменения долей (degradation / recovery) [11]

## 4. Источники

1. AWS Case Study (July 2024): idealo Increases Traffic 6x for Black Friday Using MongoDB Atlas on AWS — https://aws.amazon.com/partners/success/idealo-mongodb/
2. idealo (Unternehmen): Über idealo (78 млн визитов/мес DE; 96 млн визитов/мес в 5 странах; 606 млн offers; 50k shops) — https://www.idealo.de/unternehmen/ueber-idealo
3. idealo Nachhaltigkeitsbericht 2022 (2,5 млн visitors/day; 500 млн offers; 6 стран) — https://www.idealo.de/dam/jcr:9b512f66-82c8-4616-921f-26b77da39c7f/idealo_nachhaltigkeitsbericht_2022.pdf
4. idealo Nachhaltigkeitsbericht 2023 (2,5 млн visits/day; >560 млн offers; 50k shops; 6 стран; >4 млн products) — https://www.idealo.de/dam/jcr:f8556c6b-9dd7-4a52-bec4-158f0d3ab899/idealo_Nachhaltigkeitsbericht_2023.pdf
5. idealo Nachhaltigkeit (>4 млн products; >500 млн offers; 50k merchants) — https://www.idealo.de/unternehmen/nachhaltigkeit
6. idealo Presseinfo (05.12.2025): >3 млн price alerts и >5,2 млн wishlist adds (окт–ноя, Германия) — https://www.idealo.de/dam/jcr:b8950bb8-94eb-428a-85f8-ce4241a67398/251205_idealo_Pressemeldung_Black-Friday-Recap-2025.pdf
7. Contentsquare (2020): Digital Experience Benchmark — average 4.48 page views per session — https://go.contentsquare.com/hubfs/eBooks/2020%20Digital%20Experience%20Benchmark/2020%20Digital%20Experience%20Benchmark%20Report%20PDF%20English.pdf
8. HTTP Archive Web Almanac 2025: Page Weight (inner page 1.8MB mobile, 2MB desktop; 71 requests/inner page) — https://almanac.httparchive.org/en/2025/page-weight
9. Robust Perception: Peak-to-mean ratio (понятие коэффициента peak/mean) — https://www.robustperception.io/do-you-know-your-peak-to-mean-ratio
10. Think with Google (2017): Mobile Moments idealo (58% customer journey starts on mobile) — https://www.thinkwithgoogle.com/_qs/documents/4260/40943_170315_MobileMoments_Idealo_TwoPage_Kec6FX5.pdf
11. AWS Route 53: Latency-based routing — https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-latency.html
12. AWS Route 53: Failover records — https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values-failover.html
13. Cloudflare Learning Center: Anycast network — https://www.cloudflare.com/learning/cdn/glossary/anycast-network/
14. AWS Global Accelerator docs: anycast static IPs — https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-components.html


