# На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
# Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.

root@comp2:/home/user# pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log

root@comp2:/home/user# psql -h 127.0.0.1 -p 5432 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# create table tbl1 ( i int, c char(10));
CREATE TABLE

postgres=# insert into tbl1 (i,c) values (1,'t1'),(2,'t1'),(3,'t1');
INSERT 0 3

postgres=# select * from tbl1;
 i |     c      
---+------------
 1 | t1        
 2 | t1        
 3 | t1        
(3 rows)

postgres=# create publication tbl1_pub for table tbl1;
CREATE PUBLICATION

postgres=# alter system set wal_level=logical;
ALTER SYSTEM

postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# \q

root@comp2:/home/user# pg_ctlcluster restart 12 main

root@comp2:/home/user# psql -h 127.0.0.1 -p 5432 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# show wal_level;
 wal_level 
-----------
 logical
(1 row)

/q


# На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение. Создаем публикацию таблицы 
# test2 и подписываемся на публикацию таблицы test1 с ВМ №1. 

root@comp2:/home/user# pg_createcluster 12 main2
Creating new PostgreSQL cluster 12/main2 ...
/usr/lib/postgresql/12/bin/initdb -D /var/lib/postgresql/12/main2 --auth-local peer --auth-host md5
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locales
  COLLATE:  en_US.UTF-8
  CTYPE:    en_US.UTF-8
  MESSAGES: en_US.UTF-8
  MONETARY: ru_RU.UTF-8
  NUMERIC:  ru_RU.UTF-8
  TIME:     ru_RU.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/12/main2 ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

Success. You can now start the database server using:

    pg_ctlcluster 12 main2 start

Ver Cluster Port Status Owner    Data directory               Log file
12  main2   5433 down   postgres /var/lib/postgresql/12/main2 /var/log/postgresql/postgresql-12-main2.log

root@comp2:/home/user# pg_ctlcluster start 12 main2

Правим конфиги и заходим

root@comp2:/home/user# psql -h 127.0.0.1 -p 5433 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# create table tbl2 ( i int, c char(10));
CREATE TABLE
postgres=# insert into tbl2 (i,c) values (1,'t2'),(2,'t2'),(3,'t2');
INSERT 0 3
postgres=# create publication tbl2_pub for table tbl2;
CREATE PUBLICATION

postgres=# alter system set wal_level=logical;
ALTER SYSTEM

postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# \dRp+
                      Publication tbl2_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates 
----------+------------+---------+---------+---------+-----------
 postgres | f          | t       | t       | t       | t
Tables:
    "public.tbl2"

postgres=# \q
root@comp2:/home/user# pg_ctlcluster restart 12 main2

Теперь создадим подписки.

oot@comp2:/home/user# psql -h 127.0.0.1 -p 5432 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# create table tbl2 ( i int, c char(10));
CREATE TABLE

postgres=# create subscription tbl1_sub connection 'host=127.0.0.1 port=5433 user=postgres dbname=postgres' publication tbl2_pub with (copy_data=true);
NOTICE:  created replication slot "tbl1_sub" on publisher
CREATE SUBSCRIPTION
postgres=# drop subscription tbl1_sub;
NOTICE:  dropped replication slot "tbl1_sub" on publisher
DROP SUBSCRIPTION
postgres=# create subscription tbl2_sub connection 'host=127.0.0.1 port=5433 user=postgres dbname=postgres' publication tbl2_pub with (copy_data=true);
NOTICE:  created replication slot "tbl2_sub" on publisher
CREATE SUBSCRIPTION

Первый раз ошибся в имени подписки. Удалил и заново создал с правильным именем.
В итоге имеем:

postgres=# select * from tbl2;
 i |     c      
---+------------
 1 | t2        
 2 | t2        
 3 | t2        
 1 | t2        
 2 | t2        
 3 | t2        
(6 rows)

Теперь подпишемся на втором сервере.

root@comp2:/home/user# psql -h 127.0.0.1 -p 5433 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# create table tbl1 ( i int, c char(10));
CREATE TABLE

postgres=# create subscription tbl1_sub connection 'host=127.0.0.1 port=5432 user=postgres dbname=postgres' publication tbl1_pub with (copy_data=true);
NOTICE:  created replication slot "tbl1_sub" on publisher
CREATE SUBSCRIPTION

Проверяем.
Заходим в первый кластер (5432) и вставляем строку в tbl1.
Заходим во второй кластер (5433), вставляем строку в tbl2 и проверяем tbl1.
Заходим в первый кластер (5432) и проверяем tbl2.

root@comp2:/home/user# psql -h 127.0.0.1 -p 5432 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# insert into tbl1 (i,c) values (4,'t1');
INSERT 0 1

postgres=# \q

root@comp2:/home/user# psql -h 127.0.0.1 -p 5433 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# insert into tbl2 (i,c) values (4,'t2');
INSERT 0 1

