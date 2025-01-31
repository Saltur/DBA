# *DBA3*
# *Лабораторная работа №3*
## *Физическая репликация*
### *Настройка потоковой репликации*

Поскольку в нашей конфигурации не будет архива журнала предзаписи, важно на всех этапах использовать слот репликации — иначе при определенной задержке мастер может успеть удалить необходимые сегменты и весь процесс придется повторять с самого начала.

Создаем слот:
```
α=> SELECT pg_create_physical_replication_slot('replica');

 pg_create_physical_replication_slot 
-------------------------------------
 (replica,)
(1 row)
```
Посмотрим на созданный слот:
```
α=> SELECT * FROM pg_replication_slots \gx

-[ RECORD 1 ]-------+---------
slot_name           | replica
plugin              | 
slot_type           | physical
datoid              | 
database            | 
temporary           | f
active              | f
active_pid          | 
xmin                | 
catalog_xmin        | 
restart_lsn         | 
confirmed_flush_lsn | 
wal_status          | 
safe_wal_size       | 
```
Вначале слот не инициализирован (restart_lsn и wal_status пустые).

Как мы помним из модуля «Резервное копирование», все необходимые настройки есть по умолчанию:
```
    wal_level = replica;
    max_wal_senders;
    разрешение на подключение в pg_hba.conf.
```
Создадим автономную резервную копию, используя созданный слот. Копию расположим в подготовленном каталоге. С ключом -R утилита создает файлы, необходимые для будущей реплики.
```
student$ pg_basebackup --pgdata=/home/student/backup -R --slot=replica
```
Снова проверим слот:
```
α=> SELECT * FROM pg_replication_slots \gx

-[ RECORD 1 ]-------+----------
slot_name           | replica
plugin              | 
slot_type           | physical
datoid              | 
database            | 
temporary           | f
active              | f
active_pid          | 
xmin                | 
catalog_xmin        | 
restart_lsn         | 0/4000000
confirmed_flush_lsn | 
wal_status          | reserved
safe_wal_size       | 
```
После выполнения резервной копии слот инициализировался, и мастер теперь хранит все файлы журнала с начала копирования (restart_lsn, wal_status).

Сравните со строкой в backup_label:
```
student$ head -n 1 /home/student/backup/backup_label

START WAL LOCATION: 0/4000028 (file 000000010000000000000004)
```
Файл postgresql.auto.conf был подготовлен утилитой pg_basebackup, поскольку мы указали ключ -R. Он содержит информацию для подключения к мастеру (primary_conninfo) и имя слота репликации (primary_slot_name):
```
student$ cat /home/student/backup/postgresql.auto.conf

# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
primary_conninfo = 'user=student passfile=''/home/student/.pgpass'' channel_binding=prefer host=''/var/run/postgresql'' port=5432 sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
primary_slot_name = 'replica'
```
По умолчанию реплика будет «горячей», то есть сможет выполнять запросы во время восстановления. Если такая возможность не нужна, реплику можно сделать «теплой» (hot_standby = off).

Утилита также создала сигнальный файл standby.signal, наличие которого указывает серверу войти в режим постоянного восстановления.
```
student$ ls -l /home/student/backup/standby.signal

-rw------- 1 student student 0 янв 16 12:17 /home/student/backup/standby.signal
```
Выкладываем резервную копию в каталог данных сервера beta.
```
student$ sudo pg_ctlcluster 13 beta status

Error: /var/lib/postgresql/13/beta is not accessible or does not exist

student$ sudo rm -rf /var/lib/postgresql/13/beta

student$ sudo mv /home/student/backup /var/lib/postgresql/13/beta

student$ sudo chown -R postgres:postgres /var/lib/postgresql/13/beta
```
Журнальные записи, необходимые для восстановления согласованности, реплика получит от мастера по протоколу репликации. Далее она войдет в режим непрерывного восстановления и продолжит получать и проигрывать поток записей.
```
student$ sudo pg_ctlcluster 13 beta start
```
### *Процессы реплики*

Посмотрим на процессы реплики.
```
student$ ps -o pid,command --ppid `sudo head -n 1 /var/lib/postgresql/13/beta/postmaster.pid`

    PID COMMAND
  10435 postgres: 13/beta: startup recovering 000000010000000000000005
  10436 postgres: 13/beta: checkpointer 
  10437 postgres: 13/beta: background writer 
  10438 postgres: 13/beta: stats collector 
  10439 postgres: 13/beta: walreceiver streaming 0/5000060
```
Процесс wal receiver принимает поток журнальных записей, процесс startup применяет изменения.

