# *DBA3*
# *Лабораторная работа №3*
## *Архив журналов предзаписи*
### *Настройка непрерывной архивации*

Архив будем хранить в каталоге /var/lib/postgresql/archive. Он должен быть доступен пользователю-владельцу PostgreSQL.
```
student$ sudo mkdir /var/lib/postgresql/archive

student$ sudo chown postgres /var/lib/postgresql/archive
```
В реальной практике архив может размещаться на отдельном сервере или дисковой системе с доступом по сети.

Включим режим архивирования и установим команду копирования заполненных сегментов журнала.
```
α=> \c - postgres

You are now connected to database "student" as user "postgres".

α=> ALTER SYSTEM SET archive_mode = on;

ALTER SYSTEM
```
В archive_command мы сначала проверяем наличие файла с указанным именем в архиве, и копируем его только в случае отсутствия.
```
α=> ALTER SYSTEM SET archive_command = 'test ! -f /var/lib/postgresql/archive/%f && cp %p /var/lib/postgresql/archive/%f';

ALTER SYSTEM
```
В archive_command можно указать произвольную команду, лишь бы она завершалась со статусом 0 только в случае успеха. Например, можно организовать сжатие архивируемых сегментов:
```
α=> -- SET archive_command = 'test ! -f /var/lib/postgresql/archive/%f && gzip <%p >/var/lib/postgresql/archive/%f';
```
В идеале команда архивирования должна выполнять sync, чтобы файл гарантированно попал в энергонезависимую память. Иначе при сбое он может пропасть из архива, а сервер может успеть стереть его из каталога pg_wal.

В общем случае удобно поместить всю необходимую логику архивирования в отдельный скрипт и вызывать его:
```
α=> -- SET archive_command = 'archive.sh "%f" "%p"';
```
Изменение archive_mode требует рестарта сервера.
```
student$ sudo pg_ctlcluster 13 alpha restart
```
Проверим работу архивации. Создадим базу и таблицу.
```
student$ psql 

α=> CREATE DATABASE backup_archive;

CREATE DATABASE

α=> \c backup_archive

You are now connected to database "backup_archive" as user "student".

α=> CREATE TABLE t(s text);

CREATE TABLE

α=> INSERT INTO t VALUES ('Привет, мир!');

INSERT 0 1
```
Вот какой сегмент WAL используется сейчас:
```
α=> SELECT pg_walfile_name(pg_current_wal_lsn());

     pg_walfile_name      
--------------------------
 000000010000000000000003
(1 row)
```
Обратите внимание: первые восемь цифр в имени файла — номер текущей линии времени.

Чтобы заполнить файл, пришлось бы выполнить большое количество операций. В тестовых целях проще принудительно переключить сегмент:
```
student$ psql -U postgres -c "SELECT pg_switch_wal()"

 pg_switch_wal 
---------------
 0/301C460
(1 row)

α=> INSERT INTO t VALUES ('Доброе утро, страна!');

INSERT 0 1
```
Сегмент сменился:
```
α=> SELECT pg_walfile_name(pg_current_wal_lsn());

     pg_walfile_name      
--------------------------
 000000010000000000000004
(1 row)
```
А предыдущий должен был попасть в архив. Проверим:
```
student$ sudo ls -l /var/lib/postgresql/archive

total 16384
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000003
```
Текущий статус архивации показывает представление pg_stat_archiver:
```
α=> SELECT * FROM pg_stat_archiver \gx

-[ RECORD 1 ]------+------------------------------
archived_count     | 1
last_archived_wal  | 000000010000000000000003
last_archived_time | 2024-01-16 12:16:14.241057+03
failed_count       | 0
last_failed_wal    | 
last_failed_time   | 
stats_reset        | 2024-01-16 12:16:06.643908+03
```
Таким образом, файловая архивация настроена и работает. 
### *Потоковый архив*

Чтобы настроить пополнение архива по протоколу потоковой репликации, остановим сначала файловую архивацию.
```
α=> \c - postgres

You are now connected to database "backup_archive" as user "postgres".

α=> ALTER SYSTEM RESET archive_mode;

ALTER SYSTEM

α=> ALTER SYSTEM RESET archive_command;

ALTER SYSTEM

student$ sudo pg_ctlcluster 13 alpha restart
```
Текущее состояние архива.
```
student$ sudo ls -l /var/lib/postgresql/archive

total 32768
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000003
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000004
```
Сначала попросим утилиту pg_receivewal создать слот, чтобы гарантировать получение всех записей журнала.
```
postgres$ pg_receivewal --create-slot --slot=archive
```
Затем запустим ее фоном в режиме архивации. Увидев, что в архиве уже есть файлы, утилита запросит у сервера следующий сегмент, чтобы в архиве не было пропусков.
```
postgres$ pg_receivewal -D /var/lib/postgresql/archive --slot=archive
```
Добавим в таблицу много строк и удалим их.
```
student$ psql -d backup_archive

α=> INSERT INTO t SELECT 'И снова здравствуйте.' FROM generate_series(1,200000);

INSERT 0 200000

α=> DELETE FROM t WHERE s = 'И снова здравствуйте.';

DELETE 200000

α=> VACUUM t;

VACUUM
```
В архиве появились новые файлы.
```
student$ sudo ls -l /var/lib/postgresql/archive

total 65536
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000003
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000004
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000005
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000006.partial
```
Последний файл, скорее всего, имеет суффикс .partial — в него идет запись.

