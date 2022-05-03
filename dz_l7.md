#1 создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL)
#2 зайдите в созданный кластер под пользователем postgres

user@comp2:~$ psql -h 127.0.0.1 -U postgres -W
Password: 
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.


#3 создайте новую базу данных testdb
#4 зайдите в созданную базу данных под пользователем postgres
#5 создайте новую схему testnm
#6 создайте новую таблицу t1 с одной колонкой c1 типа integer
#7 вставьте строку со значением c1=1

postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb
Password: 
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "postgres".

testdb=# create schema testnm;
CREATE SCHEMA
testdb=# set search_path to testnm;
SET

testdb=# create table t1 (c1 integer);
CREATE TABLE
testdb=# insert into t1 (c1) values (1);
INSERT 0 1

testdb=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 testnm | t1   | table | postgres
(1 row)

testdb=# select * from t1;
 c1 
----
  1
(1 row)


#8 создайте новую роль readonly
#9 дайте новой роли право на подключение к базе данных testdb

testdb=# create role readonly with password 'ro123';
CREATE ROLE

Проверяем в другом терминале:
user@comp2:~$ psql -h 127.0.0.1 -U readonly -W -d testdb
Password: 
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=>

Я смог подключиться к testdb сразу после создания роли, поэтому следующую команду можно не использовать.
testdb=# grant connect on database testdb to readonly;
GRANT

#10 дайте новой роли право на использование схемы testnm
#11 дайте новой роли право на select для всех таблиц схемы testnm

Проверяем в другом терминале:
testdb=> set search_path to testnm;
SET
testdb=> \dt
Did not find any relations.

Даём права на использование:
testdb=# grant usage on schema testnm to readonly;
GRANT

Проверяем в другом терминале:
testdb=> set search_path to testnm;
SET
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 testnm | t1   | table | postgres
(1 row)

testdb=> select * from t1;
ERROR:  permission denied for table t1

Даём права на select:
testdb=# grant select on all tables in schema testnm to readonly;
GRANT

Проверяем в другом терминале:
testdb=> select * from t1;
 c1 
----
  1
(1 row)

#12 создайте пользователя testread с паролем test123
#13 дайте роль readonly пользователю testread
#14 зайдите под пользователем testread в базу данных testdb
#15 сделайте select * from t1;

testdb=# create user testread with password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE

user@comp2:~$ psql -h 127.0.0.1 -U testread -W -d testdb
Password: 
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
 c1 
----
  1
(1 row)

#16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
#17 напишите что именно произошло в тексте домашнего задания

После создания и заполнения БД были даны права на чтение для новой роли readonly.
Затем права readonly были назначены созданному пользователю testread.

#18 у вас есть идеи почему? ведь права то дали?
#19 посмотрите на список таблиц

testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 testnm | t1   | table | postgres
(1 row)


#20 подсказка в шпаргалке под пунктом 20
#21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)

Я прочитал дaнные из таблицы.

#22 вернитесь в базу данных testdb под пользователем postgres
#23 удалите таблицу t1
#24 создайте ее заново но уже с явным указанием имени схемы testnm
#25 вставьте строку со значением c1=1

testdb=# \c testdb;
Password: 
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "postgres".
testdb=# \dt
Did not find any relations.
Правильно, мы же теперь в public.

testdb=# drop table testnm.t1;
DROP TABLE
testdb=# create table testnm.t1 (c1 int);
CREATE TABLE
testdb=# insert into testnm.t1 (c1) values (1);
INSERT 0 1


#26 зайдите под пользователем testread в базу данных testdb
#27 сделайте select * from testnm.t1;
#28 получилось?

user@comp2:~$ psql -h 127.0.0.1 -U testread -W -d testdb
Password: 
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1

#29 есть идеи почему? если нет - смотрите шпаргалку

Удалили объект и создали другой. Точно такой же, но другой. Т.е. нужно заново права назначить.

#30 как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

Здесь потребовалась помощь шпаргалки
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES

#31 сделайте select * from testnm.t1;
#32 получилось?
#33 есть идеи почему? если нет - смотрите шпаргалку

Права уже назначаны. Нужно ещё раз удалить и создать таблицу.

testdb=# drop table testnm.t1;
DROP TABLE
testdb=# create table testnm.t1 (c1 int);
CREATE TABLE
testdb=# insert into testnm.t1 (c1) values (1);
INSERT 0 1
testdb=# 

#31 сделайте select * from testnm.t1;
#32 получилось?
#33 ура!

user@comp2:~$ psql -h 127.0.0.1 -U testread -W -d testdb
Password: 
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
 c1 
----
  1
(1 row)

testdb=>

#34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
#35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?

testdb=> create table testnm.t2 (c2 int);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2 (c2 int);
                     ^
testdb=> create table t2 (c2 int);
CREATE TABLE
testdb=> insert into t2 (c2) values (2);
INSERT 0 1

Вполне предсказуемо для схем testnm и public.

#36 есть идеи как убрать эти права? если нет - смотрите шпаргалку
testdb=# revoke create on schema public from public;
REVOKE

#37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
#38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);

testdb=> insert into t2 (c2) values (23);
INSERT 0 1
testdb=> create table t3 (c3 int);
ERROR:  permission denied for schema public
LINE 1: create table t3 (c3 int);

#39 расскажите что получилось и почему 

Права на создание отобрали, а на вставку оставили.



