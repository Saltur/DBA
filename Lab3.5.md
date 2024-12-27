# *DBA3*
# *Лабораторная работа №3*
## *Переключение на реплику*
### *Настройка потоковой репликации*

Настроим реплику так же, как делали в предыдущей теме, а потом перейдем на нее.

Создаем автономную резервную копию, попросив утилиту создать слот и необходимые файлы (postgresql.auto.conf с настройками и standby.signal).
```
student$ pg_basebackup --pgdata=/home/student/backup -R --slot=replica --create-slot
```
Выкладываем копию в каталог PGDATA сервера beta:
```
student$ sudo pg_ctlcluster 13 beta status

Error: /var/lib/postgresql/13/beta is not accessible or does not exist

student$ sudo rm -rf /var/lib/postgresql/13/beta

student$ sudo mv /home/student/backup /var/lib/postgresql/13/beta

student$ sudo chown -R postgres:postgres /var/lib/postgresql/13/beta
```
Запускаем реплику:
```
student$ sudo pg_ctlcluster 13 beta start
```
Проверим настроенную репликацию. Выполним несколько команд на мастере:
```
α=> CREATE DATABASE replica_switchover;

CREATE DATABASE

α=> \c replica_switchover

You are now connected to database "replica_switchover" as user "student".

α=> CREATE TABLE test(s text);

CREATE TABLE

α=> INSERT INTO test VALUES ('Привет, мир!');

INSERT 0 1
```
Проверим реплику:
```
student$ psql -p 5433

β=> \c replica_switchover

You are now connected to database "replica_switchover" as user "student".

β=> SELECT * FROM test;

      s       
--------------
 Привет, мир!
(1 row)
```
### *Переход на реплику*

Сейчас сервер beta является репликой (находится в режиме восстановления):
```
β=> SELECT pg_is_in_recovery();

 pg_is_in_recovery 
-------------------
 t
(1 row)
```
Повышаем реплику. В версии 13 появилась функция pg_promote(), которая выполняет то же действие.
```
student$ sudo pg_ctlcluster 13 beta promote
```
Теперь бывшая реплика стала полноценным экземпляром.
```
β=> SELECT pg_is_in_recovery();

 pg_is_in_recovery 
-------------------
 f
(1 row)
```
Мы можем изменять данные:
```
β=> INSERT INTO test VALUES ('Я - бывшая реплика (новый мастер).');

INSERT 0 1
```
### *Утилита pg_rewind*

Между тем сервер alpha еще не выключен и тоже может изменять данные:
```
α=> INSERT INTO test VALUES ('Die hard');

INSERT 0 1
```
В реальности такой ситуации необходимо всячески избегать, поскольку теперь непонятно, какому серверу верить. Придется либо полностью потерять изменения на одном из серверов, либо придумывать, как объединить данные.

Наш выбор — потерять изменения, сделанные на первом сервере.

Мы планируем использовать утилиту pg_rewind, поэтому убедимся, что включены контрольные суммы на страницах данных:
```
α=> SHOW data_checksums;

 data_checksums 
----------------
 on
(1 row)
```
Этот параметр служит только для информации; изменить его нельзя — подсчет контрольных сумм задается при инициализации кластера или утилитой pg_checksums на остановленном сервере.

Остановим целевой сервер (alpha) некорректно.
```
student$ sudo head -n 1 /var/lib/postgresql/13/alpha/postmaster.pid

12379

student$ sudo kill -9 12379
```
Создадим на сервере-источнике (beta) слот для будущей реплики:
```
β=> SELECT pg_create_physical_replication_slot('replica');

 pg_create_physical_replication_slot 
-------------------------------------
 (replica,)
(1 row)
```
И проверим, что параметр full_page_writes включен:
```
β=> SHOW full_page_writes;

 full_page_writes 
------------------
 on
(1 row)
```
Если целевой сервер не был остановлен корректно, утилита сначала запустит его в монопольном режиме и остановит с выполнением контрольной точки. Для запуска требуется наличие файла postgresql.conf в PGDATA.
```
postgres$ touch /var/lib/postgresql/13/alpha/postgresql.conf
```
В ключах утилиты pg_rewind надо указать каталог PGDATA целевого сервера и способ обращения к серверу-источнику: либо подключение от имени суперпользователя (если сервер работает), либо местоположение его каталога PGDATA (если он выключен).
```
postgres$ /usr/lib/postgresql/13/bin/pg_rewind -D /var/lib/postgresql/13/alpha --source-server='user=postgres port=5433' -R -P

pg_rewind: connected to server
pg_rewind: executing "/usr/lib/postgresql/13/bin/postgres" for target server to complete crash recovery
2024-01-16 09:18:26.098 GMT [13293] LOG:  database system was interrupted; last known up at 2024-01-16 09:18:18 GMT
2024-01-16 09:18:26.099 GMT [13293] LOG:  database system was not properly shut down; automatic recovery in progress
2024-01-16 09:18:26.099 GMT [13293] LOG:  redo starts at 0/5000ED0
2024-01-16 09:18:26.100 GMT [13293] LOG:  invalid record length at 0/501BF28: wanted 24, got 0
2024-01-16 09:18:26.100 GMT [13293] LOG:  redo done at 0/501BF00

PostgreSQL stand-alone backend 13.7 (Ubuntu 13.7-1.pgdg22.04+1)
backend> pg_rewind: servers diverged at WAL location 0/501BEC0 on timeline 1
pg_rewind: rewinding from last common checkpoint at 0/5000F08 on timeline 1
pg_rewind: reading source file list
pg_rewind: reading target file list
pg_rewind: reading WAL in target
pg_rewind: need to copy 54 MB (total source directory size is 86 MB)
    0/55371 kB (0%) copied
55371/55371 kB (100%) copied
pg_rewind: creating backup label and updating control file
pg_rewind: syncing target data directory
pg_rewind: Done!
```
В результате работы pg_rewind «откатывает» файлы данных на ближайшую контрольную точку до того момента, как пути серверов разошлись, а также создает файл backup_label, который обеспечивает применение нужных журналов для завершения восстановления.