postgres=# select * from tbl1;
 i |     c      
---+------------
 1 | t1        
 2 | t1        
 3 | t1        
 4 | t1        
(4 rows)

Репликация с первого кластера сработала.

postgres=# \q
root@comp2:/home/user# psql -h 127.0.0.1 -p 5432 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# select * from tbl2;
 i |     c      
---+------------
 1 | t2        
 2 | t2        
 3 | t2        
 1 | t2        
 2 | t2        
 3 | t2        
 4 | t2        
(7 rows)

Репликация со второго кластера сработала.

# 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). № 
# Небольшое описание, того, что получилось.

root@comp2:/home/user# pg_createcluster 12 main3
Creating new PostgreSQL cluster 12/main3 ...
/usr/lib/postgresql/12/bin/initdb -D /var/lib/postgresql/12/main3 --auth-local peer --auth-host md5
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locales
  COLLATE:  en_US.UTF-8
  CTYPE:    en_US.UTF-8
  MESSAGES: en_US.UTF-8
  MONETARY: ru_RU.UTF-8
  NUMERIC:  ru_RU.UTF-8
  TIME:     ru_RU.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/12/main3 ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

Success. You can now start the database server using:

    pg_ctlcluster 12 main3 start

Ver Cluster Port Status Owner    Data directory               Log file
12  main3   5434 down   postgres /var/lib/postgresql/12/main3 /var/log/postgresql/postgresql-12-main3.log

Правим конфиги.

root@comp2:/home/user# pg_ctlcluster start 12 main3

root@comp2:/home/user# psql -h 127.0.0.1 -p 5434 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# create database srv1;
CREATE DATABASE
postgres=# create database srv2;
CREATE DATABASE
postgres=# \c srv1
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "srv1" as user "postgres".
srv1=# create table tbl1 ( i int, c char(10));
CREATE TABLE
srv1=# create table tbl2 ( i int, c char(10));
CREATE TABLE

srv1=# \c srv2
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "srv2" as user "postgres".
srv2=# 
srv2=# create table tbl1 ( i int, c char(10));
CREATE TABLE
srv2=# create table tbl2 ( i int, c char(10));
CREATE TABLE

Таблицы подготовили. Теперь создаём републикации (tbl2 в первом кластере и tbl1 во втором кластере) и подписываемся на них.

root@comp2:/home/user# psql -h 127.0.0.1 -p 5432 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# \dt
          List of relations
 Schema |   Name   | Type  |  Owner   
--------+----------+-------+----------
 public | accounts | table | postgres
 public | tbl1     | table | postgres
 public | tbl2     | table | postgres
(3 rows)

postgres=# create publication tbl2_pub for table tbl2;
CREATE PUBLICATION

postgres=# \q
root@comp2:/home/user# psql -h 127.0.0.1 -p 5433 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

ostgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | tbl1 | table | postgres
 public | tbl2 | table | postgres
(2 rows)

postgres=# create publication tbl1_pub for table tbl1;
CREATE PUBLICATION

postgres=# \q
root@comp2:/home/user# psql -h 127.0.0.1 -p 5434 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

Подписываемся на публикации первого кластера.

postgres=# \c srv1;
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "srv1" as user "postgres".

srv1=# create subscription tbl1_sub connection 'host=127.0.0.1 port=5432 user=postgres dbname=postgres' publication tbl1_pub with (copy_data=true);
ERROR:  could not create replication slot "tbl1_sub": ERROR:  replication slot "tbl1_sub" already exists

srv1=# create subscription tbl1_sub2 connection 'host=127.0.0.1 port=5432 user=postgres dbname=postgres' publication tbl1_pub with (copy_data=true);
NOTICE:  created replication slot "tbl1_sub2" on publisher
CREATE SUBSCRIPTION

srv1=# create subscription tbl2_sub2 connection 'host=127.0.0.1 port=5432 user=postgres dbname=postgres' publication tbl2_pub with (copy_data=true);
NOTICE:  created replication slot "tbl2_sub2" on publisher
CREATE SUBSCRIPTION

srv1=# select * from tbl1;
 i |     c      
---+------------
 1 | t1        
 2 | t1        
 3 | t1        
 4 | t1        
(4 rows)

srv1=# select * from tbl2;
 i |     c      
---+------------
 1 | t2        
 2 | t2        
 3 | t2        
 1 | t2        
 2 | t2        
 3 | t2        
 4 | t2        
(7 rows)

Подписываемся на публикации второго кластера.

srv1=# \c srv2;
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "srv2" as user "postgres".

srv2=# create subscription tbl1_sub3 connection 'host=127.0.0.1 port=5433 user=postgres dbname=postgres' publication tbl1_pub with (copy_data=false);
NOTICE:  created replication slot "tbl1_sub3" on publisher
CREATE SUBSCRIPTION

