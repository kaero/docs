# Миграция данных в [!KEYREF mmy-name]

Чтобы перенести существующую базу данных MySQL в сервис [!KEYREF mmy-name], используйте утилиты `mysqldump` и `mysql`: создайте дамп рабочей базы и восстановите его в [!KEYREF MY]-кластере Облака.

Перед тем, как пытаться импортировать данные, проверьте, совпадают ли версии СУБД у существующей базы данных и вашего кластера в Облаке. Так как дамп логический и представляет собой набор SQL выражений, то перенос данных между кластерами с различными версиями возможен, но не гарантируется, более подробно в [MySQL 5.7 FAQ Migration](https://dev.mysql.com/doc/refman/5.7/en/faqs-migration.html).

Сложность и длительность миграции сильно зависит от размера базы. Миграция базы данных размером в десятки гигабайт может оказаться гораздо сложнее, чем миграция относительно небольшой базы в 1 гигабайт и с небольшим количеством таблиц.

Этапы миграции:
1. Создание дампа переносимой базы.
2. Создание виртуальной машины в Яндекс Облаке и загрузка данных на неё (опционально).
3. Создание Managed MySQL кластера.
4. Восстановление данных из дампа в кластере с использованием `mysql`.


## Создание дампа {#dump}

Создать дамп базы данных следует с помощью утилиты `mysqldump`, которая подробно описана в [документации [!KEYREF MY]](https://dev.mysql.com/doc/refman/5.7/en/mysqlpump.html).

1. Перед созданием дампа рекомендуется переключить базу в режим «только чтение», чтобы не потерять данные, которые бы появились во время создания дампа. Сам дамп базы данных создается следующей командой:

    ```bash
    $ mysqldump -h <адрес сервера СУБД> --user=<имя пользователя> --password --port=<порт> --set-gtid-purged=OFF --quick --signle-transaction <имя базы данных> > ~/db_dump.sql
    ```
   
   В случае использования на исходной БД таблиц InnoDB рекомендуется использовать опцию `--single-transaction` для гарантированной консистентности данных. Для таблиц MyISAM эта опция не имеет смысла, так как транзакции не поддерживаются. Также стоит обратить внимание на следующие флаги:

   * `--events` — если в вашей базе есть периодические события;
   * `--routines` — если в вашей базе есть функции и хранимые процедуры.

2. Для ускорения процесса можно запустить сброс дампа, используя несколько ядер процессора. Для этого нужно добавить флаг `--add-locks` и опцию `--default-parallelism` с числом, соответствующим количеству ядер, которое доступно СУБД:

    ```
    $ mysqldump -h <адрес сервера СУБД> --user=<имя пользователя> --password --port=<порт> --set-gtid-purged=OFF --add-locks --default-parallelism=<количество потоков>  <имя базы данных> > ~/db_dump.sql
    ```

3. Упакуйте дамп в архив:

    ```
    $ tar -cvzf db_dump.tar.gz ~/db_dump.sql
    ```


## (опционально) Создание виртуальной машины в Облаке и загрузка дампа {#create-vm}

Переносить данные на промежуточную виртуальную машину в [!KEYREF compute-name] нужно, если:

* К вашему кластеру БД нет доступа из интернета.
* Ваше оборудование или соединение с [!KEYREF MY]-кластером в Облаке недостаточно надежны.

Нужное количество оперативной памяти и ядер процессора зависит от объема переносимых данных и требуемой скорости переноса.

1. В консоли управление создайте новую виртуальную машину из образа Ubuntu 18.04. Параметры виртуальной машины должны зависеть от размера базы, которую Вы хотите перенести. Минимальной конфигурации (1 ядро, 2 ГБ RAM, 10 ГБ дискового пространства) должно хватить для переносы базы до 1 ГБ. Чем больше переносимая база, тем больше должно быть дискового пространства (как минимум в два раза больше, чем размер базы), и больше размер оперативной памяти.

    Виртуальная машина должна находиться в той же сети и зоне доступности, что хост-мастер кластера [!KEYREF MY]. Кроме того, виртуальной машине должен быть присвоен внешний IP-адрес, чтобы вы могли загрузить дамп извне Облака.

2. Установите клиент [!KEYREF MY] и дополнительные утилиты для работы с СУБД:
Для Debina/Ubuntu утилиты `mysqldump` и `mysql` поставляются в пакете [`mysql-client`](https://packages.ubuntu.com/search?keywords=mysql-client), его установка:

   ```
   $ sudo apt-get install mysql-client
   ```

3. Перенесите дамп базы данных на виртуальную машину, например, используя утилиту `scp`:

    ```
    scp ~/db_dump.tar.gz <имя пользователя ВМ>@<публичный адрес ВМ>:/tmp/db_dump.tar.gz
    ```

4. Распакуйте дамп:

    ```
    tar -xzf /tmp/db_dump.tar.gz
    ```
## Создание кластера [!KEYREF mmy-name]

Инструкцию по созданию кластера можно найти в разделе [[!TITLE]](./cluster-create.md).

## Восстановление данных {#restore}

Восстанавливать дамп базы  данных следует с помощью утилиты [mysql](https://dev.mysql.com/doc/refman/5.7/en/mysql.html). Восстановление рекомендуется проводить с флагом `--line-numbers`, чтобы получать больше информации в случае возникновения ошибок.

* Если вы восстанавливаете дамп с виртуальной машины в Яндекс.Облаке:

    ```
    $ mysql -h <адрес сервера СУБД> --user=<имя пользователя> --password --port=3306 --line-numbers <имя базы даннх>  < ~/db_dump.sql
    ```

* Если вы восстанавливаете дамп с собственного сервера, необходимо скачать сертификат и задать параметры SSL `--ssl-cal` и `--ssl-mode`. Команды для подключения к кластеру и ссылку на сертификaт можно найти в консоли управления, по кнопке **Подключиться**.
  
  Команда восстановления базы из дампа:

   ```
   $ mysql -h <адрес сервера СУБД> --user=<имя пользователя> --password --port=3306 --ssl-ca=~/.mysql/root.crt --ssl-mode=VERIFY_IDENTITY --line-numbers <имя базы даннх>  < ~/db_dump.sql
   ```