Заглянем в backup_label:
```
student$ sudo cat /var/lib/postgresql/13/alpha/backup_label

START WAL LOCATION: 0/5000ED0 (file 000000010000000000000005)
CHECKPOINT LOCATION: 0/5000F08
BACKUP METHOD: pg_rewind
BACKUP FROM: standby
START TIME: 2024-01-16 12:18:26 MSK
```
Ключом -R мы попросили утилиту создать сигнальный файл standby.signal и задать в конфигурационном файле строку соединения.
```
student$ sudo ls -l /var/lib/postgresql/13/alpha/standby.signal

-rw------- 1 postgres postgres 0 янв 16 12:18 /var/lib/postgresql/13/alpha/standby.signal

student$ sudo cat /var/lib/postgresql/13/alpha/postgresql.auto.conf

# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
primary_conninfo = 'user=student passfile=''/home/student/.pgpass'' channel_binding=prefer host=''/var/run/postgresql'' port=5432 sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
primary_slot_name = 'replica'
primary_conninfo = 'user=postgres passfile=''/var/lib/postgresql/.pgpass'' channel_binding=prefer port=5433 sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
```
Утилита добавляет строку для primary_conninfo в конец существующего файла конфигурации, поэтому остальные настройки (primary_slot_name) продолжат действовать.

Можно стартовать новую реплику.
```
student$ sudo pg_ctlcluster 13 alpha start
```
Слот репликации инициализировался и используется:
```
β=> SELECT * FROM pg_replication_slots \gx

-[ RECORD 1 ]-------+----------
slot_name           | replica
plugin              | 
slot_type           | physical
datoid              | 
database            | 
temporary           | f
active              | t
active_pid          | 13484
xmin                | 
catalog_xmin        | 
restart_lsn         | 0/5037998
confirmed_flush_lsn | 
wal_status          | reserved
safe_wal_size       | 
```
Данные, измененные на новом мастере, получены:
```
student$ psql -p 5432 -d replica_switchover

α=> SELECT * FROM test;

                 s                  
------------------------------------
 Привет, мир!
 Я - бывшая реплика (новый мастер).
(2 rows)
```
Проверим еще:
```
β=> INSERT INTO test VALUES ('Еще строка с нового мастера.');

INSERT 0 1

α=> SELECT * FROM test;

                 s                  
------------------------------------
 Привет, мир!
 Я - бывшая реплика (новый мастер).
 Еще строка с нового мастера.
(3 rows)
```
Таким образом, два сервера поменялись ролями. 
### *Проблемы с файловым архивом*

