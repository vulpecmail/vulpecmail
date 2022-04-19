#создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a
#поставьте на нее PostgreSQL 14 через sudo apt
#проверьте что кластер запущен через sudo -u postgres pg_lsclusters

Postgres Был установлен при выполнении предыдущих ДЗ


root@comp2:/home/user# apt-get install postgresql
Reading package lists... Done
Building dependency tree       
Reading state information... Done
postgresql is already the newest version (12+214ubuntu0.1).
The following packages were automatically installed and are no longer required:
  libbabeltrace1 libc6-dbg libdw1 libfwupdplugin1
Use 'sudo apt autoremove' to remove them.
0 upgraded, 0 newly installed, 0 to remove and 27 not upgraded.

user@comp2:~$ sudo su postgres
postgres@comp2:/home/user$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
postgres@comp2:/home/user$



#зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым postgres=# create table test(c1 text); postgres=# insert into test #values('1'); \q

postgres@comp2:/home/user$ psql -h 127.0.0.1
Password for user postgres: 
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# create table test(c1 text); insert into test values('1');
CREATE TABLE
INSERT 0 1
postgres=# select * from test;
 c1 
----
 1
(1 row)

postgres=# \q
postgres@comp2:/home/user$




#остановите postgres например через sudo -u postgres pg_ctlcluster 14 main stop

postgres@comp2:/home/user$ pg_ctlcluster stop 12 main
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@12-main
postgres@comp2:/home/user$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 down   postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
postgres@comp2:/home/user$ 



#создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB
#добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
#проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/#sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
#перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)

Подключил внешний USB-диск (док-станция). В fstab прописывать не стал, т.к. съемные носители лучше не прописывать автомаунтом.

root@comp2:/mnt# fdisk -l
...
Disk /dev/sda: 3,65 TiB, 4000787030016 bytes, 7814037168 sectors
Disk model: EFAX-68JH4N1    
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 2A86BB3C-4D63-4AE0-9633-C90C2565AFE8

Device     Start        End    Sectors  Size Type
/dev/sda1   2048 7814035455 7814033408  3,7T Linux filesystem

root@comp2:/mnt# mount -t ext4 /dev/sda1 /mnt/ext_disk

root@comp2:/mnt# mount
...
/dev/sda1 on /mnt/ext_disk type ext4 (rw,relatime)




#сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
#перенесите содержимое /var/lib/postgres/14 в /mnt/data - mv /var/lib/postgresql/14 /mnt/data

На /mnt/ext_disk поставил режим доступа 777.

postgres@comp2:/home/user$ cd /mnt/ext_disk/
postgres@comp2:/mnt/ext_disk$ mkdir postgres
postgres@comp2:/mnt/ext_disk$ mv /var/lib/postgresql/12 /mnt/ext_disk/postgres




#попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start
#напишите получилось или нет и почему
#
#задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/14/main который надо поменять и поменяйте его
#напишите что и почему поменяли

Не запустится, т.к. в конфиге прописан другой путь.
Сразу правим postgresql.conf.

root@comp2:/etc/postgresql/12/main# cat postgresql.conf |grep data_dir
#data_directory = '/var/lib/postgresql/12/main'		# use data in another directory
data_directory = '/mnt/ext_disk/postgres/12/main'		# use data in another directory



#попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start
#напишите получилось или нет и почему

postgres@comp2:/mnt/ext_disk$ pg_ctlcluster start 12 main
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@12-main
postgres@comp2:/mnt/ext_disk$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory                 Log file
12  main    5432 online postgres /mnt/ext_disk/postgres/12/main /var/log/postgresql/postgresql-12-main.log

Выставили правильный  путь и кластер запустился без проблем.




#зайдите через через psql и проверьте содержимое ранее созданной таблицы

postgres@comp2:/mnt/ext_disk$ psql -h 127.0.0.1
Password for user postgres: 
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | test | table | postgres
(1 row)

postgres=# select * from test;
 c1 
----
 1
(1 row)




#задание со звездочкой *: не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте #внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, #расскажите как вы это сделали и что в итоге получилось.

postgres@comp2:/mnt/ext_disk$ pg_createcluster 12 main2
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

Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Ver Cluster Port Status Owner    Data directory               Log file
12  main2   5433 down   postgres /var/lib/postgresql/12/main2 /var/log/postgresql/postgresql-12-main2.log

postgres@comp2:/mnt/ext_disk$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory                 Log file
12  main    5432 online postgres /mnt/ext_disk/postgres/12/main /var/log/postgresql/postgresql-12-main.log
12  main2   5433 down   postgres /var/lib/postgresql/12/main2   /var/log/postgresql/postgresql-12-main2.log
postgres@comp2:/mnt/ext_disk$ pg_ctlcluster stop 12 main
  sudo systemctl stop postgresql@12-main
postgres@comp2:/mnt/ext_disk$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory                 Log file
12  main    5432 down   postgres /mnt/ext_disk/postgres/12/main /var/log/postgresql/postgresql-12-main.log
12  main2   5433 down   postgres /var/lib/postgresql/12/main2   /var/log/postgresql/postgresql-12-main2.log
postgres@comp2:/mnt/ext_disk$ 

По пути /etc/postgresql/12/main2 правим файл postgresql.conf (инструкция data_directory)

postgres@comp2:/mnt/ext_disk$ pg_ctlcluster start 12 main2
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@12-main2
postgres@comp2:/mnt/ext_disk$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory                 Log file
12  main    5432 down   postgres /mnt/ext_disk/postgres/12/main /var/log/postgresql/postgresql-12-main.log
12  main2   5433 online postgres /mnt/ext_disk/postgres/12/main /var/log/postgresql/postgresql-12-main2.log

postgres@comp2:/mnt/ext_disk$ psql -h 127.0.0.1 -p 5433
Password for user postgres: 
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# select * from test;
 c1 
----
 1
(1 row)

postgres=# 








