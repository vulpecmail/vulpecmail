# ДЗ по лекции 2.

#создать новый проект в Google Cloud Platform, 
#например postgres2022-, где yyyymmdd год, месяц и день вашего рождения 
#(имя проекта должно быть уникально на уровне GCP) 
#дать возможность доступа к этому проекту пользователю ifti@yandex.ru 
#с ролью Project Editor 
#далее создать инстанс виртуальной машины Compute Engine с дефолтными параметрами
#добавить свой ssh ключ в GCE metadata зайти удаленным ssh (первая сессия), 
#не забывайте про ssh-add

Создана виртуальная машина Ubuntu в Яндексе (cloud-vulpecmail/default) 
с именем postgres. Дан доступ пользователю ifti@yandex.ru с ролью editor 
(идентификатор aje38a4hci4d6p9582hf). Добавлен ssh-ключ.

#зайти удаленным ssh (первая сессия), не забывайте про ssh-add

user@comph:~$ ssh dvkalinkin@51.250.106.22
The authenticity of host '51.250.106.22 (51.250.106.22)' can't be established.
ECDSA key fingerprint is SHA256:wY+/tnQ3aNomYq5xFjLsfiiFFiAEyuDlABxp1whHtkI.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '51.250.106.22' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-105-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Sat Mar 26 19:36:41 2022 from 188.243.182.103
dvkalinkin@postgres:~$ 

#поставить PostgreSQL

dvkalinkin@postgres:~$ sudo apt list postgresql -a
Listing... Done
postgresql/focal-updates,focal-updates,focal-security,focal-security 12+214ubuntu0.1 all
postgresql/focal,focal 12+214 all

dvkalinkin@postgres:~$ sudo apt install postgresql
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following packages were automatically installed and are no longer required:
  linux-headers-5.4.0-42 linux-headers-5.4.0-42-generic
  linux-image-5.4.0-42-generic linux-modules-5.4.0-42-generic
  linux-modules-extra-5.4.0-42-generic
Use 'sudo apt autoremove' to remove them.
Suggested packages:
  postgresql-doc
The following NEW packages will be installed:
  postgresql
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 3,924 B of archives.
After this operation, 67.6 kB of additional disk space will be used.
Get:1 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 postgresql all 12+214ubuntu0.1 [3,924 B]
Fetched 3,924 B in 0s (180 kB/s)       
Selecting previously unselected package postgresql.
(Reading database ... 143035 files and directories currently installed.)
Preparing to unpack .../postgresql_12+214ubuntu0.1_all.deb ...
Unpacking postgresql (12+214ubuntu0.1) ...
Setting up postgresql (12+214ubuntu0.1) ...
dvkalinkin@postgres:~$ 

#зайти вторым ssh (вторая сессия)

user@comph:~$ ssh dvkalinkin@51.250.106.22
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-105-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Sun Mar 27 08:43:40 2022 from 188.243.182.103
dvkalinkin@postgres:~$ 

#запустить везде psql из под пользователя postgres

dvkalinkin@postgres:~$ psql -U postgres -h 127.0.0.1
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# 

#выключить auto commit

postgres=# \set AUTOCOMMIT OFF

#сделать в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, #second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit; 

postgres=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
postgres=# insert into persons(first_name, second_name) values('ivan', 'ivanov');
INSERT 0 1
postgres=# insert into persons(first_name, second_name) values('petr', 'petrov');
INSERT 0 1
postgres=# commit;
COMMIT
postgres=#

#посмотреть текущий уровень изоляции: show transaction isolation level 

postgres=# show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
(1 row)

#начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции 
#в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev'); 

postgres=# insert into persons(first_name, second_name) values('sergey', 'sergeev'); 
INSERT 0 1

#сделать select * from persons во второй сессии 

postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

#видите ли вы новую запись и если да то почему? 
Записи нет.

#завершить первую транзакцию - commit; 

postgres=# commit;
COMMIT

#сделать select * from persons во второй сессии 

postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

#видите ли вы новую запись и если да то почему? 
#Новая запись появилась, т.к. транзакция по вставке записи успешно завершена, данные зафиксированы.

#завершите транзакцию во второй сессии 

postgres=# commit;
COMMIT

#начать новые но уже repeatable read транзации - set transaction isolation level repeatable read; 

postgres=# \set transaction isolation level repeatable read;

#в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
#сделать select * from persons во второй сессии 
#видите ли вы новую запись и если да то почему? 

Новой записи нет.

#завершить первую транзакцию - commit; 
#сделать select * from persons во второй сессии 
#видите ли вы новую запись и если да то почему? 

Новой записи нет. В режиме Repeatable Read видны только те данные, которые были зафиксированы до начала транзакции.

#завершить вторую транзакцию 
#сделать select * from persons во второй сессии 
#видите ли вы новую запись и если да то почему?

Новая запись появилась, т.к. транзакция завершена и можно получить доступ к обновлённым данным.