Сейчас beta — основной сервер, а alpha — реплика. Настроим на обоих файловую архивацию в общий архив.
```
student$ psql -p 5433 -d replica_switchover

student$ sudo mkdir /var/lib/postgresql/archive

student$ sudo chown postgres:postgres /var/lib/postgresql/archive

β=> \c - postgres

You are now connected to database "replica_switchover" as user "postgres".

β=> ALTER SYSTEM SET archive_mode = on;

ALTER SYSTEM

β=> ALTER SYSTEM SET archive_command = 'test ! -f /var/lib/postgresql/archive/%f && cp %p /var/lib/postgresql/archive/%f';

ALTER SYSTEM

α=> \c - postgres

You are now connected to database "replica_switchover" as user "postgres".

α=> ALTER SYSTEM SET archive_mode = on;

ALTER SYSTEM

α=> ALTER SYSTEM SET archive_command = 'test ! -f /var/lib/postgresql/archive/%f && cp %p /var/lib/postgresql/archive/%f';

ALTER SYSTEM

β=> \q

α=> \q
```
Перезапускаем оба сервера.
```
student$ sudo pg_ctlcluster 13 beta restart

student$ sudo pg_ctlcluster 13 alpha restart
```
Текущий сегмент журнала:
```
student$ psql -p 5433 -d replica_switchover -U postgres

β=> SELECT pg_walfile_name(pg_current_wal_lsn());

     pg_walfile_name      
--------------------------
 000000020000000000000005
(1 row)
```
Принудительно переключим сегмент WAL, вызвав функцию pg_switch_wal. Чтобы переключение произошло, нужно гарантировать, что текущий и следующий сегменты содержат какие-либо записи.
```
β=> SELECT pg_switch_wal();
INSERT INTO test SELECT now();

 pg_switch_wal 
---------------
 0/503A080
(1 row)

INSERT 0 1
```
Теперь записывается следующий сегмент, а предыдущий попал в архив:
```
β=> SELECT
	pg_walfile_name(pg_current_wal_lsn()) current_wal,
	last_archived_wal,
	last_failed_wal
FROM pg_stat_archiver;

       current_wal        |    last_archived_wal     | last_failed_wal 
--------------------------+--------------------------+-----------------
 000000020000000000000006 | 000000020000000000000005 | 
(1 row)
```
Теперь представим, что возникли трудности с архивацией. Причиной может быть, например, заполнение диска или проблемы с сетевым соединением, а мы смоделируем их, возвращая статус 1 из команды архивации.
```
β=> ALTER SYSTEM SET archive_command = 'exit 1';
SELECT pg_reload_conf();

ALTER SYSTEM
 pg_reload_conf 
----------------
 t
(1 row)
```
Опять переключим сегмент WAL.
```
β=> SELECT pg_switch_wal();
INSERT INTO test SELECT now();

 pg_switch_wal 
---------------
 0/60001D8
(1 row)

INSERT 0 1
```
Сегмент не архивируется.
```
β=> SELECT
	pg_walfile_name(pg_current_wal_lsn()) current_wal,
	last_archived_wal,
	last_failed_wal
FROM pg_stat_archiver;

       current_wal        |    last_archived_wal     |     last_failed_wal      
--------------------------+--------------------------+--------------------------
 000000020000000000000007 | 000000020000000000000005 | 000000020000000000000006
(1 row)
```
Процесс archiver будет продолжать попытки, но безуспешно.
```
student$ tail -n 4 /var/log/postgresql/postgresql-13-beta.log

2024-01-16 12:18:56.503 MSK [14045] LOG:  archive command failed with exit code 1
2024-01-16 12:18:56.503 MSK [14045] DETAIL:  The failed archive command was: exit 1
2024-01-16 12:18:57.508 MSK [14045] LOG:  archive command failed with exit code 1
2024-01-16 12:18:57.508 MSK [14045] DETAIL:  The failed archive command was: exit 1
```
Alpha в режиме реплики не выполняла архивацию, а после перехода не будет архивировать пропущенный сегмент.

Остановим сервер beta, переключаемся на alpha.
```
β=> \q

student$ sudo head -n 1 /var/lib/postgresql/13/beta/postmaster.pid

14039

student$ sudo kill -9 14039

student$ sudo pg_ctlcluster 13 alpha promote
```
Еще раз принудительно переключим сегмент, теперь уже на alpha.
```
student$ psql -U postgres -d replica_switchover

α=> SELECT pg_switch_wal();
INSERT INTO test SELECT now();

 pg_switch_wal 
---------------
 0/7002120
(1 row)

INSERT 0 1
```
Что с архивом?
```
student$ ls -l /var/lib/postgresql/archive

total 49156
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000020000000000000005
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000020000000000000007.partial
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000030000000000000007
-rw------- 1 postgres postgres       83 янв 16 12:18 00000003.history
```
Сегмент 000000020000000000000006 отсутствует, архив теперь непригоден для восстановления и репликации. 
### *Архивация с реплики*

Чтобы при переключении на реплику архив не пострадал, на реплике нужно использовать значение archive_mode = always. При этом команда архивации должна корректно обрабатывать одновременную запись сегмента мастером и репликой.

