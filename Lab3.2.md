# *DBA3*
# *Лабораторная работа №2*
## *Базовая резервная копия*
### *Холодная файловая копия*

Файлы остановленного кластера можно скопировать и запустить с ними второй сервер. При этом не важно, был ли сервер остановлен корректно.

Создадим базу данных и таблицу.
```
student$ sudo pg_ctlcluster 13 alpha start

student$ psql 

α=> CREATE DATABASE backup_base;

CREATE DATABASE

α=> \c backup_base

You are now connected to database "backup_base" as user "student".

α=> CREATE TABLE t(s text);

CREATE TABLE

α=> INSERT INTO t VALUES ('Привет, мир!');

INSERT 0 1
```
Аварийно останавливаем сервер и копируем файлы на сервер beta:
```
student$ sudo head -n 1 /var/lib/postgresql/13/alpha/postmaster.pid

6241

student$ sudo kill -9 6241

student$ sudo pg_ctlcluster 13 beta status

Error: /var/lib/postgresql/13/beta is not accessible or does not exist

student$ sudo rm -rf /var/lib/postgresql/13/beta

student$ sudo cp -rp /var/lib/postgresql/13/alpha /var/lib/postgresql/13/beta
```
Сам резервный сервер уже предварительно собран и установлен.

Beta восстанавливает согласованность и запускается:
```
student$ sudo pg_ctlcluster 13 beta start

student$ sudo tail -n 5 /var/log/postgresql/postgresql-13-beta.log

2024-01-16 12:15:13.144 MSK [6609] LOG:  database system was not properly shut down; automatic recovery in progress
2024-01-16 12:15:13.160 MSK [6609] LOG:  redo starts at 0/3000F20
2024-01-16 12:15:13.163 MSK [6609] LOG:  invalid record length at 0/301BEF8: wanted 24, got 0
2024-01-16 12:15:13.163 MSK [6609] LOG:  redo done at 0/301BED0
2024-01-16 12:15:13.476 MSK [6608] LOG:  database system is ready to accept connections

student$ psql -p 5433 -d backup_base

β=> SELECT * FROM t;

      s       
--------------
 Привет, мир!
(1 row)

student$ sudo pg_ctlcluster 13 beta stop
```
### *Базовая резервная копия*

Теперь мы хотим сделать базовую копию работающего сервера.
```
student$ sudo pg_ctlcluster 13 alpha start
```
Значения параметров по умолчанию позволяют сразу использовать протокол репликации:
```
student$ psql -U postgres

α=> SELECT name, setting
FROM pg_settings
WHERE name IN ('wal_level','max_wal_senders','max_replication_slots');

         name          | setting 
-----------------------+---------
 max_replication_slots | 10
 max_wal_senders       | 10
 wal_level             | replica
(3 rows)
```
Разрешение на локальное подключение по протоколу репликации в pg_hba.conf также прописано по умолчанию (хотя это и зависит от конкретной пакетной сборки):
```
α=> SELECT type, database, user_name, address, auth_method
FROM pg_hba_file_rules()
WHERE 'replication' = ANY(database);

 type  |   database    | user_name |  address  | auth_method 
-------+---------------+-----------+-----------+-------------
 local | {replication} | {all}     |           | trust
 host  | {replication} | {all}     | 127.0.0.1 | md5
 host  | {replication} | {all}     | ::1       | md5
(3 rows)
```
Чтобы утилита pg_basebackup могла подключиться к серверу под ролью student, эта роль должна иметь атрибут REPLICATION:
```
α=> \du student

                             List of roles
 Role name |             Attributes              |      Member of      
-----------+-------------------------------------+---------------------
 student   | Create role, Create DB, Replication | {pg_read_all_stats}
```
Выполним команду pg_basebackup. В нашем случае и сервер-источник, и резервная копия будут располагаться на одном сервере.

Если бы мы использовали табличные пространства, дополнительно пришлось бы указать для них другие пути в ключе --tablespace-mapping, но в данном случае этого не требуется.

Для мониторинга добавим ключ --progress. Ту же информацию можно получить в реальном времени из представления pg_stat_progress_basebackup.
```
student$ pg_basebackup --pgdata=/home/student/backup --progress

waiting for checkpoint
    0/40184 kB (0%), 0/1 tablespace
40194/40194 kB (100%), 0/1 tablespace
40194/40194 kB (100%), 1/1 tablespace
```
По умолчанию в начале копирования выполняется «протяженная» контрольная точка в соответствии с обычной настройкой. Это может занять заметное время: если контрольные точки выполняются по расписанию, то соответствующую долю от значения параметра checkpoint_timeout.
```
α=> SHOW checkpoint_timeout; SHOW checkpoint_completion_target;

 checkpoint_timeout 
--------------------
 5min
(1 row)

 checkpoint_completion_target 
------------------------------
 0.5
(1 row)
```
Если требуется выполнить контрольную точку как можно быстрее, надо указать ключ --checkpoint=fast.