Процессы wal writer и autovacuum launcher отсутствуют.

И сравним с процессами мастера.
```
student$ ps -o pid,command --ppid `sudo head -n 1 /var/lib/postgresql/13/alpha/postmaster.pid`

    PID COMMAND
   9985 postgres: 13/alpha: checkpointer 
   9986 postgres: 13/alpha: background writer 
   9987 postgres: 13/alpha: walwriter 
   9988 postgres: 13/alpha: autovacuum launcher 
   9989 postgres: 13/alpha: stats collector 
   9990 postgres: 13/alpha: logical replication launcher 
  10028 postgres: 13/alpha: student student [local] idle
  10440 postgres: 13/alpha: walsender student [local] streaming 0/5000060
```
Здесь добавился процесс wal sender, обслуживающий подключение по протоколу репликации. 
### *Использование реплики*

Выполним несколько команд на мастере:
```
α=> CREATE DATABASE replica_physical;

CREATE DATABASE

α=> \c replica_physical

You are now connected to database "replica_physical" as user "student".

α=> CREATE TABLE test(s text);

CREATE TABLE

α=> INSERT INTO test VALUES ('Привет, мир!');

INSERT 0 1
```
Проверим реплику:
```
student$ psql -p 5433

β=> \c replica_physical

You are now connected to database "replica_physical" as user "student".

β=> SELECT * FROM test;

      s       
--------------
 Привет, мир!
(1 row)
```
При этом изменения на реплике не допускаются:
```
β=> INSERT INTO test VALUES ('Replica');

ERROR:  cannot execute INSERT in a read-only transaction
```
Вообще реплику от мастера можно отличить с помощью функции:
```
β=> SELECT pg_is_in_recovery();

 pg_is_in_recovery 
-------------------
 t
(1 row)
```
### *Мониторинг репликации*

Состояние репликации можно смотреть в специальном представлении на мастере. Чтобы пользователь получил доступ к этой информации, ему должна быть выдана роль pg_read_all_stats (или он должен быть суперпользователем).
```
α=> \du student

                             List of roles
 Role name |             Attributes              |      Member of      
-----------+-------------------------------------+---------------------
 student   | Create role, Create DB, Replication | {pg_read_all_stats}

α=> SELECT * FROM pg_stat_replication \gx

-[ RECORD 1 ]----+------------------------------
pid              | 10440
usesysid         | 16384
usename          | student
application_name | 13/beta
client_addr      | 
client_hostname  | 
client_port      | -1
backend_start    | 2024-01-16 12:17:17.774325+03
backend_xmin     | 
state            | streaming
sent_lsn         | 0/501BFC0
write_lsn        | 0/501BFC0
flush_lsn        | 0/501BFC0
replay_lsn       | 0/501BFC0
write_lag        | 00:00:00.000123
flush_lag        | 00:00:00.004935
replay_lag       | 00:00:00.004976
sync_priority    | 0
sync_state       | async
reply_time       | 2024-01-16 12:17:24.807756+03
```
Обратите внимание на поля *_lsn (и *_lag) — они показывают отставание реплики на разных этапах. Сейчас все позиции совпадают, отставание нулевое.

Теперь вставим в таблицу большое количество строк, чтобы увидеть репликацию в процессе работы.
```
α=> INSERT INTO test SELECT 'Just a line' FROM generate_series(1,1000000);

INSERT 0 1000000

α=> SELECT *, pg_current_wal_lsn() from pg_stat_replication \gx

-[ RECORD 1 ]------+------------------------------
pid                | 10440
usesysid           | 16384
usename            | student
application_name   | 13/beta
client_addr        | 
client_hostname    | 
client_port        | -1
backend_start      | 2024-01-16 12:17:17.774325+03
backend_xmin       | 
state              | streaming
sent_lsn           | 0/697A000
write_lsn          | 0/691A000
flush_lsn          | 0/68FA000
replay_lsn         | 0/68B9FF8
write_lag          | 00:00:02.241692
flush_lag          | 00:00:02.247271
replay_lag         | 00:00:02.258431
sync_priority      | 0
sync_state         | async
reply_time         | 2024-01-16 12:17:27.97807+03
pg_current_wal_lsn | 0/94F9C90
```
Видно, что возникла небольшая задержка. Однако не следует рассчитывать на точность выше чем полсекунды, так как статистика обновляется примерно с такой частотой.

