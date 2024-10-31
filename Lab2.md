# *Лабораторная работа №2*

Подключение
Запуск
```
$ psql -U postgres psql
```
 Проверим текущее подключение:
```
=> \conninfo

You are connected to database "sta" as user "sta" via socket in 
"/var/run/postgresql" at port "5432"
```
 Команды SQL, в отличие от команд psql, могут располагаться на нескольких строках. Для отправки команды SQL на выполнение завершаем ее точкой с запятой:
```
=> SELECT schemaname, tablename, tableowner 
FROM pg_tables 
LIMIT 5;
```
вывод на экран: 
```
 schemaname |       tablename       | tableowner 
------------+-----------------------+------------
 pg_catalog | pg_statistic          | postgres
 pg_catalog | pg_type               | postgres
 pg_catalog | pg_foreign_table      | postgres
 pg_catalog | pg_authid             | postgres
 pg_catalog | pg_statistic_ext_data | postgres
(5 rows)
```
 Отключим выравнивание, вывод заголовка и итоговой строки, а в качестве разделителя столбцов установим пробел:
```
=> \t \a

Tuples only is on.
Output format is unaligned.


=> \pset fieldsep ' '

Field separator is " ".



=> SELECT schemaname, tablename, tableowner FROM pg_tables LIMIT 5;

pg_catalog pg_statistic postgres
pg_catalog pg_type postgres
pg_catalog pg_foreign_table postgres
pg_catalog pg_authid postgres
pg_catalog pg_statistic_ext_data postgres

=> \t \a

Tuples only is off.
Output format is aligned
```
 Расширенный формат удобен, когда нужно вывести много столбцов для одной или нескольких записей:
```
=> \x

Expanded display is on.

=> SELECT * FROM pg_tables WHERE tablename = 'pg_class';

-[ RECORD 1 ]-----------
schemaname  | pg_catalog
tablename   | pg_class
tableowner  | postgres
tablespace  | 
hasindexes  | t
hasrules    | f
hastriggers | f
rowsecurity | f

=> \x

Expanded display is off.
```
 Расширенный формат можно установить только для одного запроса, если в конце вместо точки с запятой указать \gx:
```
=> SELECT * FROM pg_tables WHERE tablename = 'pg_proc' \gx

-[ RECORD 1 ]-----------
schemaname  | pg_catalog
tablename   | pg_proc
tableowner  | postgres
tablespace  | 
hasindexes  | t
hasrules    | f
hastriggers | f
rowsecurity | f

```
 Все возможности форматирования результатов запросов доступны через команду \pset. А без параметров она покажет текущие настройки форматирования:
```
=> \pset

border                   1
columns                  0
csv_fieldsep             ','
expanded                 off
fieldsep                 ' '
fieldsep_zero            off
footer                   on
format                   aligned
linestyle                ascii
null                     ''
numericlocale            off
pager                    1
pager_min_lines          0
recordsep                '\n'
recordsep_zero           off
tableattr                
title                    
tuples_only              off
unicode_border_linestyle single
unicode_column_linestyle single
unicode_header_linestyle single
xheader_width            full
```
 В psql можно выполнять команды shell:
```
=> \! pwd

/home/salakhov
```
 Можно установить переменную окружения операционной системы: 
```
postgres=# \salakhov HELLO Hello
invalid command \salakhov
Try \? for help.
postgres=# \! echo $HELLO
```
Можно записать вывод команды в файл с помощью \o[ut]: 
```
postgres=# \o tmp/dba1_log
tmp/dba1_log: Отказано в доступе
postgres=# SELECT schemaname, tablename, tableowner FROM pg_tables LIMIT 5;
 schemaname |       tablename       | tableowner 
------------+-----------------------+------------
 pg_catalog | pg_statistic          | postgres
 pg_catalog | pg_type               | postgres
 pg_catalog | pg_foreign_table      | postgres
 pg_catalog | pg_authid             | postgres
 pg_catalog | pg_statistic_ext_data | postgres
(5 rows)

postgres=# \! cat tmp/dba1_log
cat: tmp/dba1_log: Отказано в доступе
```
 Вернем вывод на экран:
```
 \o
```
Выполнение скриптов 
```
postgres=# SELECT format('SELECT count(*) FROM %I;', tablename)
FROM pg_tables
LIMIT 3
\g (tuples_only=on format=unaligned) | cat -n
     1	SELECT count(*) FROM pg_statistic;
     2	SELECT count(*) FROM pg_type;
     3	SELECT count(*) FROM pg_foreign_table;
```
В команде \g можно указать имя файла, в который будет направлен вывод: 
```
SELECT format('SELECT count(*) FROM %I;', tablename)
FROM pg_tables
LIMIT 3
\g (tuples_only=on format=unaligned) tmp/dba1_log
tmp/dba1_log: Отказано в доступе

postgres=# \! cat tmp/dba1_log
cat: tmp/dba1_log: Отказано в доступе

postgres=# \i tmp/dba1_log
tmp/dba1_log: Отказано в доступе

postgres=# SELECT format('SELECT count(*) FROM %I;', tablename)
FROM pg_tables
LIMIT 3
\gexec
 count 
-------
   409
(1 row)

 count 
-------
   613
(1 row)

 count 
-------
     0
(1 row)
```
Запомним в переменной psql User значение переменной окружения USER: 
```
postgres=# \getenv salakhov salakhov
postgres=# \set Test Hi
postgres=# \echo :Test :User!
Hi postgres!
```
Сбросить переменную можно так: 
```
postgres=# \echo :Test
Hi
```
Результат запроса можно записать в переменную. Для этого запрос нужно завершить командой \gset: 
```
postgres=# SELECT now() AS curr_time \gset
postgres=# \echo :curr_time
2024-10-28 11:05:11.71172+03
```
Без параметров \set выдает значения всех установленных переменных: 
```
postgres=# \set
AUTOCOMMIT = 'on'
COMP_KEYWORD_CASE = 'preserve-upper'
DBNAME = 'postgres'
ECHO = 'none'
ECHO_HIDDEN = 'off'
ENCODING = 'UTF8'
ERROR = 'false'
FETCH_COUNT = '0'
HIDE_TABLEAM = 'off'
HIDE_TOAST_COMPRESSION = 'off'
HISTCONTROL = 'none'
HISTSIZE = '500'
HOST = '/var/run/postgresql'
IGNOREEOF = '0'
LAST_ERROR_MESSAGE = ''
LAST_ERROR_SQLSTATE = '00000'
ON_ERROR_ROLLBACK = 'off'
ON_ERROR_STOP = 'off'
PORT = '5432'
PROMPT1 = '%/%R%x%# '
PROMPT2 = '%/%R%x%# '
PROMPT3 = '>> '
QUIET = 'off'
ROW_COUNT = '1'
SERVER_VERSION_NAME = '16.4 (Ubuntu 16.4-0ubuntu0.24.04.2)'
SERVER_VERSION_NUM = '160004'
SHELL_ERROR = 'true'
SHELL_EXIT_CODE = '1'
SHOW_ALL_RESULTS = 'on'
SHOW_CONTEXT = 'errors'
SINGLELINE = 'off'
SINGLESTEP = 'off'
SQLSTATE = '00000'
Test = 'Hi'
USER = 'postgres'
User = 'postgres'
VERBOSITY = 'default'
VERSION = 'PostgreSQL 16.4 (Ubuntu 16.4-0ubuntu0.24.04.2) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.2.0-23ubuntu4) 13.2.0, 64-bit'
VERSION_NAME = '16.4 (Ubuntu 16.4-0ubuntu0.24.04.2)'
VERSION_NUM = '160004'
curr_time = '2024-10-28 11:05:11.71172+03'
```
 В скриптах можно использовать условные операторы.
Предположим, что мы хотим проверить, установлено ли значение переменной working_dir, и если нет, то присвоить ей имя текущего каталога. Для проверки существования переменной можно использовать следующую конструкцию, возвращающую логическое значение:
``` 
postgres=# \echo :{?working_dir}
FALSE
```
Следующий условный оператор psql проверяет существование переменной и при необходимости устанавливает значение по умолчанию:
```
postgres@# \if :{?working_dir}                    
\else
\set working_dir `pwd`
\endif
\set command ignored; use \endif or Ctrl-C to exit current \if block
```
Теперь мы можем быть уверены, что переменная working_dir определена: 
```
postgres@# \echo :working_dir
\echo command ignored; use \endif or Ctrl-C to exit current \if block
```