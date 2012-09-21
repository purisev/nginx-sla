# nginx-sla

Модуль [nginx](http://nginx.org/ru/), реализующий сбор расширенной статистики по HTTP-кодам и временам ответов апстримов для дальнейшей передачи в средства мониторинга типа [zabbix](http://www.zabbix.com/).

## Сборка

Для сборки необходимо сконфигурировать nginx с дополнительным параметром:

```
./configure --add-module=/path/to/nginx-sla
```

## Конфигурация

```
синтаксис: sla_pool название [timings=время:время:...:время] [http=статус:статус:...:статус] [avg_window=число] [default];
умолчание: timings=300:500:2000, http=200:301:302:304:400:401:403:404:499:500:502:503:504, avg_window=1600
контекст:  http
```

Задает именованный пул для сбора статистики. Должен быть задан хотя бы один пул.

* `название` - имя пула, использующееся в выводе статистики и директиве `sla_pass`;
* `timings` - интервалы времени в ms, в которые отсчитывается "попадание" времени запроса;
* `http` - отслеживаемые статусы http;
* `avg_window` - размер окна для вычисления скользящего среднего времени ответа;
* `default` - задает пул по умолчанию - в этот пул попадают все запросы, для которых не указан явно другой пул директивой `sla_pass`.

Размер окна для вычисления скользящего среднего времени ответа рекомендуется выбирать исходя из среднего количества динамических запросов в секунду помноженное на интервал времени сбора данных.

```
синтаксис: sla_pass название
умолчание: -
контекст:  http, server, location
```

Указывает имя пула, в который требуется собирать статистику. В случае значения `off` отключает сбор статистики (в т.ч. и в пул по умолчанию).

```
синтаксис: sla_status
умолчание: -
контекст:  server, location
```

Обработчик вывода статистики.

```
синтаксис: sla_purge
умолчание: -
контекст:  server, location
```

Обработчик обнуления счетчиков статистики.

## Пример конфигурации

```
spa_pool main timings=100:200:300:500:1000:2000 http=200:404 default;

server {
    listen [::1]:80;
    location / {
        sla_pass main;
        ...
    }
    location /sla_status {
        sla_status;
        sla_pass off;
        allow ::1;
        deny all;
    }
    location /sla_purge {
        sla_purge;
        sla_pass off;
        allow ::1;
        deny all;
    }
}
```

## Ограничения

Ограничения могут быть изменены на этапе компиляции указанием соответствующих директив препроцессора:

* `NGX_HTTP_SLA_MAX_NAME_LEN` - максимальная длина имени апстрима (по умолчанию 256 байт);
* `NGX_HTTP_SLA_MAX_HTTP_LEN` - максимальное количество отслеживаемых статусов HTTP (по умолчанию 32);
* `NGX_HTTP_SLA_MAX_TIMINGS_LEN` - максимальное количество отслеживаемых таймингов (по умолчанию 32);
* `NGX_HTTP_SLA_MAX_QUANTILES_LEN` - максимальное число отслеживаемых квантилей (по умолчанию 16);
* `NGX_HTTP_SLA_MAX_COUNTERS_LEN` - максимальное количество счетчиков (апстримов) в пуле (по умолчанию 16).

## Содержимое статистики

```
main.all.http = 1024
main.all.http_200 = 914
...
main.all.http_xxx = 2048
main.all.http_2xx = 914
...
main.all.time.avg = 125
main.all.time.avg.mov = 124
main.all.300 = 233
main.all.300.agg = 233
main.all.500 = 33
main.all.500.agg = 266
main.all.2000 = 40
main.all.2000.agg = 306
main.all.inf = 0
main.all.inf.agg = 306
main.all.25% = 124
main.all.75% = 126
main.all.90% = 126
main.all.99% = 130
...
main.<upstream>.http_200 = 270
...
```

* Здесь первое значение является именем пула статистики;
* Второе значение является именем апстрима или `all` для всех апстримов пула (включая локальную статику);
* Третье значение - имя ключа статистики:
  * `http` - количество обработанных ответов с известными HTTP статусами;
  * `http_200` - количество ответов с HTTP-статусом 200 (может быть `http_301`, `http_404`, `http_500` и т.д.);
  * `http_xxx` - количество обработанных ответов в группах статусов HTTP (фактически, количество всех обработанных ответов);
  * `http_2xx` - количество ответов в группе с HTTP-статусом 2xx (всего 5 групп соответствующих `1xx`, `2xx` ... `5xx`);
  * `time` - характеристика времени ответов;
  * `500` - количество ответов апстримов в интервале времени 300-500 ms;
  * `90%` - время ответа в ms для 90% запросов (процентиль, может принимать значения `25%`, `50%`, `75%`, `90%`, `95%`, `98%`, `99%`);
  * `inf` - алиас для "бесконечного" интервала времени;
* Четвертое и пятое значение - тип статистики:
  * `avg` - среднее;
  * `mov` - скользящее (среднее);
  * `agg` - аггрегированная статистика по всем интервалам до текущего. Так, например, в 500.agg попадают все запросы, которые выполнились от 0 и до 500 ms;

## Используемые алгоритмы

Для вычисления процентилей используется алгоритм EWSA ("Exponentially Weighted Stochastic Approximation") - подробности см. "[Incremental Quantile Estimation for Massive Tracking](ftp://ftp10.us.freebsd.org/users/azhang/disc/disc01/cd1/out/papers/kdd/p516-chen.pdf)", Fei Chen, Diane Lambert, и Jose C. Pinheiro (2000).

Параметры алгоритма могут быть изменены на этапе компиляции указанием соответствующих директив препроцессора:

* `NGX_HTTP_SLA_QUANTILE_M` - размер буфера FIFO для обновления данных (по умолчанию 100);
* `NGX_HTTP_SLA_QUANTILE_W` - весовой коэффициент обновления вычисляемых квантилей (по умолчанию 0.05).

Перед изменением данных параметров имеет смысл внимательно ознакомиться с описанием алгоритма.