Теперь мы настроили потоковую архивацию. 
### *Базовая резервная копия*

Поскольку архив пополняется автоматически, попросим pg_basebackup не добавлять файлы журнала к резервной копии:
```
student$ pg_basebackup --wal-method=none --pgdata=/home/student/backup

NOTICE:  WAL archiving is not enabled; you must ensure that all required WAL segments are copied through other means to complete the backup
```
Каталог pg_wal резервной копии пуст:
```
student$ ls -l /home/student/backup/pg_wal/

total 4
drwx------ 2 student student 4096 янв 16 12:16 archive_status
```
А в архиве прибавилось файлов:
```
student$ sudo ls -l /var/lib/postgresql/archive

total 81920
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000003
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000004
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000005
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000006
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000007
```
Заглянем в сгенерированный файл метки:
```
student$ cat /home/student/backup/backup_label

START WAL LOCATION: 0/7000028 (file 000000010000000000000007)
CHECKPOINT LOCATION: 0/7000060
BACKUP METHOD: streamed
BACKUP FROM: master
START TIME: 2024-01-16 12:16:20 MSK
LABEL: pg_basebackup base backup
START TIMELINE: 1
```
Главная информация в этом файле — номер линии времени (строка START TIMELINE) и указание начальной точки для восстановления (START WAL LOCATION). Теоретически восстановление можно начать и с более ранней (но не более поздней) позиции, но это потребует больше времени. 
### *Восстановление из базовой резервной копии*

Зададим настройки восстановления на сервере beta (до версии 12 их нужно помещать в отдельный файл recovery.conf). В простейшем случае достаточно указать команду восстановления, которая будет копировать указанный сегмент WAL обратно из архива по указанному пути:
```
student$ echo "restore_command = 'cp /var/lib/postgresql/archive/%f %p'" >>/home/student/backup/postgresql.auto.conf
```
Если не указана целевая точка восстановления (один из параметров recovery_target*), то к базовой резервной копии будут применены записи WAL из всех файлов в архиве.

Наличие файла recovery.signal — указание серверу при старте войти в режим управляемого восстановления:
```
student$ touch /home/student/backup/recovery.signal
```
Выкладываем резервную копию в каталог данных сервера beta и запускаем его.
```
student$ sudo pg_ctlcluster 13 beta status

Error: /var/lib/postgresql/13/beta is not accessible or does not exist

student$ sudo rm -rf /var/lib/postgresql/13/beta

student$ sudo mv /home/student/backup /var/lib/postgresql/13/beta

student$ sudo chown -R postgres /var/lib/postgresql/13/beta

student$ sudo pg_ctlcluster 13 beta start
```
После успешного восстановления файл recovery.signal удаляется, а файл метки переименовывается в backup_label.old:
```
student$ sudo ls -l /var/lib/postgresql/13/beta | egrep 'recovery.*|backup_label.*'

-rw------- 1 postgres student    224 янв 16 12:16 backup_label.old
```
Проверим, что восстановлено в таблице:
```
student$ psql -p 5433 -d backup_archive

β=> SELECT * FROM t;

          s           
----------------------
 Привет, мир!
 Доброе утро, страна!
(2 rows)
```
### *Линии времени*

Что стало с линией времени после восстановления?
```
β=> SELECT pg_walfile_name(pg_current_wal_lsn());

     pg_walfile_name      
--------------------------
 000000020000000000000008
(1 row)
```
Номер увеличился на единицу.

В каталоге pg_wal появился файл истории, соответствующий этой ветви:
```
postgres$ ls -l /var/lib/postgresql/13/beta/pg_wal/00000002.history

-rw------- 1 postgres student 41 янв 16 12:16 /var/lib/postgresql/13/beta/pg_wal/00000002.history
```
В нем есть информация о «точках ветвления», через которые мы пришли в данную линию времени:
```
student$ sudo cat /var/lib/postgresql/13/beta/pg_wal/00000002.history

1	0/8000000	no recovery target specified
```
Эти файлы PostgreSQL использует, когда мы указываем линию времени в параметре recovery_target_timeline. Поэтому файлы истории подлежат архивации вместе с сегментами WAL, и удалять из архива их не надо. 
### *Очистка архива*

Сейчас архив первого сервера содержит ненужные файлы, они попали туда до формирования базовой копии:
```
student$ sudo ls -l /var/lib/postgresql/archive

total 98304
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000003
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000004
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000005
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000006
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000007
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000008.partial
```
Утилите pg_archivecleanup нужно передать путь к архиву и имя последнего сохраняемого сегмента WAL (его можно найти в файле backup_label).
```
student$ sudo head -n 1 /var/lib/postgresql/13/beta/backup_label.old

START WAL LOCATION: 0/7000028 (file 000000010000000000000007)

student$ sudo pg_archivecleanup /var/lib/postgresql/archive 000000010000000000000007
```
Посмотрим, какие файлы остались в архиве:
```
student$ sudo ls -l /var/lib/postgresql/archive

total 32768
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000007
-rw------- 1 postgres postgres 16777216 янв 16 12:16 000000010000000000000008.partial
```
Архив очищен.

Чтобы архив очищался по окончании восстановления, можно задать параметр
```
α=> -- SET recovery_end_command = 'pg_archivecleanup /var/lib/postgresql/archive %r'
```
