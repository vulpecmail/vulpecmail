#сделать в GCE инстанс с Ubuntu 20.04
#поставить на нем Docker Engine

Делал на домашней машине


root@comp2:/home/user# curl -fsSL https://get.docker.com -o get-docker.sh


root@comp2:/home/user# . get-docker.sh
oot@comp2:/home/user# . get-docker.sh
# Executing docker install script, commit: 93d2499759296ac1f9c510605fef85052a2c32be
++ sh -c 'apt-get update -qq >/dev/null'
++ sh -c 'DEBIAN_FRONTEND=noninteractive apt-get install -y -qq apt-transport-https ca-certificates curl >/dev/null'
++ sh -c 'curl -fsSL "https://download.docker.com/linux/ubuntu/gpg" | gpg --dearmor --yes -o /usr/share/keyrings/docker-archive-keyring.gpg'
++ sh -c 'echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu focal stable" > /etc/apt/sources.list.d/docker.list'
++ sh -c 'apt-get update -qq >/dev/null'
++ sh -c 'DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends  docker-ce-cli docker-scan-plugin docker-ce >/dev/null'
++ version_gte 20.10
++ '[' -z '' ']'
++ return 0
++ sh -c 'DEBIAN_FRONTEND=noninteractive apt-get install -y -qq docker-ce-rootless-extras >/dev/null'

================================================================================

To run Docker as a non-privileged user, consider setting up the
Docker daemon in rootless mode for your user:

    dockerd-rootless-setuptool.sh install

Visit https://docs.docker.com/go/rootless/ to learn about rootless mode.


To run the Docker daemon as a fully privileged service, but granting non-root
users access, refer to https://docs.docker.com/go/daemon-access/

WARNING: Access to the remote API on a privileged Docker daemon is equivalent
         to root access on the host. Refer to the 'Docker daemon attack surface'
         documentation for details: https://docs.docker.com/go/attack-surface/

================================================================================

root@comp2:/home/user# usermod -aG docker user



#сделать каталог /var/lib/postgres

root@comp2:/var/lib# mkdir postgres
root@comp2:/var/lib# chown postgres:postgres postgres

root@comp2:/home/user# systemctl start docker

root@comp2:/var/lib# docker network create pg-net
f6de764633a525b56065fdf6b5b9e61101dede38198e994244f26805a88b6d24




#развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres

root@comp2:/var/lib# docker run --name pg-in-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
Unable to find image 'postgres:14' locally
14: Pulling from library/postgres
c229119241af: Pull complete 
3ff4ca332580: Pull complete 
5037f3c12de6: Pull complete 
0444ef779945: Pull complete 
47098a4166e7: Pull complete 
203cca980fab: Pull complete 
a479b6c0e001: Pull complete 
1eaa9abe8ca4: Pull complete 
cad613328fe3: Pull complete 
1ce5087aacfa: Pull complete 
b133d2355caa: Pull complete 
b2694eb85faf: Pull complete 
503b75e1e236: Pull complete 
Digest: sha256:e3d8179786b8f16d066b313f381484a92efb175d1ce8355dc180fee1d5fa70ec
Status: Downloaded newer image for postgres:14
2b58c29dab206469f99847c9cafd111f6ada1fc72542e236b898ef45eb6a478d




#развернуть контейнер с клиентом postgres
#подключится из контейнера с клиентом к контейнеру с сервером и сделать
#таблицу с парой строк

root@comp2:/var/lib# docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-in-docker -U postgres
Password for user postgres: 
psql (14.2 (Debian 14.2-1.pgdg110+1))
Type "help" for help.

postgres=# 

#проверяем

root@comp2:/var/lib# docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
a56a84291f35   postgres:14   "docker-entrypoint.s…"   37 seconds ago   Up 36 seconds   5432/tcp                                    pg-client
2b58c29dab20   postgres:14   "docker-entrypoint.s…"   3 minutes ago    Up 3 minutes    0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-in-docker


postgres=# create table table2 (f1 varchar(20), f2 varchar(20));
CREATE TABLE
postgres=# insert into table2 (f1,f2) values ('s1.1','s1.2');
INSERT 0 1
postgres=# insert into table2 (f1,f2) values ('s2.1','s2.2');
INSERT 0 1

postgres=# select * from table2;
  f1  |  f2  
------+------
 s1.1 | s1.2
 s2.1 | s2.2
(2 rows)

postgres=# \q




#подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP

user@comp:/home$ psql -h 192.168.1.35 -U postgres
Password for user postgres: 
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1), server 14.2 (Debian 14.2-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

postgres=# select * from table2;
  f1  |  f2  
------+------
 s1.1 | s1.2
 s2.1 | s2.2
(2 rows)





#удалить контейнер с сервером

root@comp2:/var/lib# docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
2b58c29dab20   postgres:14   "docker-entrypoint.s…"   15 minutes ago   Up 15 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-in-docker
root@comp2:/var/lib# docker stop 2b58c29dab20
2b58c29dab20

root@comp2:/var/lib# docker rm pg-in-docker
pg-in-docker





#подключится снова из контейнера с клиентом к контейнеру с сервером
#проверить, что данные остались на месте

root@comp2:/var/lib# docker run --name pg-in-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
5c740b0ed224b559ba6fcce1abb30fa11840fda32df93ade68dd833b62276bac


root@comp2:/var/lib# docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-in-docker -U postgres
Password for user postgres: 
psql (14.2 (Debian 14.2-1.pgdg110+1))
Type "help" for help.

postgres=# 

postgres=# select * from table2;
  f1  |  f2  
------+------
 s1.1 | s1.2
 s2.1 | s2.2
(2 rows)


















