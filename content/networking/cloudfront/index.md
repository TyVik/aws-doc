---
title: "CloudFront"
weight: 2
type: "page"
creatordisplayname: "TyVik"
creatoremail: "tyvik8@gmail.com"
lastmodifierdisplayname: "TyVik"
lastmodifieremail: "tyvik8@gmail.com"
---

Как я уже говорил ранее, раздавать статику непосредственно с S3 не самая лучшая идея, особенно в триальном режиме. Для этих целей существует CDN от Amazon под названием CloudFront. Сегодня поговорим о нём.

Вкратце, любой CDN представляет собой прокси-сервер к вашей статике, отвечая в том числе за её сжатие, кеширование, актуальность и пр. У меня статика хранится в корзине geo-puzzle: чуть-чуть js, немного css и картинок. Всего 5Мб и 50 файлов, но уже в начале месяца Amazon меня предупредил, что количество бесплатных GET запросов подходит к концу. Пришлось настраивать CDN. Жмём «Create distribution» и разбираемся чего же от нас хотят.

## Выбор источника

![Выбор источника](/networking/cloudfront/images/origin.png)
Для начала нас просят определиться с типом CDN: Web (раздача статики) или RTMP (стримминг медифайлов). Мне нужен обычный Web. А вот на втором шаге начинается веселье… Я не ожидал такого количества настроек. Но, на самом деле, ничего страшного в них нет. В Origin Domain Name выбираем корзину, которая будет служить источником, в Origin Path — каталог, который будет являться корневым для CDN (то есть путь bucket-name.s3.amazonaws.com/static/favicon.ico превратиться в random-name.cloudfront.net/favicon.ico), Restrict Bucket Access и Origin Custom Headers нужны чтобы предотвратить прямые запросы к S3. Мне это не нужно, но, как пример, в заголовках можно добавить специальный ключ, а на S3 ограничить по нему доступ. Origin ID — просто некий идентификатор. Вы можете создать на одну корзину S3 несколько дистрибуций CloudFront, направих их на разные каталоги.

## Настройки кеширования

![Настройки кеширования](/networking/cloudfront/images/cache.png)
На скриншоте я привёл на мой взгляд оптимальные настройки для кеширования статики. Мы будем редиректить пользователя с HTTP на HTTPS (Viewer Protocol Policy), кешировать (Object Caching) и попутно сжимать контент (Compress Objects Automatically).

