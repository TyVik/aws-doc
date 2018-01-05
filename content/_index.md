---
title: "Index"
weight: 1
type: "page"
creatordisplayname: "TyVik"
creatoremail: "tyvik8@gmail.com"
lastmodifierdisplayname: "TyVik"
lastmodifieremail: "tyvik8@gmail.com"
---

Не так давно мой проект geopuzzle.org переехал с DigitalOcean на AWS. Отчасти причиной послужили возросшие нагрузки, и дроплета за $5 стало не хватать. Однако, основная причина заключается в том, что мне хочется ~~поиграться~~ изучить серьёзный стек технологий, а лучшее всего это сделать реализовав какой-нибудь pet-project. Django, PostGIS, SPARQL, WebSockets и React уже опробованы, дело за деплоем и созданием инфраструктуры. Передо мной был выбор из 3х платформ: Microsoft Azure, Google Cloud Platform и Amazon Web Services. Все они предоставляют некоторое количество ресурсов в триальном периоде, который длится год. Так как на работе у меня используется AWS, и опыта работы с ним уже больше года, то я остановился на последнем варианте. Так что этот цикл статей будет посвящён началу работы с AWS. Если будут замечания, обязательно дополню статью.

# Плюсы и минусы AWS

Прежде чем переезжать на AWS хотелось бы понять зачем. Для этого составлю список pro и contra.
Плюсы:

* виртуалка с 1 Gb RAM, отдельная база данных, хранилище для файликов да ещё с CDN на год - БЕСПЛАТНО!

* теперь не надо париться с настройкой postgres или redis

* AWS CLI - можете разворачивать сервера прямо из консоли

* платите только за то, что используете

Минусы:

* вы платите за всё, что используете

* после тестового периода обслуживание становится дорогим

* жуткий vendor-lock

* железо иногда умирает, и техподдержка может только посоветовать восстановиться из бэкапа

# Особенности

AWS предназначен в первую очередь для среднего и крупного бизнеса. Я же хочу показать как он может быть полезен отдельному разработчику. Во-первых, цены... Они взлетают просто до небес. Если виртуалки стоят примерно столько же, сколько и у DO, то вот за самую слабую БД надо отдать уже $13/mo. Прикинуть свой бюджет можно вот [здесь](http://calculator.s3.amazonaws.com/index.html). Я хочу уложиться вот в [эту сумму](http://calculator.s3.amazonaws.com/index.html#s=RDS&key=calc-FreeTier-NGC-140321). НО! Это если не заботиться об издержках. AWS предоставляет большую скидку при аренде серверов на длительный срок, аж до 50%. Например, арендовать машину аналогичную $10/mo от DO, вы можете всего за $69/1ye или $124/3ye. Но там тоже есть свои нюансы, о которых я расскажу более подробно в дальнейших статьях. Пока достаточно помнить, что на аренде ресурсов на длительный срок можно нехило сэкономить.

Во-вторых, вы будете платить за все ресурсы. Нужено место на виртуалке - плати, заливаешь в БД из дома - плати за трафик, смотришь список файлов в каталоге S3 - тоже плати. К слову, обычно есть бесплатная квота, которой в принципе хватает... но вот заливка 100 Gb фотоархива на S3 может вылиться в [$11.36](https://aws.amazon.com/ru/s3/pricing/) ($8.91 передача данных + $0.15 за 30k запросов на сохранение + $2.3/mo за хранение). Так что имейте ввиду, что трафик и запросы здесь не всегда бесплатны.

В-третьих, если вы намереваетесь использовать что-то кроме виртуалок, БД, кешей и хранилища, то рискуете навсегда остаться на AWS. Совсем не обязательно, что аналоги сервисов будут у Google или Microsoft. Так что будьте внимательны! Тем не менее, если вы укладываетесь в рамки [бесплатного годового обслуживания](https://aws.amazon.com/ru/free/), то ничто не мешает после первого года завести новый аккаунт и перекочевать туда :) Да, есть нюансы, но это вполне рабочая схема.

В-четвёртых, железо дохнет. Пару раз до машины было просто не достучаться. Техподдержка предложила убить виртуалку и восстановить из бэкапа и очень долго удивлялась как это у нас нет бэкапа :) Шучу, бэкапы делайте всегда. Благо что для некоторых сервисов они бесплатны и делаются автоматически.

В общем, если знать особенности платформы, то содержание проекта может быть даже дешевле чем на DigitalOcean, и уж точно дешевле чем у Heroku. О ценовой политике обязательно будет отдельная статья.

# Так оно мне нужно или нет?

Пожалуй, самый главный вопрос :) Итак, мои рекомендации:

1. вы программист и хотите изучить облачную архитектуру - однозначно стоит! Такое ощущение, что 2/3 компаний на западе сидят на AWS, а остальные на Google Cloud.

2. вы небольшой стартап - стоит, но не увлекайтесь. Считайте, что первый год вы будете хоститься на халяву, но после - либо выкупать аренду хотя бы на год вперёд, либо переезжать на новый аккаунт, либо съезжать совсем. Так что не попадитесь на vendor-lock.

3. вы небольшая компания - уже под вопросом. Выкупайте несколько виртуалок на 3 года вперёд и разворачивайте окружение сами. Вряд ли выам нужен будет отдельный redis за $12/mo, а вот своя виртуалка с ним же обойдётся всего в $3.

4. вы веб-студия - аналогично предыдущему пункту. Если есть пул клиентов, то всех их можно разместить у себя, сделав shared хостинг. Возрастает нагрузка - переносите клиента на отдельную машину и берёте больше денег.
На этом пока всё

Подумайте, прикиньте, переварите и переходите к следующей статье - "Регистрация на AWS".