srv2=# create subscription tbl2_sub3 connection 'host=127.0.0.1 port=5433 user=postgres dbname=postgres' publication tbl2_pub with (copy_data=false);
NOTICE:  created replication slot "tbl2_sub3" on publisher
CREATE SUBSCRIPTION

Проверяем.

root@comp2:/home/user# psql -h 127.0.0.1 -p 5432 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# insert into tbl1 (i,c) values (5,'t1');
INSERT 0 1

postgres=# \q
root@comp2:/home/user# psql -h 127.0.0.1 -p 5433 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# insert into tbl2 (i,c) values (5,'t2');
INSERT 0 1

postgres=# \q
root@comp2:/home/user# psql -h 127.0.0.1 -p 5434 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# \c srv1;
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "srv1" as user "postgres".
srv1=# select * from tbl1;
 i |     c      
---+------------
 1 | t1        
 2 | t1        
 3 | t1        
 4 | t1        
 5 | t1        
(5 rows)

srv1=# select * from tbl2;
 i |     c      
---+------------
 1 | t2        
 2 | t2        
 3 | t2        
 1 | t2        
 2 | t2        
 3 | t2        
 4 | t2        
 5 | t2        
(8 rows)

Все цепочки с первого кластера отработали.

srv1=# \c srv2;
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "srv2" as user "postgres".
srv2=# select * from tbl1;
 i |     c      
---+------------
 1 | t1        
 2 | t1        
 3 | t1        
 4 | t1        
 5 | t1        
(5 rows)

srv2=# select * from tbl2;
 i |     c      
---+------------
 5 | t2        
(1 row)

Все цепочки со второго кластера отработали. В tbl2 одна запись, т.к. при создании подписки данные не копировали.

# реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. 

root@comp2:/home/user# pg_createcluster 12 main4
Creating new PostgreSQL cluster 12/main4 ...
/usr/lib/postgresql/12/bin/initdb -D /var/lib/postgresql/12/main4 --auth-local peer --auth-host md5
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locales
  COLLATE:  en_US.UTF-8
  CTYPE:    en_US.UTF-8
  MESSAGES: en_US.UTF-8
  MONETARY: ru_RU.UTF-8
  NUMERIC:  ru_RU.UTF-8
  TIME:     ru_RU.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/12/main4 ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

Success. You can now start the database server using:

    pg_ctlcluster 12 main4 start

Ver Cluster Port Status Owner    Data directory               Log file
12  main4   5435 down   postgres /var/lib/postgresql/12/main4 /var/log/postgresql/postgresql-12-main4.log

Удаляем все файлы из /var/lib/postgresql/12/main4 и делаем бэкап.

root@comp2:/home/user# pg_basebackup -h 127.0.0.1 -p 5434 -U postgres -R -D /var/lib/postgresql/12/main4
root@comp2:/home/user# chown -R postgres:postgres /var/lib/postgresql/12/main4
root@comp2:/home/user# pg_ctlcluster start 12 main4


root@comp2:/home/user# psql -h 127.0.0.1 -p 5435 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 srv1      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 srv2      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(5 rows)

postgres=# \c srv1;
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "srv1" as user "postgres".
srv1=# select * from tbl2;
 i |     c      
---+------------
 1 | t2        
 2 | t2        
 3 | t2        
 1 | t2        
 2 | t2        
 3 | t2        
 4 | t2        
 5 | t2        
(8 rows)

Теперь проверим цепочки.

srv1=# \q
root@comp2:/home/user# psql -h 127.0.0.1 -p 5432 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# insert into tbl1 (i,c) values (6,'t1');
INSERT 0 1

postgres=# \q
root@comp2:/home/user# psql -h 127.0.0.1 -p 5433 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# insert into tbl2 (i,c) values (6,'t2');
INSERT 0 1

postgres=# \q
root@comp2:/home/user# psql -h 127.0.0.1 -p 5435 -U postgres
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# \c srv1;
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "srv1" as user "postgres".
srv1=# select * from tbl1;
 i |     c      
---+------------
 1 | t1        
 2 | t1        
 3 | t1        
 4 | t1        
 5 | t1        
 6 | t1        
(6 rows)

srv1=# select * from tbl2;
 i |     c      
---+------------
 1 | t2        
 2 | t2        
 3 | t2        
 1 | t2        
 2 | t2        
 3 | t2        
 4 | t2        
 5 | t2        
 6 | t2        
(9 rows)

srv1=# \c srv2;
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "srv2" as user "postgres".
srv2=# select * from tbl1;
 i |     c      
---+------------
 1 | t1        
 2 | t1        
 3 | t1        
 4 | t1        
 5 | t1        
 6 | t1        
(6 rows)

srv2=# select * from tbl2;
 i |     c      
---+------------
 5 | t2        
 6 | t2        
(2 rows)

Всё отработало нормально.

# Написать с какими проблемами столкнулись.
Технически, ничего сложного.



