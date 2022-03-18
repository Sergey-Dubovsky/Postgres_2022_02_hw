1. Уcтановил локальную виртуалку с Ubuntu LTS 20.04.04 
1. Зарегался в GCP 
2. В локальную виртуалку поставил gcloud и авторизовался в нем через браузер

[инструкция по установке gcloud](https://geekflare.com/gcloud-installation-guide/#anchor-debian-ubuntu)

```bash
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

sudo apt-get install apt-transport-https ca-certificates gnupg

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -

sudo apt-get update && sudo apt-get 
install google-cloud-sdk

sudo apt-get install google-cloud-sdk-app-engine-java

gcloud init
```

4. Создал проект postgres2022-19730918, добавил  ifti@yandex.ru с ролью Project Editor 
5. Включил Compute Engine API, создал инстанс

2 vCPU 4 Mb RAM
Name    postgresql-1
Type    New SSD persistent disk
Size    10 GB 
ImageUbuntu 20.04 LTS 

```
gcloud compute instances create postgresql-1 --project=postgres2022-19730918 --zone=europe-north1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=566999235043-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=postgresql-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19730918/zones/europe-north1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```
6. Проверил список инстансов , старт-стоп
```
gcloud compute instances list
gcloud compute instances stop postgresql-1
gcloud compute instances start postgresql-1
```

7. В локальной виртуалке создал SSH ключ, и загрузил в GCS

```
ssh-keygen -t rsa
cat .ssh/id_rsa.pub
-- загрузил в GoogleCloud

eval `ssh-agent -s`
ssh-add .ssh/id_rsa
```
8. Подключился к вируталке
```
ssh serg@35.228.91.58
```
9. Установил postgres
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
```

10. в другой вкладке подключился еще раз в инстансу

11. в обеих сессиях запустил psql из под юзера postgres
```
sudo -u postgres psql
```

12. отключаем Autocommit и создаем тестовую таблицу
```
\set AUTOCOMMIT off
postgres=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
postgres=# select * from persons;commit;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

COMMIT

```
13. В 1й сессии проверяем уровень изоляции - read commited
```
postgres=# show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
(1 row)

postgres=# begin;
BEGIN
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
postgres=*# 
```
во 2й сессии
```
postgres=# select * from persons;commit;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

COMMIT
```
В 1й сессии
```
postgres=*# commit;
COMMIT
```
во 2й сессии
```
postgres=# select * from persons;commit;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)

COMMIT
```
Пока транзакция в первой сессии не завершена, добавленные данные не видны во второй сессии

14. В 1й сессии начинем транзакцию repeatable read и добавляем запись
```
postgres=# begin transaction isolation level repeatable read;
BEGIN
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
postgres=*# commit;
COMMIT
```

во 2й сессии
```
postgres=# begin transaction isolation level repeatable read;
BEGIN
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)

postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)

postgres=*# commit;
COMMIT
postgres=# begin;
BEGIN
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
  7 | sveta      | svetova
(4 rows)

postgres=*# commit;
COMMIT
```
Во второй сессии добавленная запись становится видна толко после завершени транзакции в первой и во второй сессии. То есть пока транзакция во второй сессии, начатая до добавлени данных, не будет завершена, во второй сессии не цдастся увидеть добавленную запись.

