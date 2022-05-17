#    Настройте выполнение контрольной точки раз в 30 секунд.

postgres=# select name,setting from pg_settings where name like '%point%';
             name             | setting 
------------------------------+---------
 checkpoint_completion_target | 0.9
 checkpoint_flush_after       | 32
 checkpoint_timeout           | 30
 checkpoint_warning           | 30
 log_checkpoints              | off

#    10 минут c помощью утилиты pgbench подавайте нагрузку.
#    Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем #приходится в среднем на одну контрольную точку.

Удаляем все wal-файлы и запускаем pgbench.

root@debian:/# pgbench -c8 -P 1 -T 600 postgres -h 127.0.0.1 -U postgres

...
progress: 597.0 s, 1001.4 tps, lat 7.824 ms stddev 2.589
progress: 598.0 s, 1003.1 tps, lat 7.706 ms stddev 2.795
progress: 599.0 s, 1125.5 tps, lat 5.544 ms stddev 2.749
progress: 600.0 s, 1013.3 tps, lat 7.720 ms stddev 2.211
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 599151
latency average = 7.736 ms
latency stddev = 2.893 ms
initial connection time = 19.836 ms
tps = 998.591098 (without initial connection time)

Смотрим на файлы

root@debian:/bitnami/postgresql/data/pg_wal# ls -l
total 180228
-rw------- 1 postgres postgres 16777216 May 15 14:38 00000001000000030000006A
-rw------- 1 postgres postgres 16777216 May 15 14:39 00000001000000030000006B
-rw------- 1 postgres postgres 16777216 May 15 14:39 00000001000000030000006C
-rw------- 1 postgres postgres 16777216 May 15 14:39 00000001000000030000006D
-rw------- 1 postgres postgres 16777216 May 15 14:39 00000001000000030000006E
-rw------- 1 postgres postgres 16777216 May 15 14:40 00000001000000030000006F
-rw------- 1 postgres postgres 16777216 May 15 14:40 000000010000000300000070
-rw------- 1 postgres postgres 16777216 May 15 14:40 000000010000000300000071
-rw------- 1 postgres postgres 16777216 May 15 14:41 000000010000000300000072
-rw------- 1 postgres postgres 16777216 May 15 14:38 000000010000000300000073
-rw------- 1 postgres postgres 16777216 May 15 14:38 000000010000000300000074
drwx------ 2 postgres postgres     4096 May  5 17:00 archive_status

Первый файл заканчивался на 50, последний заканчивается на 74. 
Всего было сгенерировано 36 файлов.
Размер каждого файла - 16 Мб

Общий объем сгенерированных файлов = 36*16Мб = 576 Мб
Количество контрольных точек - 20.
Объем данных, приходящихся на 1 контрольную точку = 576Мб/20 = 28.8 Мб


Запускаем ещё раз.
...
progress: 596.0 s, 1008.1 tps, lat 7.455 ms stddev 2.933
progress: 597.0 s, 1102.9 tps, lat 5.578 ms stddev 2.856
progress: 598.0 s, 993.4 tps, lat 7.894 ms stddev 2.326
progress: 599.0 s, 993.1 tps, lat 7.876 ms stddev 2.379
progress: 600.0 s, 1030.0 tps, lat 6.942 ms stddev 3.195
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 584326
latency average = 7.906 ms
latency stddev = 2.884 ms
initial connection time = 22.296 ms
tps = 973.893517 (without initial connection time)

Смотрим файлы.

root@debian:/bitnami/postgresql/data/pg_wal# ls -l
total 180228
-rw------- 1 postgres postgres 16777216 May 15 15:30 00000001000000030000008D
-rw------- 1 postgres postgres 16777216 May 15 15:30 00000001000000030000008E
-rw------- 1 postgres postgres 16777216 May 15 15:31 00000001000000030000008F
-rw------- 1 postgres postgres 16777216 May 15 15:31 000000010000000300000090
-rw------- 1 postgres postgres 16777216 May 15 15:31 000000010000000300000091
-rw------- 1 postgres postgres 16777216 May 15 15:31 000000010000000300000092
-rw------- 1 postgres postgres 16777216 May 15 15:32 000000010000000300000093
-rw------- 1 postgres postgres 16777216 May 15 15:32 000000010000000300000094
-rw------- 1 postgres postgres 16777216 May 15 15:33 000000010000000300000095
-rw------- 1 postgres postgres 16777216 May 15 15:29 000000010000000300000096
-rw------- 1 postgres postgres 16777216 May 15 15:30 000000010000000300000097
drwx------ 2 postgres postgres     4096 May  5 17:00 archive_status

