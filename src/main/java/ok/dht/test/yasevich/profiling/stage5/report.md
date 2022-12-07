* `1 Мб` flushThreshold
* Размер очереди `100`
* Размер кластера `3`
* Таймаут проксированного запроса `100ms`

Прогрев сервер, запускаем профилирование put запросов, потом сразу же запускаем профилирование get.
Профилировщик запускаем в трёх режимах: cpu, alloc и lock

`./profiler.sh -f put.html -e cpu,alloc,lock --chunktime 1s start Server`

wrk2 запускаем в

* 64 соединений
* 8 потоков
* 1 минута

## Тест 1

`ack/from` = `2/3` (по умолчанию для кластера из трёх нод)
Найдём rate, с которым справляется наш сервис.

Для put это `27К` ([wrk-put-report](wrk/put-report-1_2)):

* В таком случаем мы имеем:
    * средняя задержка `31.66ms`
    * `90%` перцентиль `70.21ms`
    * с `97.5%` задержка переваливает за `100ms`
    * максимальная не превышает `200ms`
* Начиная с rate 27.5к сервис [перестаёт справляться](wrk/get-report-1_1)
    * средняя задержка `512.96ms`
    * Задержка переваливает за `1s` примерно с перцентиля `93%`.

Для get это `25.8К`:

* 27000 rate - [не ок](wrk/get-report-1_1)
* 26500 get - [лучше](wrk/get-report-1_2), но не так хорошо как для put:
    * переваливаем за `200ms` с перцентиля `70%`
    * с `99.7%` за `500ms`
* `25800` rate - [ок](wrk/get-report-1_3), значения задержек почти как для put (чуть больше), кроме:
    * на `99.5%` достигаем `200ms`, что максимум для put
    * на `99.9%` `437.25ms` (`176.25ms` для put)

Поправив асинхронное взаимодействие, получилось, что цифры в режиме `2/3` получились заметно лучше, чем в stage4 (`9к`
для get и put)

В сравнении с ранними stage, put и get почти сравнялись (`27500` и `25800`), хоть и сохраняется чуть большая
производительность на put. В stage3 это было `60к` и `50k`.

[Результаты CPU](html/cpu1.html) профилирования:

* ~`24%` работа селекторов one.nio
* ~`6%` работа селекторов httpClient
* `1.4%` работа сборщика мусора
* `13%` работа FJP, из которых `8.94%` это парки, а оставшиеся это работа с dao (CF.asyncRun) и отправка ответа в сокет
* `53.16%` работа воркеров:
    * `11.33` take из очереди
    * `7%` take из LinkedBlockingQueue (очередь FixedThreadPool`a)
    * `3%` sendResponse
    * dao upsert `0.27%`
    * dao.get `3.50%`
    * Оставшееся - работа с CF и http клиентом

Передав в httpClient свой executor (fixedThreadPool на 16 потоков),
получилось избавиться от картины из stage4, когда 24% времени потоки пула клиента занимались парками

[Результаты alloc](html/alloc1.html) профилирования:

* `30%` one.nio селекторы
    * `5.44%` моих аллокаций
        * извлечение параметров, хедера
        * а так же, оказывается, есть [аллокации](imgs/alloc.png) на класс
          ReplicationParams (решил вынести парсинг параметров ack и from из handleRequest())
* в воркерах
    * `15.77%` dao
    * `4.74%` sendResponse
    * `30%` completableFuture и httpClient
    * `9.89%` аллокаций в методе handleReplicatingRequest(), где мы отправляем запрос с помощью httpClient и получаем CF
* В ForkJoinPool:
    * `7.06%` sendResponse
    * `4.39%` dao

[Результаты lock](html/lock1.html) профилирования:

* `77.55%` java.internal.net.http
    * в т.ч. `0.53%` и `0.59%` в handleReplicatingRequest()
* `14.51%` take() из очередей (`3.78` очередь воркеров, остальное LinkedBlockingQueue пула http клиента)
* `3.9%` в one.nio селекторах на queue.offer()

## Тест 2

Попробуем изменить размер кластера на `5`
`ack/from` = `3/5` (по умолчанию для кластера из пяти нод)

Для put на том же rate (`27к`) [справились](wrk/put-report-2), хотя задержки для последних перценитлей заметно больше:

* `296.70ms` vs `70.21ms` для `90%`
* `494.33ms` vs `176.25ms` для `99.9%`

Для get [не справились](wrk/get-report-2_1)

* Постепенно уменьшая до `24к` всё ещё [не справляемся](wrk/get-report-2_2)
* Уже [лучше](wrk/get-report-2_3) для rate `23к`

Увеличив размер кластера до 5, получилось:

* рабочий rate для put остался примерно тем же (`27к`)
* для get точка разладки сместилась и теперь рабочий rate до `23к`

## Тест 3

`ack/from` = `5/5`!

Для put:

* с rate `27к` не справились
* с `15к` немного [не справляемся](wrk/put-report-3_2)
* `14500` - [лучше](wrk/put-report-3_1)

Для get:

* `12750` - [не справились](wrk/get-report-3_2)
* `12500` - [справились](wrk/get-report-3_1)

Рабочий rate снизился примерно в два раза

### Итого:

* Поправили асинхронное взаимодействие и получили больший rate (для ack/from=2/3) в сравнении с stage4
* При этом rate меньше в сравнении со stage3, но теперь мы имеем репликацию!
* Работа с запросами, которым важен ответ от всех нод, заметно тяжелее
* На профилях много времени уходит на взаимодействия по протоколу http, для большей производительности хотелось бы
  уменьшить этот оверхед

