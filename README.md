# 目的
Docker Volumeを使ったバックアップハンズオン

# 前提
| ソフトウェア     | バージョン    | 備考         |
|:---------------|:-------------|:------------|
| vagrant        | 1.7.4        |             |
| docker         | 1.10.3       |             |
| docker-compose | 1.6.2        |             |


# 構成

+ セットアップ
+ コンテナ内部のデータ管理
+ DockerComposeを使ったコンテナ内部のデータ管理
+ 後片付け
+ 参照

# 詳細
## セットアップ
### 実行環境起動
```
$ vagrant up
$ vagrant ssh
$ cd /vagrant/
```

## コンテナ内部のデータ管理
### データボリューム
#### データボリュームを追加する
```
$ docker run -d -p 5000:5000 --name web -v /webapp training/webapp python app.py
```

#### ボリュームの配置
```
$ docker inspect web
・・・
        },
        "Mounts": [
            {
                "Name": "137a2a98095cb60112afc9aa7114ab154958da309c0ddc737cf0a3cc56ba8264",
                "Source": "/var/lib/docker/volumes/137a2a98095cb60112afc9aa7114ab154958da309c0ddc737cf0a3cc56ba8264/_data",
                "Destination": "/webapp",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],

・・・
```
#### ホストディレクトリにデータボリュームとしてマウントする
```
$ docker rm -f web
$ docker run -d -p 5000:5000 --name web -v $(pwd)/src/webapp:/opt/webapp training/webapp python app.py
```
#### ホストのファイルをデータボリュームとしてマウントする
```
$ docker run --rm -it -v ~/.bash_history:/root/.bash_history ubuntu /bin/bash
```

### データボリュームコンテンを作成してマウントする
```
$ docker create -v /dbdata --name pg_dbstore postgres /bin/true
$ docker run -d --volumes-from pg_dbstore --name pg_db1 postgres
$ docker exec -it pg_db1 psql -U postgres
postgres=# \d
No relations found.
postgres=# create table test ( shainno int,shimei text);
CREATE TABLE
postgres=# \d
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)
postgres=# \q
$ docker run -d --volumes-from pg_dbstore --name pg_db2 postgres
$ docker exec -it pg_db2 psql -U postgres
psql (9.5.3)
Type "help" for help.

postgres=# \d
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)

postgres=# \q
$ docker run -d --volumes-from pg_dbstore --name pg_db2 postgres
$ docker exec -it pg_db2 psql -U postgres
psql (9.5.3)
Type "help" for help.

postgres=# \d
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)

postgres=# \q
```

```
$ docker create -v /dbdata --name my_dbstore mysql /bin/true
$ docker run -d --volumes-from my_dbstore --name my_db1  -e MYSQL_ROOT_PASSWORD=my-secret-pw  mysql
$ docker exec -it my_db1 mysql -uroot -pmy-secret-pw
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.12 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database test;
Query OK, 1 row affected (0.00 sec)

mysql> use test;
Database changed
mysql> create table test ( shainno int,shimei text);
Query OK, 0 rows affected (0.01 sec)

mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| test           |
+----------------+
1 row in set (0.00 sec)

mysql> exit
Bye
$ docker stop my_db1
$ docker run -d --volumes-from my_dbstore --name my_db2  -e MYSQL_ROOT_PASSWORD=my-secret-pw  mysql
$ docker exec -it my_db2 mysql -uroot -pmy-secret-pw
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.12 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.01 sec)

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| test           |
+----------------+
1 row in set (0.00 sec)

mysql> exit
Bye
```

### バックアップ、リストアまたはデータボリュームの移行
```
$ docker stop my_db2
$ docker start my_db1
$ docker exec my_db1 sh -c 'mysqldump -uroot -pmy-secret-pw -A > /dbdata/all.dump'
$ docker run --rm --volumes-from my_dbstore -v $(pwd)/backup:/backup ubuntu tar cvf /backup/my_backup.tar /dbdata/all.dump
$ docker run -v /dbdata --name my_dbstore2 ubuntu /bin/bash
$ docker run --rm --volumes-from my_dbstore2 -v $(pwd)/backup:/backup ubuntu bash -c "cd /dbdata && tar xvf /backup/my_backup.tar --strip 1"
$ docker stop my_db1
$ docker run -d --volumes-from my_dbstore2 --name my_db3 -e MYSQL_ROOT_PASSWORD=my-secret-pw  mysql
$ docker exec my_db3 sh -c 'mysql -uroot -pmy-secret-pw < /dbdata/all.dump'
$ docker exec -it my_db3 mysql -uroot -pmy-secret-pw
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.12 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.00 sec)
```

### ボリュームの削除
```
$ docker rm -f $(docker ps -q -a)
$ docker volume rm $(docker volume ls -q)
```

## DockerComposeを使ったコンテナ内部のデータ管理

### データボリュームコンテンを作成してマウントする
```
$ docker volume create --name=my_dbstore
$ docker volume create --name=my_dbstore2
$ docker volume ls
DRIVER              VOLUME NAME
local               my_dbstore
local               my_dbstore2
$ docker-compose run my_dba1
Creating vagrant_my_db1_1
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 2003 (HY000): Can't connect to MySQL server on 'my_db1' (111)
vagrant@ubuntu-trusty64-ja:/vagrant$ docker-compose run my_dba1
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.12 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database test;
Query OK, 1 row affected (0.00 sec)

mysql> use test;
Database changed
mysql> create table test ( shainno int,shimei text);
Query OK, 0 rows affected (0.01 sec)

mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| test           |
+----------------+
1 row in set (0.00 sec)

mysql> exit
Bye
$ docker-compose stop my_db1
```

```
$ docker-compose run my_dba2
Creating vagrant_my_db2_1
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.12 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.01 sec)

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| test           |
+----------------+
1 row in set (0.00 sec)

mysql> exit
Bye
$ docker-compose stop my_db2
```

### バックアップ、リストアまたはデータボリュームの移行
```
$ docker-compose run my_dbdata_export
$ docker-compose run my_dba3
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.12 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> exit
Bye
$ docker-compose run my_dbdata_import
$ docker-compose run my_dba3
$ docker-compose run my_dba3
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.12 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.00 sec)

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| test           |
+----------------+
1 row in set (0.00 sec)

mysql> exit
Bye
```

## 後片付け
```
$ exit
$ vagrant destroy
```

# 参照
+ [Manage data in containers](https://docs.docker.com/engine/userguide/containers/dockervolumes/)
+ [Dockerfile reference](https://docs.docker.com/engine/reference/builder/#cmd)
+ [Docker-compose.yml: from V1 to V2](https://medium.com/@giorgioto/docker-compose-yml-from-v1-to-v2-3c0f8bb7a48e#.ee5uwvz7e)
+ [DockerHub mysql](https://hub.docker.com/_/mysql/)
+ [DockerHub postgres](https://hub.docker.com/_/postgres/)