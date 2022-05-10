#    создать GCE инстанс типа e2-medium и диском 10GB
#    установить на него PostgreSQL 14 с дефолтными настройками

Всё делалось на виртуальной машине bitnami-postgresql
bitnami@debian:~$ psql -h 127.0.0.1 -U postgres
psql (14.2)
Type "help" for help.

#    применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла

postgres=# select name,setting from pg_settings;

                  name                  |                   setting                    
----------------------------------------+----------------------------------------------
 checkpoint_completion_target           | 0.9
 default_statistics_target              | 500
 effective_cache_size                   | 393216
 effective_io_concurrency               | 2
 maintenance_work_mem                   | 524288
 max_connections                        | 40
 max_wal_size                           | 16384
 min_wal_size                           | 4096
 shared_buffers                         | 131072
 wal_buffers                            | 2048
 work_mem                               | 6553

postgres=# show effective_cache_size;
 effective_cache_size 
----------------------
 3GB
(1 row)

postgres=# show shared_buffers;
 shared_buffers 
----------------
 1GB
(1 row)

postgres=# show wal_buffers;
 wal_buffers 
-------------
 16MB
(1 row)

postgres=# show maintenance_work_mem;
 maintenance_work_mem 
----------------------
 512MB
(1 row)

Вывод select отличается от инструкции show. Show выводит в более читабельном виде.

#    выполнить pgbench -i postgres

root@debian:/home/bitnami# pgbench -i postgres -h 127.0.0.1 -U postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.06 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.37 s (drop tables 0.02 s, create tables 0.03 s, client-side generate 0.19 s, vacuum 0.08 s, primary keys 0.06 s).

postgres=# select name,setting from pg_settings where name like 'autovacuum%';

 autovacuum                            | on
 autovacuum_analyze_scale_factor       | 0.1
 autovacuum_analyze_threshold          | 50
 autovacuum_freeze_max_age             | 200000000
 autovacuum_max_workers                | 3
 autovacuum_multixact_freeze_max_age   | 400000000
 autovacuum_naptime                    | 60
 autovacuum_vacuum_cost_delay          | 2
 autovacuum_vacuum_cost_limit          | -1
 autovacuum_vacuum_insert_scale_factor | 0.2
 autovacuum_vacuum_insert_threshold    | 1000
 autovacuum_vacuum_scale_factor        | 0.2
 autovacuum_vacuum_threshold           | 50
 autovacuum_work_mem                   | -1
(14 rows)

postgres=# select name,setting from pg_settings where name like 'vacuum%';

 vacuum_cost_delay                 | 0
 vacuum_cost_limit                 | 200
 vacuum_cost_page_dirty            | 20
 vacuum_cost_page_hit              | 1
 vacuum_cost_page_miss             | 2
 vacuum_defer_cleanup_age          | 0
 vacuum_failsafe_age               | 1600000000
 vacuum_freeze_min_age             | 50000000
 vacuum_freeze_table_age           | 150000000
 vacuum_multixact_failsafe_age     | 1600000000
 vacuum_multixact_freeze_min_age   | 5000000
 vacuum_multixact_freeze_table_age | 150000000
(12 rows)

#    запустить pgbench -c8 -P 60 -T 600 -U postgres postgres
#    дать отработать до конца
#    дальше настроить autovacuum максимально эффективно
#    построить график по получившимся значениям
#    так чтобы получить максимально ровное значение tps


root@debian:/home/bitnami# pgbench -c8 -P 60 -T 600 postgres -h 127.0.0.1 -U postgres
pgbench (14.2)
starting vacuum...end.
progress: 60.0 s, 1040.3 tps, lat 7.328 ms stddev 2.524
progress: 120.0 s, 980.6 tps, lat 7.709 ms stddev 2.675
progress: 180.0 s, 974.5 tps, lat 7.773 ms stddev 2.964
progress: 240.0 s, 1003.5 tps, lat 7.512 ms stddev 2.748
progress: 300.0 s, 986.9 tps, lat 7.521 ms stddev 2.819
progress: 360.0 s, 992.2 tps, lat 7.540 ms stddev 2.799
progress: 420.0 s, 987.3 tps, lat 7.756 ms stddev 2.648
progress: 480.0 s, 998.3 tps, lat 7.516 ms stddev 2.744
progress: 540.0 s, 971.1 tps, lat 7.817 ms stddev 2.713
progress: 600.0 s, 1009.2 tps, lat 7.393 ms stddev 2.738
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 596638
latency average = 7.584 ms
latency stddev = 2.743 ms
initial connection time = 39.405 ms
tps = 994.423193 (without initial connection time)

Таблица зависимости tps от autovacuum_max_workers:
Количество Workers/Среднее значение tps
2 1024.397473
3 1047.351118
4 1050.147374
5 1037.538913
6 1030.677125
8 1034.843439
10 1034.979089

Таблица зависимости tps от autovacuum_vacuum_threshold:
25 1035.593424
35 1035.989758
50 1066.896613
75 1056.573259

autovacuum_vacuum_cost_limit = 1000, 500, 250
 tps = 1060.800690 (without initial connection time)
 tps = 1037.913252 (without initial connection time)
 tps = 1022.766575 (without initial connection time)
 
 postgres=# select name,setting from pg_settings where name like 'autovacuum%';
                 name                  |  setting  
---------------------------------------+-----------
 autovacuum                            | on
 autovacuum_analyze_scale_factor       | 0.1
 autovacuum_analyze_threshold          | 50
 autovacuum_freeze_max_age             | 200000000
 autovacuum_max_workers                | 10
 autovacuum_multixact_freeze_max_age   | 400000000
 autovacuum_naptime                    | 60
 autovacuum_vacuum_cost_delay          | 10
 autovacuum_vacuum_cost_limit          | 1000
 autovacuum_vacuum_insert_scale_factor | 0.2
 autovacuum_vacuum_insert_threshold    | 1000
 autovacuum_vacuum_scale_factor        | 0.05
 autovacuum_vacuum_threshold           | 25


root@debian:/opt/bitnami# pgbench -c8 -P 60 -T 600 postgres -h 127.0.0.1 -U postgres
pgbench (14.2)
starting vacuum...end.
progress: 60.0 s, 1054.9 tps, lat 7.055 ms stddev 2.517
progress: 120.0 s, 1038.6 tps, lat 7.457 ms stddev 2.236
progress: 180.0 s, 1073.2 tps, lat 6.745 ms stddev 2.722
progress: 240.0 s, 1095.0 tps, lat 6.254 ms stddev 2.877
progress: 300.0 s, 1053.6 tps, lat 7.071 ms stddev 2.591
progress: 360.0 s, 1035.9 tps, lat 7.309 ms stddev 2.607
progress: 420.0 s, 1089.5 tps, lat 6.359 ms stddev 2.940
progress: 480.0 s, 1078.0 tps, lat 6.640 ms stddev 2.777
progress: 540.0 s, 1059.3 tps, lat 7.035 ms stddev 2.601
progress: 600.0 s, 1029.2 tps, lat 7.216 ms stddev 2.767
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 636450
latency average = 6.907 ms
latency stddev = 2.700 ms
initial connection time = 50.868 ms
tps = 1060.800690 (without initial connection time)