Первый файл заканчивался на 72. Всего было сгененрировань 35 файлов.

Общий объем сгенерированных файлов = 35*16Мб = 560 Мб
Количество контрольных точек - 20.
Объем данных, приходящихся на 1 контрольную точку = 560 Мб/20 = 28 Мб

#    Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему #так произошло?

Смотрим логи и видим, что все контрольные точки выполнялись точно по расписанию,
т.к. не был превышен параметр shared_buffers.

2022-05-15 15:22:39.095 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:22:50.671 GMT [1984] LOG:  checkpoint complete: wrote 12 buffers (6.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=11.522 s, sync=0.039 s, total=11.577 s; sync files=23, longest=0.032 s, average=0.002 s; distance=16104 kB, estimate=16104 kB
2022-05-15 15:23:09.059 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:23:18.287 GMT [1984] LOG:  checkpoint complete: wrote 19 buffers (9.5%); 0 WAL file(s) added, 0 removed, 0 recycled; write=9.151 s, sync=0.074 s, total=9.228 s; sync files=11, longest=0.072 s, average=0.007 s; distance=27973 kB, estimate=27973 kB
2022-05-15 15:23:39.081 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:23:51.023 GMT [1984] LOG:  checkpoint complete: wrote 19 buffers (9.5%); 0 WAL file(s) added, 0 removed, 0 recycled; write=11.890 s, sync=0.050 s, total=11.942 s; sync files=21, longest=0.028 s, average=0.003 s; distance=28286 kB, estimate=28286 kB
2022-05-15 15:24:09.082 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:24:21.091 GMT [1984] LOG:  checkpoint complete: wrote 18 buffers (9.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=11.965 s, sync=0.042 s, total=12.010 s; sync files=11, longest=0.038 s, average=0.004 s; distance=28379 kB, estimate=28379 kB
2022-05-15 15:24:39.087 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:24:50.920 GMT [1984] LOG:  checkpoint complete: wrote 16 buffers (8.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=11.776 s, sync=0.054 s, total=11.834 s; sync files=21, longest=0.029 s, average=0.003 s; distance=27710 kB, estimate=28312 kB
2022-05-15 15:25:09.035 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:25:21.038 GMT [1984] LOG:  checkpoint complete: wrote 22 buffers (11.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=11.956 s, sync=0.044 s, total=12.003 s; sync files=12, longest=0.041 s, average=0.004 s; distance=27702 kB, estimate=28251 kB
2022-05-15 15:25:39.067 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:25:50.515 GMT [1984] LOG:  checkpoint complete: wrote 18 buffers (9.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=11.401 s, sync=0.043 s, total=11.448 s; sync files=21, longest=0.029 s, average=0.003 s; distance=27572 kB, estimate=28183 kB
2022-05-15 15:26:09.068 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:26:21.038 GMT [1984] LOG:  checkpoint complete: wrote 21 buffers (10.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=11.931 s, sync=0.037 s, total=11.971 s; sync files=12, longest=0.020 s, average=0.004 s; distance=27644 kB, estimate=28129 kB
2022-05-15 15:26:39.116 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:26:51.017 GMT [1984] LOG:  checkpoint complete: wrote 15 buffers (7.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=11.840 s, sync=0.057 s, total=11.901 s; sync files=24, longest=0.033 s, average=0.003 s; distance=27550 kB, estimate=28071 kB
2022-05-15 15:27:09.099 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:27:20.951 GMT [1984] LOG:  checkpoint complete: wrote 18 buffers (9.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=11.816 s, sync=0.033 s, total=11.852 s; sync files=12, longest=0.017 s, average=0.003 s; distance=27829 kB, estimate=28047 kB
2022-05-15 15:27:39.100 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:27:51.034 GMT [1984] LOG:  checkpoint complete: wrote 14 buffers (7.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=11.876 s, sync=0.055 s, total=11.934 s; sync files=21, longest=0.033 s, average=0.003 s; distance=27533 kB, estimate=27995 kB
2022-05-15 15:28:09.042 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:28:21.177 GMT [1984] LOG:  checkpoint complete: wrote 22 buffers (11.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=12.109 s, sync=0.023 s, total=12.135 s; sync files=12, longest=0.021 s, average=0.002 s; distance=27546 kB, estimate=27951 kB
2022-05-15 15:28:39.106 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:28:51.307 GMT [1984] LOG:  checkpoint complete: wrote 23 buffers (11.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=12.151 s, sync=0.046 s, total=12.201 s; sync files=18, longest=0.029 s, average=0.003 s; distance=27604 kB, estimate=27916 kB
2022-05-15 15:29:09.005 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:29:21.448 GMT [1984] LOG:  checkpoint complete: wrote 23 buffers (11.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=12.395 s, sync=0.044 s, total=12.444 s; sync files=22, longest=0.040 s, average=0.002 s; distance=27716 kB, estimate=27896 kB
2022-05-15 15:29:39.062 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:29:51.196 GMT [1984] LOG:  checkpoint complete: wrote 18 buffers (9.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=12.103 s, sync=0.029 s, total=12.135 s; sync files=19, longest=0.026 s, average=0.002 s; distance=27423 kB, estimate=27849 kB
2022-05-15 15:30:09.061 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:30:20.979 GMT [1984] LOG:  checkpoint complete: wrote 23 buffers (11.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=11.870 s, sync=0.046 s, total=11.918 s; sync files=20, longest=0.029 s, average=0.003 s; distance=27492 kB, estimate=27813 kB
2022-05-15 15:30:39.114 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:30:51.601 GMT [1984] LOG:  checkpoint complete: wrote 17 buffers (8.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=12.406 s, sync=0.076 s, total=12.487 s; sync files=23, longest=0.053 s, average=0.004 s; distance=27432 kB, estimate=27775 kB
2022-05-15 15:31:09.089 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:31:21.481 GMT [1984] LOG:  checkpoint complete: wrote 24 buffers (12.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=12.357 s, sync=0.031 s, total=12.392 s; sync files=12, longest=0.029 s, average=0.003 s; distance=27517 kB, estimate=27749 kB
2022-05-15 15:31:39.078 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:31:51.049 GMT [1984] LOG:  checkpoint complete: wrote 21 buffers (10.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=11.941 s, sync=0.028 s, total=11.972 s; sync files=19, longest=0.022 s, average=0.002 s; distance=27633 kB, estimate=27738 kB
2022-05-15 15:32:09.095 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:32:21.309 GMT [1984] LOG:  checkpoint complete: wrote 22 buffers (11.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=12.193 s, sync=0.018 s, total=12.214 s; sync files=13, longest=0.016 s, average=0.002 s; distance=27724 kB, estimate=27736 kB
2022-05-15 15:32:39.525 GMT [1984] LOG:  checkpoint starting: time
2022-05-15 15:32:48.020 GMT [1984] LOG:  checkpoint complete: wrote 41 buffers (20.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=8.440 s, sync=0.052 s, total=8.495 s; sync files=11, longest=0.034 s, average=0.005 s; distance=24895 kB, estimate=27452 kB


#    Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

postgres=# show fsync;
 fsync 
-------
 on
(1 row)

postgres=# alter system set fsync=off;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# show fsync;
 fsync 
-------
 off
(1 row)

root@debian:/opt/bitnami/postgresql/conf# pgbench -c8 -P 60 -T 600 postgres -h 127.0.0.1 -U postgres
pgbench (14.2)
starting vacuum...end.
progress: 60.0 s, 1128.1 tps, lat 7.053 ms stddev 1.990
progress: 120.0 s, 1113.6 tps, lat 7.152 ms stddev 2.043
progress: 180.0 s, 1111.9 tps, lat 7.163 ms stddev 2.058
progress: 240.0 s, 1113.9 tps, lat 7.150 ms stddev 2.056
progress: 300.0 s, 1093.1 tps, lat 7.286 ms stddev 2.074
progress: 360.0 s, 1101.3 tps, lat 7.232 ms stddev 2.067
progress: 420.0 s, 1112.5 tps, lat 7.159 ms stddev 2.067
progress: 480.0 s, 1114.6 tps, lat 7.145 ms stddev 2.087
progress: 540.0 s, 1093.0 tps, lat 7.287 ms stddev 2.101
progress: 600.0 s, 1105.3 tps, lat 7.206 ms stddev 1.932
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 665252
latency average = 7.183 ms
latency stddev = 2.049 ms
initial connection time = 47.054 ms
tps = 1108.813087 (without initial connection time)

Выключение fsync даёт прирост в скорости порядка 100 tps.
Это связано с тем, что postgres не ждёт завершения записи на диск.

#    Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте #несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте #выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

На кластер bitnami поставил классический postgres для удобства и дальше работал с уже с ним.

root@debian:/etc/postgresql/14# pg_createcluster 14 main2
Creating new PostgreSQL cluster 14/main2 ...
/usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/main2 --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/14/main2 ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory               Log file
14  main2   5434 down   postgres /var/lib/postgresql/14/main2 /var/log/postgresql/postgresql-14-main2.log

root@debian:/etc/postgresql/14# pg_checksums -D /var/lib/postgresql/14/main2 -e
Checksum operation completed
Files scanned:  931
Blocks scanned: 3207
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster

root@debian:/home/bitnami# psql -h 127.0.0.1 -U postgres -p 5434
psql (14.2, server 14.3 (Debian 14.3-1.pgdg100+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# create table test (i int);
CREATE TABLE

postgres=# insert into test (i) values (1);
INSERT 0 1
postgres=# insert into test (i) values (2);
INSERT 0 1
postgres=# insert into test (i) values (3);
INSERT 0 1

postgres=# SELECT pg_relation_filepath('test');
 pg_relation_filepath 
----------------------
 base/13707/16384
(1 row)

root@debian:/etc/postgresql/14# pg_ctlcluster stop 14 main2

Изменяем пару байт в файле 16384

root@debian:/etc/postgresql/14# pg_ctlcluster start 14 main2
root@debian:/etc/postgresql/14# pg_lsclusters 
Ver Cluster Port Status Owner    Data directory               Log file
14  main    5433 online postgres /var/lib/postgresql/14/main  /var/log/postgresql/postgresql-14-main.log
14  main2   5434 online postgres /var/lib/postgresql/14/main2 /var/log/postgresql/postgresql-14-main2.log

root@debian:/home/bitnami# psql -h 127.0.0.1 -U postgres -p 5434
psql (14.2, server 14.3 (Debian 14.3-1.pgdg100+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 49424 but expected 824
ERROR:  invalid page in block 0 of relation base/13707/16384

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure 
-------------------------
 off
(1 row)

postgres=# alter system set ignore_checksum_failure=on;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure 
-------------------------
 on
(1 row)

postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 49424 but expected 824
 i 
---
 1
 2
 3
(3 rows)

Данные на месте. Повезло.

root@debian:/var/lib/postgresql/14/main2/base/13707# pg_ctlcluster stop 14 main2

root@debian:/var/lib/postgresql/14/main2/base/13707# pg_checksums -c -D /var/lib/postgresql/14/main2/
pg_checksums: error: checksum verification failed in file "/var/lib/postgresql/14/main2//base/13707/16384", block 0: calculated checksum C110 but block contains 338
Checksum operation completed
Files scanned:  932
Blocks scanned: 3209
Bad checksums:  1
Data checksum version: 1

root@debian:/var/lib/postgresql/14/main2/base/13707# pg_checksums -d -D /var/lib/postgresql/14/main2/
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums disabled in cluster

root@debian:/var/lib/postgresql/14/main2/base/13707# pg_checksums -e -D /var/lib/postgresql/14/main2/
Checksum operation completed
Files scanned:  932
Blocks scanned: 3209
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster

root@debian:/var/lib/postgresql/14/main2/base/13707# pg_ctlcluster start 14 main2

root@debian:/home/bitnami# psql -h 127.0.0.1 -U postgres -p 5434
psql (14.2, server 14.3 (Debian 14.3-1.pgdg100+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# select * from test;
 i 
---
 1
 2
 3
(3 rows)

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure 
-------------------------
 off
(1 row)

Починилась. Значит тоже вариант.
Если бы не починилась, то, возможно, осталась бы только структура.