Восстановим архивацию на сервере alpha. Файл будет копироваться только при отсутствии в архиве, а наличие файла в архиве не будет считаться ошибкой.
```
α=> ALTER SYSTEM SET archive_command = 'test -f /var/lib/postgresql/archive/%f || cp %p /var/lib/postgresql/archive/%f';
SELECT pg_reload_conf();

ALTER SYSTEM
 pg_reload_conf 
----------------
 t
(1 row)
```
Добавим слот для реплики.
```
α=> SELECT pg_create_physical_replication_slot('replica');

 pg_create_physical_replication_slot 
-------------------------------------
 (replica,)
(1 row)
```
Теперь настроим beta как реплику с архивацией в режиме always.
```
student$ cat << EOF | sudo -u postgres tee /var/lib/postgresql/13/beta/postgresql.auto.conf
primary_conninfo='user=student port=5432'
primary_slot_name='replica'
archive_mode='always'
archive_command='test -f /var/lib/postgresql/archive/%f || cp %p /var/lib/postgresql/archive/%f'
EOF

primary_conninfo='user=student port=5432'
primary_slot_name='replica'
archive_mode='always'
archive_command='test -f /var/lib/postgresql/archive/%f || cp %p /var/lib/postgresql/archive/%f'
```
Стартуем реплику.
```
postgres$ touch /var/lib/postgresql/13/beta/standby.signal

student$ sudo pg_ctlcluster 13 beta start

student$ psql -U postgres -d replica_switchover
```
Повторим опыт. Переключаем сегмент:
```
α=> SELECT pg_switch_wal();
INSERT INTO test SELECT now();

 pg_switch_wal 
---------------
 0/8000338
(1 row)

INSERT 0 1
```
Проверяем состояние архивации:
```
α=> SELECT
	pg_walfile_name(pg_current_wal_lsn()) current_wal,
	last_archived_wal,
	last_failed_wal
FROM pg_stat_archiver;

       current_wal        |    last_archived_wal     | last_failed_wal 
--------------------------+--------------------------+-----------------
 000000030000000000000009 | 000000030000000000000008 | 
(1 row)
```
Заполненный сегмент попал в архив.

На сервере alpha возникли проблемы с архивацией, команда возвращает 1:
```
α=> ALTER SYSTEM SET archive_command = 'exit 1';
SELECT pg_reload_conf();

ALTER SYSTEM
 pg_reload_conf 
----------------
 t
(1 row)
```
Alpha продолжает генерировать сегменты WAL.
```
α=> SELECT pg_switch_wal();
INSERT INTO test SELECT now();

 pg_switch_wal 
---------------
 0/90000C0
(1 row)

INSERT 0 1
```
Но основной сервер их не архивирует:
```
α=> SELECT
	pg_walfile_name(pg_current_wal_lsn()) current_wal,
	last_archived_wal,
	last_failed_wal
FROM pg_stat_archiver;

       current_wal        |    last_archived_wal     |     last_failed_wal      
--------------------------+--------------------------+--------------------------
 00000003000000000000000A | 000000030000000000000008 | 000000030000000000000009
(1 row)
```
Однако архивация с реплики срабатывает и сегмент оказывается в архиве:
```
student$ ls -l /var/lib/postgresql/archive

total 131076
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000010000000000000005
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000020000000000000005
-rw------- 1 postgres postgres 16777216 янв 16 12:19 000000020000000000000006
-rw------- 1 postgres postgres 16777216 янв 16 12:19 000000020000000000000007
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000020000000000000007.partial
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000030000000000000007
-rw------- 1 postgres postgres 16777216 янв 16 12:19 000000030000000000000008
-rw------- 1 postgres postgres 16777216 янв 16 12:19 000000030000000000000009
-rw------- 1 postgres postgres       83 янв 16 12:18 00000003.history
```
Выполняем переключение на реплику.
```
α=> \q

student$ sudo head -n 1 /var/lib/postgresql/13/alpha/postmaster.pid

14086

student$ sudo kill -9 14086

student$ sudo pg_ctlcluster 13 beta promote
```
Beta стала основным сервером и генерирует файлы WAL.
```
student$ psql -p 5433 -U postgres -d replica_switchover

β=> SELECT pg_switch_wal();
INSERT INTO test SELECT now();

 pg_switch_wal 
---------------
 0/A0000F0
(1 row)

INSERT 0 1
```
Еще раз заглянем в архив:
```
student$ ls -l /var/lib/postgresql/archive

total 163848
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000010000000000000005
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000020000000000000005
-rw------- 1 postgres postgres 16777216 янв 16 12:19 000000020000000000000006
-rw------- 1 postgres postgres 16777216 янв 16 12:19 000000020000000000000007
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000020000000000000007.partial
-rw------- 1 postgres postgres 16777216 янв 16 12:18 000000030000000000000007
-rw------- 1 postgres postgres 16777216 янв 16 12:19 000000030000000000000008
-rw------- 1 postgres postgres 16777216 янв 16 12:19 000000030000000000000009
-rw------- 1 postgres postgres 16777216 янв 16 12:19 00000003000000000000000A.partial
-rw------- 1 postgres postgres       83 янв 16 12:18 00000003.history
-rw------- 1 postgres postgres 16777216 янв 16 12:19 00000004000000000000000A
-rw------- 1 postgres postgres      125 янв 16 12:19 00000004.history
```
В архиве появились файлы реплики, пропусков в нем нет, проблема решена. 