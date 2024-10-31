# *Лабараторная работа №14*
# *Резервное копирование* 
## **COPY**
Существует два вида резервирования: логическое и физическое.
Логическая резервная копия - это набор команд SQL, восстанавливающий кластер с нуля.
### *Ход работы*
Для начала создадим БД и таблицу в ней.
```
=> CREATE DATABASE backup_overview;

CREATE DATABASE

=> \c backup_overview

You are now connected to database "backup_overview" as user "postgres".

=> create table t(id numeric, s text);

CREATE TABLE

=> insert into t values (1, 'Привет!'), (2, ''), (3, NULL);

INSERT 0 3

=> select * from t;

 id |    s    
----+---------
  1 | Привет!
  2 | 
  3 | 
(3 rows)
```

Вот как выглядит таблица в выводе команды COPY:

```
=> copy t to stdout;
1	Привет!
2	
3	\N
```

## **Утилита pg_dump**
Посмотрим на результат работы утилиты pg_dump в простом формате (plain).
### *Ход работы*
```
pg_dump -d backup_overview --create

--
-- PostgreSQL database dump
--

-- Dumped from database version 14.13 (Ubuntu 14.13-0ubuntu0.22.04.1)
-- Dumped by pg_dump version 14.13 (Ubuntu 14.13-0ubuntu0.22.04.1)

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

--
-- Name: backup_overview; Type: DATABASE; Schema: -; Owner: postgres
--

CREATE DATABASE backup_overview WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE = 'ru_RU.UTF-8';


ALTER DATABASE backup_overview OWNER TO postgres;

\connect backup_overview

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- Name: t; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.t (
    id numeric,
    s text
);


ALTER TABLE public.t OWNER TO postgres;

--
-- Data for Name: t; Type: TABLE DATA; Schema: public; Owner: postgres
--

COPY public.t (id, s) FROM stdin;
\.


--
-- PostgreSQL database dump complete
--
```

В качестве примера использования скопируем таблицу в другую базу.

```
=> CREATE DATABASE backup_overview2;

CREATE DATABASE

=> pg_dump -d backup_overview --table=t | psql -d backup_overview2

SET
SET
SET
SET
SET
 set_config 
------------
 
(1 row)

SET
SET
SET
SET
SET
SET
CREATE TABLE
ALTER TABLE
COPY 3

=> psql -d backup_overview

=> SELECT * FROM t;

 id |     s     
----+-----------
  1 | Hi there!
  2 | 
  3 | 
(3 rows)
```
## **Автономная резервная копия**
### *Ход работы*
 Значения параметров по умолчанию позволяют использовать репликацию:

```
=> SELECT name, setting FROM pg_settings WHERE name IN ('wal_level','max_wal_senders');

      name       | setting 
-----------------+---------
 max_wal_senders | 10
 wal_level       | replica
(2 rows)
```
Разрешение на локальное подключение по протоколу репликации в pg_hba.conf также прописано по умолчанию (хотя это и зависит от конкретной пакетной сборки):
```
=> SELECT type, database, user_name, address, auth_method FROM pg_hba_file_rules() WHERE 'replication' = ANY(database);

 type  |   database    | user_name |  address  |  auth_method  
-------+---------------+-----------+-----------+---------------
 local | {replication} | {all}     | <null>    | trust
 host  | {replication} | {all}     | 127.0.0.1 | scram-sha-256
 host  | {replication} | {all}     | ::1       | scram-sha-256
(3 rows)
```

Еще один кластер баз данных replica был предварительно инициализирован на порту 5433. Убедимся, что кластер остановлен, с помощью утилиты пакета для Ubuntu:

```
=> pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

 Создадим резервную копию. Используем формат по умолчанию (plain):
```
=> rm -rf /home//tmp/basebackup

=> pg_basebackup --pgdata=/home/ubun/tmp/basebackup --checkpoint=fast
```
Утилита pg_basebackup сразу после подключения к серверу выполняет контрольную точку. По умолчанию грязные буферы записываются постепенно, чтобы не создавать пиковую нагрузку (запись длится до 4,5 минут). Если указать --checkpoint=fast, буферы записываются без пауз. 
## **Восстановление**
### *Ход работы*
 Заменим каталог кластера replica созданной копией, предварительно убедившись, что кластер остановлен:
```
=>  sudo pg_ctlcluster 16 replica status

pg_ctl: no server running

=>  sudo rm -rf /var/lib/postgresql/16/replica

=>  sudo mv /home/ubun/tmp/basebackup/ /var/lib/postgresql/16/replica
```
Файлы кластера должны принадлежать пользователю postgres.
```
=>  sudo chown -R postgres:postgres /var/lib/postgresql/16/replica
```
Проверим содержимое каталога:
```
=>  sudo ls -l /var/lib/postgresql/16/replica

total 344
-rw------- 1 postgres postgres    225 июл  8 14:55 backup_label
-rw------- 1 postgres postgres 268206 июл  8 14:55 backup_manifest
drwx------ 8 postgres postgres   4096 июл  8 14:55 base
drwx------ 2 postgres postgres   4096 июл  8 14:55 global
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_commit_ts
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_dynshmem
drwx------ 4 postgres postgres   4096 июл  8 14:55 pg_logical
drwx------ 4 postgres postgres   4096 июл  8 14:55 pg_multixact
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_notify
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_replslot
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_serial
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_snapshots
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_stat
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_stat_tmp
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_subtrans
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_tblspc
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_twophase
-rw------- 1 postgres postgres      3 июл  8 14:55 PG_VERSION
drwx------ 3 postgres postgres   4096 июл  8 14:55 pg_wal
drwx------ 2 postgres postgres   4096 июл  8 14:55 pg_xact
-rw------- 1 postgres postgres     88 июл  8 14:55 postgresql.auto.conf
```
В процессе запуска произойдет восстановление из резервной копии.
```
=>  sudo pg_ctlcluster 16 replica start
```
Теперь оба сервера работают одновременно и независимо.

Основной сервер:
```
=> INSERT INTO t VALUES (4, 'Основной сервер');

INSERT 0 1

=> SELECT * FROM t;

 id |        s        
----+-----------------
  1 | Hi there!
  2 | 
  3 | <null>
  4 | Основной сервер
(4 rows)
```
Сервер, восстановленный из резервной копии:
```
=>  psql -p 5433 -d backup_overview

=> INSERT INTO t VALUES (4, 'Резервная копия');

INSERT 0 1

=> SELECT * FROM t;

 id |        s        
----+-----------------
  1 | Hi there!
  2 | 
  3 | 
  4 | Резервная копия
(4 rows)
```