Проверим содержимое каталога с данными, в который была записана базовая копия:
```
student$ ls -l /home/student/backup

total 300
-rw------- 1 student student    224 янв 16 12:15 backup_label
-rw------- 1 student student    224 янв 16 12:15 backup_label.old
-rw------- 1 student student 219599 янв 16 12:15 backup_manifest
drwx------ 7 student student   4096 янв 16 12:15 base
drwx------ 2 student student   4096 янв 16 12:15 global
drwx------ 2 student student   4096 янв 16 12:15 pg_commit_ts
drwx------ 2 student student   4096 янв 16 12:15 pg_dynshmem
drwx------ 4 student student   4096 янв 16 12:15 pg_logical
drwx------ 4 student student   4096 янв 16 12:15 pg_multixact
drwx------ 2 student student   4096 янв 16 12:15 pg_notify
drwx------ 2 student student   4096 янв 16 12:15 pg_replslot
drwx------ 2 student student   4096 янв 16 12:15 pg_serial
drwx------ 2 student student   4096 янв 16 12:15 pg_snapshots
drwx------ 2 student student   4096 янв 16 12:15 pg_stat
drwx------ 2 student student   4096 янв 16 12:15 pg_stat_tmp
drwx------ 2 student student   4096 янв 16 12:15 pg_subtrans
drwx------ 2 student student   4096 янв 16 12:15 pg_tblspc
drwx------ 2 student student   4096 янв 16 12:15 pg_twophase
-rw------- 1 student student      3 янв 16 12:15 PG_VERSION
drwx------ 3 student student   4096 янв 16 12:15 pg_wal
drwx------ 2 student student   4096 янв 16 12:15 pg_xact
-rw------- 1 student student     88 янв 16 12:15 postgresql.auto.conf
```
Все необходимые файлы журнала находятся в каталоге pg_wal:
```
student$ ls -l /home/student/backup/pg_wal/

total 16388
-rw------- 1 student student 16777216 янв 16 12:15 000000010000000000000004
drwx------ 2 student student     4096 янв 16 12:15 archive_status
```
### *Восстановление из базовой резервной копии*

Скопируем базовую копию в каталог данных сервера beta.
```
student$ sudo pg_ctlcluster 13 beta status

pg_ctl: no server running

student$ sudo rm -rf /var/lib/postgresql/13/beta/*

student$ sudo cp -r /home/student/backup/* /var/lib/postgresql/13/beta

student$ sudo chown -R postgres /var/lib/postgresql/13/beta
```
Запускаем второй сервер.
```
student$ sudo pg_ctlcluster 13 beta start
```
Теперь оба сервера работают одновременно и независимо. Проверим:
```
student$ psql -p 5433 -d backup_base

β=> SELECT * FROM t;

      s       
--------------
 Привет, мир!
(1 row)
```
### *Проверка целостности*

Резервная копия содержит файл манифеста, вот его начало и конец:
```
student$ head -n 5 ~/backup/backup_manifest

{ "PostgreSQL-Backup-Manifest-Version": 1,
"Files": [
{ "Path": "backup_label", "Size": 224, "Last-Modified": "2024-01-16 09:15:28 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "14719530" },
{ "Path": "pg_xact/0000", "Size": 8192, "Last-Modified": "2024-01-16 09:15:27 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "376e144c" },
{ "Path": "global/2677", "Size": 16384, "Last-Modified": "2022-07-14 08:23:57 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "7d22c055" },

student$ tail -n 6 ~/backup/backup_manifest

{ "Path": "global/pg_control", "Size": 8192, "Last-Modified": "2024-01-16 09:15:28 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "43872087" }
],
"WAL-Ranges": [
{ "Timeline": 1, "Start-LSN": "0/4000028", "End-LSN": "0/4000100" }
],
"Manifest-Checksum": "8576ce3318b828ab11e801515df2a80e09877498c890c530882dcff3f48567e7"}
```
Проверим целостность копии утилитой pg_verifybackup:
```
student$ /usr/lib/postgresql/13/bin/pg_verifybackup ~/backup

backup successfully verified
```
Утилита проверяет по манифесту наличие файлов, их контрольные суммы, а также наличие и возможность чтения всех записей WAL, необходимых для восстановления. 