# *Лабораторная работа №4*
## *Управление транзакциями* 
По умолчанию psql работает в режиме автофиксации:
```
postgres=# \echo :AUTOCOMMIT
on
```
 Это приводит к тому, что любая одиночная команда, выданная без явного указания начала транзакции, сразу же фиксируется.

Создадим таблицу с одной строкой:
``` 
postgres=# \echo :AUTOCOMMIT
on
postgres=# CREATE TABLE t(
  id integer,
  s text
);
CREATE TABLE

postgres=# INSERT INTO t(id, s) VALUES (1, 'foo');
INSERT 0 1

postgres=# SELECT * FROM t;
 id |  s  
----+-----
  1 | foo
(1 row)

postgres=# BEGIN;
BEGIN


postgres=*# INSERT INTO t(id, s) VALUES (2, 'bar');
INSERT 0 1


postgres=*#  SELECT * FROM t;
 id |  s  
----+-----
  1 | foo
  2 | bar
(2 rows)

postgres=*# COMMIT;
COMMIT

postgres=# SELECT * FROM t;
 id |  s  
----+-----
  1 | foo
  2 | bar
(2 rows)
```
Режим без автофиксации неявно начинает транзакцию при первой выданной команде; изменения надо фиксировать самостоятельно. 
```
postgres=# \set AUTOCOMMIT off
postgres=# INSERT INTO t(id, s) VALUES (3, 'baz');
INSERT 0 1

postgres=*# COMMIT;
COMMIT
postgres=# SELECT * FROM t;
 id |  s  
----+-----
  1 | foo
  2 | bar
  3 | baz
(3 rows)
```
Восстановим режим, в котором psql работает по умолчанию. 
```
postgres=*#  \set AUTOCOMMIT on
```
Отдельные изменения можно откатывать, не прерывая транзакцию целиком (хотя необходимость в этом возникает нечасто). 
```
postgres=*# BEGIN;
WARNING:  there is already a transaction in progress
BEGIN
postgres=*# SAVEPOINT sp;
SAVEPOINT
postgres=*# INSERT INTO t(id, s) VALUES (4, 'qux');
INSERT 0 1
postgres=*# SELECT * FROM t;
 id |  s  
----+-----
  1 | foo
  2 | bar
  3 | baz
  4 | qux
(4 rows)
```
 Теперь откатим все до точки сохранения.

Откат к точке сохранения не подразумевает передачу управления (то есть не работает как GOTO); отменяются только те изменения состояния БД, которые были выполнены от момента установки точки до текущего момента. 
```
postgres=*# ROLLBACK TO sp;
ROLLBACK
postgres=*# SELECT * FROM t;
 id |  s  
----+-----
  1 | foo
  2 | bar
  3 | baz
(3 rows)
```
Сейчас изменения отменены, но транзакция продолжается: 
```
postgres=*# INSERT INTO t(id, s) VALUES (4, 'xyz');
INSERT 0 1
postgres=*# SELECT * FROM t;
 id |  s  
----+-----
  1 | foo
  2 | bar
  3 | baz
  4 | xyz
(4 rows)
```
## *Подготовленные операторы*
В SQL оператор подготавливается командой PREPARE (эта команда является расширением PostgreSQL, она отсутствует в стандарте): 
```
postgres=*# PREPARE q(integer) AS
  SELECT * FROM t WHERE id = $1;
PREPARE
```
После подготовки оператор можно вызывать по имени, передавая фактические параметры: 
```
postgres=*# EXECUTE q(1);
 id |  s  
----+-----
  1 | foo
(1 row)
```
Все подготовленные операторы текущего сеанса можно увидеть в представлении: 
```
postgres=*# SELECT * FROM pg_prepared_statements \gx
-[ RECORD 1 ]---+---------------------------------
name            | q
statement       | PREPARE q(integer) AS           +
                |   SELECT * FROM t WHERE id = $1;
prepare_time    | 2024-10-28 11:33:27.997015+03
parameter_types | {integer}
result_types    | {integer,text}
from_sql        | t
generic_plans   | 0
custom_plans    | 1
```
## *Курсоры* 
При выполнении команды SELECT сервер передает, а клиент получает сразу все строки: 
```
postgres=*# SELECT * FROM t ORDER BY id;
 id |  s  
----+-----
  1 | foo
  2 | bar
  3 | baz
  4 | xyz
(4 rows)
```
Курсор позволяет получать данные построчно. 
```
postgres=*#  BEGIN;
WARNING:  there is already a transaction in progress
BEGIN
postgres=*# DECLARE c CURSOR FOR
  SELECT * FROM t ORDER BY id;
DECLARE CURSOR
postgres=*# FETCH c;
 id |  s  
----+-----
  1 | foo
(1 row)
```
 Этот размер играет важную роль, когда строк очень много: обрабатывать большой объем данных построчно крайне неэффективно.

Что, если в процессе чтения мы дойдем до конца таблицы? 
```
postgres=*# FETCH 2 c;
 id |  s  
----+-----
  2 | bar
  3 | baz
(2 rows)

postgres=*# FETCH 2 c;
 id |  s  
----+-----
  4 | xyz
(1 row)
```
По окончании работы открытый курсор закрывают, освобождая ресурсы: 
```
postgres=*# CLOSE c;
CLOSE CURSOR
postgres=*# COMMIT;
COMMIT
```