Проверим реплику:
```
β=> SELECT count(*) FROM test;

  count  
---------
 1000001
(1 row)
```
Все строки успешно доехали.

И еще раз проверим состояние репликации:
```
α=> SELECT *, pg_current_wal_lsn() from pg_stat_replication \gx

-[ RECORD 1 ]------+------------------------------
pid                | 10440
usesysid           | 16384
usename            | student
application_name   | 13/beta
client_addr        | 
client_hostname    | 
client_port        | -1
backend_start      | 2024-01-16 12:17:17.774325+03
backend_xmin       | 
state              | streaming
sent_lsn           | 0/94F9C90
write_lsn          | 0/94F9C90
flush_lsn          | 0/94F9C90
replay_lsn         | 0/94F9C90
write_lag          | 00:00:01.880631
flush_lag          | 00:00:01.886028
replay_lag         | 00:00:02.019843
sync_priority      | 0
sync_state         | async
reply_time         | 2024-01-16 12:17:29.808353+03
pg_current_wal_lsn | 0/94F9C90
```
Все позиции выровнялись. 
### *Влияние слота репликации*

Остановим реплику.
```
student$ sudo pg_ctlcluster 13 beta stop
```
Ограничим размер WAL двумя сегментами.
```
α=> \c - postgres

You are now connected to database "replica_physical" as user "postgres".

α=> ALTER SYSTEM SET min_wal_size='32MB';

ALTER SYSTEM

α=> ALTER SYSTEM SET max_wal_size='32MB';

ALTER SYSTEM

α=> SELECT pg_reload_conf();

 pg_reload_conf 
----------------
 t
(1 row)
```
Слот неактивен, но помнит номер последней записи, полученной репликой:
```
α=> SELECT active, restart_lsn, wal_status FROM pg_replication_slots \gx

-[ RECORD 1 ]----------
active      | f
restart_lsn | 0/94F9C90
wal_status  | reserved
```
Вставим еще строки в таблицу.
```
α=> INSERT INTO test SELECT 'Just a line' FROM generate_series(1,1000000);

INSERT 0 1000000
```
Номер LSN в слоте не изменился:
```
α=> SELECT active, restart_lsn, wal_status FROM pg_replication_slots \gx

-[ RECORD 1 ]----------
active      | f
restart_lsn | 0/94F9C90
wal_status  | extended
```
Выполним контрольную точку, которая должна очистить журнал, поскольку его размер превышает допустимый.
```
α=> CHECKPOINT;

CHECKPOINT
```
Каков теперь размер журнала?
```
α=> SELECT pg_size_pretty(sum(size)) FROM pg_ls_waldir();

 pg_size_pretty 
----------------
 80 MB
(1 row)
```
Контрольная точка не удалила сегменты — их удерживает слот, несмотря на превышение лимита. Если оставить слот без присмотра, дисковое пространство может быть исчерпано.

Скорее всего, бесперебойная работа основного сервера важнее, чем синхронизация реплики. Чтобы предотвратить разрастание журнала, можно ограничить объем, удерживаемый слотом:
```
α=> ALTER SYSTEM SET max_slot_wal_keep_size='16MB';

ALTER SYSTEM

α=> SELECT pg_reload_conf();

 pg_reload_conf 
----------------
 t
(1 row)

α=> CHECKPOINT;

CHECKPOINT
```
Теперь журнал очищается контрольной точкой:
```
α=> SELECT pg_size_pretty(sum(size)) FROM pg_ls_waldir();

 pg_size_pretty 
----------------
 32 MB
(1 row)
```
Но слот уже не обеспечивает наличие записей WAL (wal_status=lost):
```
α=> SELECT active, restart_lsn, wal_status FROM pg_replication_slots \gx

-[ RECORD 1 ]-----
active      | f
restart_lsn | 
wal_status  | lost
```
А реплика не может синхронизироваться:
```
student$ sudo pg_ctlcluster 13 beta start

student$ sudo tail -n 2 /var/log/postgresql/postgresql-13-beta.log

2024-01-16 12:17:37.044 MSK [11864] LOG:  started streaming WAL from primary at 0/9000000 on timeline 1
2024-01-16 12:17:37.044 MSK [11864] FATAL:  could not receive data from WAL stream: ERROR:  requested WAL segment 000000010000000000000009 has already been removed
```
Для синхронизации реплики в таких случаях можно воспользоваться архивом, задав параметр restore_command. Если это невозможно, придется заново настроить репликацию, повторив формирование базовой копии. 