Время жизни объектов в CDN определяется параметром Object Caching. Если вы ранее уже настроили установку заголовка Cache-Control для файлов на S3, то можете оставлять переключатель на Use Origin Cache Headers, в противном случае лучше выбрать Customize. Откроются дополнительные поля ввода для TTL, их можно оставить как есть. Вообще тема кеширования очень объёмна. Я могу посоветовать [эту](https://habrahabr.ru/post/253121/) и [эту](http://prgssr.ru/development/luchshie-praktiki-keshirovaniya.html) статью для понимая как это всё работает.

Тут же можно задать и лямбду, которая будет вызываться по событию обращения к Origin. Пока слабо себе представляю зачем, но, допустим, чтобы провалидировать стили или скрипты…

Все указанные здесь настройки опциональны. Оно заработает в любом случае, а настроить под себя вы сможете  после создания дистрибуции.

## Настройки дистрибуции

![Настройки дистрибучии](/networking/cloudfront/images/distribution.png)
А вот это очень важный раздел, описывающий входную точку CDN.

Price Class — в каких регионах размещать контент, для free tier доступны только США, Канада и Европа. В остальных регионах цена почти в 2 раза больше.

AWS WAF Web ACL (шта?!) — это правила защиты от DDoS, они задаются в сервисе WAF & Shield.

Alternate Domain Names позволяет подключать свой собственный домен в качестве CDN, то есть чтобы статика располагалась на https://static.your-domain.com. Красиво, да? Вот только сертификата на CNAME домены у меня нет, так что я остановлюсь на том, что предоставляет сам Amazon. Да-да, он для вас зарегистрирует новое имя третьего уровня и подключит сертификат. Здесь же можно настроить логгирование запросов в какую-нибудь корзину S3. Их можно будет в дальнейшем использовать для автоматизированного анализа нагрузки.

Снова жмём Create distribution и ждём порядка 10-15 минут. У меня получилась такая картина:
![Список дистрибуций](/networking/cloudfront/images/index.png)
После того, как статус перешёл в состояние deployed, хорошо бы убедиться, что всё заработало. Запросим какой-нибудь файлик с S3 через CDN. Если он лежит по пути /static/js/main.js и Origin Path был установлен в /static, то искать его надо по пути <random>.cloudfront.net/js/main.js. То есть каталог static является корнем CDN. Если не получилось, отписывайтесь в комменты 🙂 Более того, я бы ещё рекомендовал проверить заголовки кеширования: Last-Modified, Etag и Cache-Control. Обратите внимание на служебный заголовок x-cache — он говорит был ли файл взят с S3 или CloudFront. Несколько возможных значений: «RefreshHit from cloudfront», «Miss from cloudfront», «Hit from cloudfront».

## Reports & Analytics

Ранее я упоминал, что логи можно выгружать на S3. Это будет полезно для машинного анализа, нам же проще смотреть их в разделе Reports & Analytics. Кратко пробегусь по каждому разделу.

### Cache Statistics

Показывает насколько хорошо у нас закешировался контент. Как правило интересны только 2 графика:

* Percentage of Viewer Requests by Result Type — общий анализ кеша. Errors быть не должно, Hits (попадания в кеш) должно увеличиваться, Misses (промахи) — уменьшаться.

* HTTP Status Codes — красивая статистика по HTTP кодам. 5xx быть не должно, 4xx допустимы, от 3xx надо по возможности избавиться, 2xx — всё OK.

### Monitoring and Alarms

Оповещение админа а том, что с CDN творится нечто странное. Можно настроить по кол-ву запросов или по HTTP кодам. Я рекомендую настроить 2 метрики: по 4xx и по 5xx ошибкам. Первая более важна и говорит, что проблема на нашей стороне — какие-то файлы более недоступны, и скорее всего проблема с Origin. Вторая просто ставит нас в известность, что CDN отвалился. Как правило, админы Amazon уже в курсе, но лучше убедиться, что проблема зарегистрирована.

### Popular Objects

Очень важный раздел, показывающий что же у нас находится в CDN, а также насколько она эффективна. Разберём на небольшом примере:

![Статистика](/networking/cloudfront/images/objects.png)

Здесь у меня целых 2 проблемы: у js скриптов Hits равно нулю, то есть они всегда читаются с S3 и большое кол-во статусов 3xx. Насчёт первой проблемы — Cache-Control у js скриптов был выставлен в no-cache, а вот насчёт второй надо будет уже смотреть код.

### Viewers

Небольшая аналитика по пользователям. Какие используются браузеры, ОС и устройства, а также из какого региона идут запросы. Этот раздел будет интересен лишь в том случае, если у вас не подключна Google Analytics или Yandex.Metrika.

## Редактирование настроек

После того, как CDN заработал, посмотрим какие у него ещё ручки есть. Выбираем в списке дистрибуцию и кликаем на Distribution Settings.

### General

Уже знакомые настройки: месторасположения, защита от DDoS, настройка имени домена, протоколирование… Здесь же можно временно отключить работу конкретного CDN.

### Origins & Invalidations

Эти две вкладки тесно связаны между собой. Origins описывает источники, а в Invalidations можно заставить CloudFront перекешировать определённые файлы. При деплое можно, конечно, просто заменять файлики на S3 и инвалидировать кеш, но у этого подхода есть большой минус — если что-то пойдёт не так, откатиться будет непросто. Гораздо легче организовать [blue/green deployment](https://rtfm.co.ua/aws-blue-green-deployment/). Смысл заключается в том, чтобы хранить статику в каталоге с версией. То есть на S3 одновременно будут 2 каталога — текущий (green) и будущий (blue). При неудачном переключении CDN на blue источник, мы всегда сможем откатиться обратно на green.

### Behavior

На этой вкладке можно задать разное поведение для разных типов файлов. Как минимум одно, Default, уже существует и относится ко всему содержимому. Зачем нужно разное поведение? Ну, допустим, для файлов фреймворка — вряд ли вы их часто будете менять, а посему можно установить им TTL в максимум. Или, например, отключить сжатие превьюшек. Только не забудьте после настройки правильно указать приоритет.

## Заключение

Я описал некоторый минимум для начала работы с CloudFront, но забыл упомянуть о [ценах](https://aws.amazon.com/ru/cloudfront/pricing/). Один миллион запросов к файлу размером 15 Kb обойдётся вам всего в $2. Эту сумму можно существенно минимизировать, правильно настроив кеширование. 1M запросов — много это или мало? Делим на 30 дней и на 24 часа — получается ~1500 уникальных (мы ж кешируем?!) обращений. Если скриптов на странице около 30, то 500 уникальных пользователей в час!