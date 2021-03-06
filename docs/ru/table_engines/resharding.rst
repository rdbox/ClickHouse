Перешардирование
----------------

.. code-block:: sql

  ALTER TABLE t RESHARD [COPY] [PARTITION partition] TO описание кластера USING ключ шардирования

Запрос работает только для Replicated-таблиц и для Distributed-таблиц, смотрящих на Replicated-таблицы.

При выполнении, запрос сначала проверяет корректность запроса, наличие свободного места на серверах и кладёт в ZooKeeper по некоторому пути задачу, которую нужно сделать. Дальнейшее выполнение делается асинхронно.

Для того, чтобы использовать перешардирование, в конфигурационном файле каждого сервера должен быть указан путь в ZooKeeper к очереди задач:

.. code-block:: xml

  <resharding>
  	<task_queue_path>/clickhouse/task_queue</task_queue_path>
  </resharding>

При выполнении запроса ``ALTER TABLE t RESHARD``, узел в ZooKeeper создаётся, если его не было.

Описание кластера - список шардов с весами, на которые нужно перераспределить указанные данные.
Шард указывается в виде адреса таблицы в ZooKeeper. Например, ``/clickhouse/tables/01-03/hits``
Относительный вес шарда (не обязательно, по умолчанию, 1) может быть указан после ключевого слова WEIGHT.
Пример:

.. code-block:: sql

  ALTER TABLE merge.hits
  RESHARD PARTITION 201501
  TO
  	'/clickhouse/tables/01-01/hits' WEIGHT 1,
  	'/clickhouse/tables/01-02/hits' WEIGHT 2,
  	'/clickhouse/tables/01-03/hits' WEIGHT 1,
  	'/clickhouse/tables/01-04/hits' WEIGHT 1
  USING UserID

Ключ шардирования (в примере: ``UserID``) имеет такой же смысл, как для Distributed таблиц. Вы можете указать rand() в качестве ключа шардирования для случайного перераспределения данных.

При выполнении запроса, сразу проверяется:
 * совпадение структур таблиц локально и на всех указанных шардах.
 * наличие на локальном сервере свободного места в количестве, равном размеру партиции в байтах, с запасом в 10%.
 * наличие на всех репликах всех указанных шардов, кроме являющейся локальной, если такая есть, свободного места в количестве равном размеру партиции, домноженном на отношение веса шарда к суммарному весу, с запасом в 10%.

Далее, асинхронное выполнение задачи состоит из следующих шагов:
 #. Нарезка партиции на кусочки на локальном сервере.
    Для этого делается слияние всех кусков, входящих в партицию и, одновременно, разбиение их на несколько, согласно ключу шардирования.
    Результат складывается в директорию /reshard в директории с данными таблицы.
    Исходные куски никак не модифицируются и весь процесс не влияет на рабочие данные таблицы.

 #. Копирование всех кусков на удалённые серверы (на каждую реплику соответствующих шардов).

 #. Выполнение запроса ALTER TABLE t DROP PARTITION на локальном сервере, выполнение запросов ALTER TABLE t ATTACH PARTITION на всех шардах.
    Замечание: это делается неатомарно. Есть момент времени, в течение которого пользователь может увидеть отсутствие данных партиции.

    В случае указания в запросе слова COPY, исходные данные не удаляются. Это подходит для копирования данных с одного кластера на другой с одновременным изменением схемы шардирования.

 #. Удаление временных данных с локального сервера.

При наличии нескольких запросов на перешардирование, соответствующие задачи будут делаться последовательно.

Указанный выше запрос предназначен для того, чтобы перешардировать одну партицию.
Если не указать партицию в запросе, то в задачи на перешардирование будут добавлены все партиции. Пример:

.. code-block:: sql
  
  ALTER TABLE merge.hits
  RESHARD
  TO ...

В этом случае, последовательно вставляются задачи на перешардирование каждой партиции.

В случае перешардирования Distributed-таблицы, производится перешардирование каждого шарда (соответствующий запрос отправляется на каждый шард).

Вы можете перешардировать Distributed-таблицу как в саму себя, так и в другую таблицу.

Перешардирование предназначено для перераспределения "старых" данных: в случае, если во время работы, перешардируемая партиция была изменена, то перешардирование этой партиции отменяется.

На каждом сервере, перешардирование осуществляется в один поток, чтобы в процессе длительных операций перешардирования, не мешать другим задачам.

По состоянию на июнь 2016, перешардирование находится в состоянии "бета": тестировалось лишь на небольшом объёме данных - до 5 ТБ.
