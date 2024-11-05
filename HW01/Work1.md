# Создание виртуальной машины: 

> **Cоздать новый проект в Яндекс облако или на любых ВМ, например postgres2024-, где yyyymmdd год, месяц и день вашего рождения (имя проекта должно быть уникально)
далее создать инстанс виртуальной машины с дефолтными параметрами - 1-2 ядра, 2-4Гб памяти, любой линукс, на курсе Ubuntu 100%**

Создал ВМ Ubuntu 20.04 в VirtualBox с параметрами - 2 ядра, 2Гб памяти, настроил сеть для доступа с хоста и выход в интернет

>  **Добавить свой ssh ключ**

создал ключ ssh используя <u>PowerShell</u>
```sh
ssh-keygen -t rsa -b 2048
```
Для подключения использую <u>MobaXterm</u>
Подключился и скопировал  public key на ВМ и добавил ключ в uthorized_keys
```sh
cat id-rsa.pub >> ~/.ssh/authorized_keys
```
> **Установка Postgres**
```sh
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc
```

> **Установить пароль для Postgres**
```sh
sudo -u postgres psql
\password   #12345
\q
```

> **Добавить сетевые правила для подключения к Postgres**
```sh
cd /etc/postgresql/16/main/
sudo nano /etc/postgresql/17/main/postgresql.conf
#listen_addresses = 'localhost'
listen_addresses = '*'

sudo nano /etc/postgresql/17/main/pg_hba.conf
#host    all             all             127.0.0.1/32            scram-sha-256 password
host    all             all             0.0.0.0/0               scram-sha-256 
```

```sh
sudo systemctl status postgresql@17-main.service
● postgresql@17-main.service - PostgreSQL Cluster 17-main
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: active (running) since Sat 2024-11-02 05:26:16 UTC; 42s ago
    Process: 4507 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 17-main start (code=exited, status=0/SUCCESS)
   Main PID: 4513 (postgres)
      Tasks: 6 (limit: 2276)
     Memory: 19.5M (peak: 27.0M)
        CPU: 138ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@17-main.service
             ├─4513 /usr/lib/postgresql/17/bin/postgres -D /var/lib/postgresql/17/main -c config_file=/etc/postgresql/17/main/postgresql.conf
             ├─4514 "postgres: 17/main: checkpointer "
             ├─4515 "postgres: 17/main: background writer "
             ├─4517 "postgres: 17/main: walwriter "
             ├─4518 "postgres: 17/main: autovacuum launcher "
             └─4519 "postgres: 17/main: logical replication launcher "

ноя 02 05:26:14 ubuntu systemd[1]: Starting postgresql@17-main.service - PostgreSQL Cluster 17-main...
ноя 02 05:26:16 ubuntu systemd[1]: Started postgresql@17-main.service - PostgreSQL Cluster 17-main.
```

> **Подключение к Postgres**

psql 
```sh
postgres=# \l
                                                       Список баз данных
    Имя    | Владелец | Кодировка | Провайдер локали | LC_COLLATE  |  LC_CTYPE   | Локаль | Правила ICU |     Права доступа
-----------+----------+-----------+------------------+-------------+-------------+--------+-------------+-----------------------
 postgres  | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |        |             |
 template0 | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |        |             | =c/postgres          +
           |          |           |                  |             |             |        |             | postgres=CTc/postgres
 template1 | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |        |             | =c/postgres          +
           |          |           |                  |             |             |        |             | postgres=CTc/postgres
```
> **Выключить auto commit**
```sh
postgres=# \set AUTOCOMMIT OFF
```
# Уровни изоляции транзакций
## создадим табличку для тестов

> Cделать в первой сессии новую таблицу и наполнить ее данными
    create table persons(id serial, first_name text, second_name text);
    insert into persons(first_name, second_name) values('ivan', 'ivanov');
    insert into persons(first_name, second_name) values('petr', 'petrov');
    commit;
```sh
postgres=# create table persons(id serial, first_name text, second_name text);
postgres=# insert into persons(first_name, second_name) values('ivan', 'ivanov');
postgres=# insert into persons(first_name, second_name) values('petr', 'petrov');
postgres=# commit;

postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
```
> **Посмотреть текущий уровень изоляции: show transaction isolation level**
```sh
postgres=*# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
```

> **Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
    в первой сессии добавить новую запись
    insert into persons(first_name, second_name) values('sergey', 'sergeev');
    сделать select * from persons во второй сессии. Видите ли вы новую запись и если да то почему?**

#### 1 console
```sh
postgres=# begin;
BEGIN
postgres=# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
```
#### 2 console
```sh
postgres=*# begin;
BEGIN
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
```
<span style='color: green ;'>*вторая транзакция не видит незафиксированные изменения т.к. грязное чтение не допускается*</span>

> **Завершить первую транзакцию - commit;
сделать select * from persons во второй сессии
видите ли вы новую запись и если да то почему?
завершите транзакцию во второй сессии**
 
 #### 1 console
 ```sh
 postgres=# commit;
```

 #### 2 console
 ```sh
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
postgres=*# commit;
```
 <span style='color: green ;'>*Запрос во 2 сессии получает уже новые данные т.к. есть аномалия неповторяющегося чтения, которая допускается на уровне Read Committed*</span>


>    **Начать новые но уже repeatable read транзакции - set transaction isolation level repeatable read;
    в первой сессии добавить новую запись
    insert into persons(first_name, second_name) values('sveta', 'svetova');
    сделать select * from persons во второй сессии
    видите ли вы новую запись и если да то почему?**

### 1 console
 ```sh
 postgres=# begin;
BEGIN
postgres=# set transaction isolation level repeatable read;
postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 repeatable read

postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1

postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 строки)
 ```
 ### 2 console
 ```sh
 postgres=# begin;
BEGIN
postgres=# set transaction isolation level repeatable read;
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
 ```
 <span style='color: green ;'>*Запрос во 2 сессии получает cтарые данные т.к. режиме Repeatable Read видны только те данные, которые были зафиксированы до начала транзакции *</span>


 >  **Завершить первую транзакцию - commit;
    сделать select * from persons во второй сессии
    видите ли вы новую запись и если да то почему?**

 #### 1 console
  ```sh
postgres=# commit;
  ```
 #### 2 console 
  ```sh   
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
 ```
<span style='color: green ;'>*Запрос во 2 сессии получает cтарые данные т.к. режиме Repeatable Read видны только те данные, которые были зафиксированы до начала транзакции*</span>

 >   **Завершить вторую транзакцию
    сделать select * from persons во второй сессии
    видите ли вы новую запись и если да то почему?**

### 2 console  
 ```sh 
postgres=*# commit; 
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 строки)
 ```

<span style='color: green ;'>*Запрос во 2 сессии получает новые данные т.к. начинается новая транзакция, а в режиме Repeatable Read видны только те данные, которые были зафиксированы до начала транзакции*</span>