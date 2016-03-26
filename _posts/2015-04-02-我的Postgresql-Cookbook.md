---
layout: post
title: 我的 Postgresql Cookbook
---


## 创建用户

`createuser scene -h localhost`

`createuser pgsql -S -l -P -d` 非超级用户，可登陆，需要密码，可创建数据库

`createuser og_admin -S -l -P -d`

## 创建数据库

`createdb -h localhost scene_development -O scene`

此命令会将数据库scene_development的所有权赋给用户scene

### 创建一个可以登陆和创建数据库的非超级用户

`create role pg_user login createdb`
`createuser -d -l -w -S -R pguser`    创建一个可以创建数据库，可以登陆，无密码的非超级用户,不可以创建role


## 连接数据库

`psql -h localhost -U scene -d scene_development`

`psql -h localhost -d palottery -U palottery -p 6433`

## iptable with postgresql

~~~bash
sudo iptables -A INPUT -p tcp -s 0/0 --sport 1024:65535 -d 10.61.0.103 --dport 5432 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp -s 10.61.0.103 --sport 5432 -d 0/0 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT
~~~

参考1 http://www.cyberciti.biz/tips/howto-iptables-postgresql-open-port.html

参考2 http://www.cyberciti.biz/tips/postgres-allow-remote-access-tcp-connection.html

## pg_dump

###  备份整个数据库

`pg_dump -h qa.fun-guide.mobi -U pgsql huafei_development > huafeibak.out`

`pg_dump -h localhost -U pgsql fchk_staging -p 6543 > fchk_staging_20131108.sql`

### 只备份数据

`pg_dump -h qa.fun-guide.mobi -U pgsql huafei_development -a > huafeibak-data-only.out`

### 导入数据

`psql -h localhost -U pgsql -d huafei_development < huafeibak-data-only.out`


## 用socket连接，与rails项目结合

~~~bash

mkdir /var/pgsql_socket

sudo chmod go=xwr /var/pgsql_socket

~~~


vi ~/pg_data/postgresql.conf, 设置,

~~~
unix_socket_directory = '/var/pgsql_socket'
重启postgresql
database.yml文件内容
development:
  adapter:  postgresql
  encoding: unicode
  database: lot_channels_development
  pool:     5
  username: lot_channels
  password:
~~~

创建数据库用户,

~~~
createuser -d -l -W -S lot_channels
~~~

## 改变表的所有者

`ALTER TABLE schema_migrations OWNER TO pgsql`

## pg开发环境搭建

`create database`

~~~
initdb /usr/local/var/postgres -E utf8
~~~

- 启动database server

~~~
postgres -D /usr/local/var/postgres
pg_ctl -D /usr/local/var/postgres -l logfile start
~~~


## 启动,重加载,停止

`pg_ctl -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log start`
`pg_ctl -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log reload`
`pg_ctl -D /usr/local/var/postgres stop`

## UTF8编码问题

PG::Error: ERROR:  encoding UTF8 does not match locale en_US

http://stackoverflow.com/questions/13115692/encoding-utf8-does-not-match-locale-en-us-the-chosen-lc-ctype-setting-requires

`sudo su postgres`

psql 登录到数据库

`pdate pg_database set datistemplate=false where datname='template1'`

`drop database Template1`

`create database template1 with owner=postgres encoding='UTF-8' lc_collate='en_US.utf8' lc_ctype='en_US.utf8' template template0`


## add column 实际例子

~~~
ALTER TABLE company_messages
      ADD COLUMN start_time timestamp without time zone,
      ADD COLUMN end_time timestamp without time zone,
      ADD COLUMN message_type integer NOT NULL DEFAULT 0;
~~~	  


## pg_dump schema

table 从属于 schema,

=> \dt orders 

完整的意思是:

=> \dt public.orders

public 就是一个 schema

比如我们数据库中有一个 autoship 的 schema, 我们列出看其下的 tables, 可以这样操作:

=> \dt autoship.*;


现在我们需要将 autoship 这个 schema 备份到本地，可以这样操作:

### 只备份 schema, 不包含数据

`pd_dump -s -n autoship DB_NAME -f autoship_schema.sql`


### 只包含数据

`pd_dump -a -n autoship DB_NAME -f autoship_schema.sql`


### 只备份其中一个表, 比如 autoship.orders

1. 只有数据，不包含表结构:  `pd_dump -a -t autoship.orders DB_NAME -f autoship_schema.sql`

2. 只有表结构，不包含数据: `pd_dump -s -t autoship.orders DB_NAME -f autoship_schema.sql`

3. 表结构和数据都包含: `pd_dump -t autoship.orders DB_NAME -f autoship_schema.sql`


### 完整的备份 schema 过程

~~~
$ pg_dump -s -n public my_db -h my_host -U my_user -f my_schema.sql 
$ createdb -h localhost my_db
$ psql -d my_db < my_schema.sql
~~~

### 只备份某个表的数据，然后将数据倒入到本地数据库中

~~~
$ pg_dump -a -t public.my_table my_db -h my_host -U my_user -f my_table.sql
$ psql my_db < my_table.sql
~~~

### 备份多个表的数据

~~~
$ pg_dump  -a -t my_table1 -t my_table2 my_db -h my_host -U my_user -f my_tables.sql
$ psql my_db -f my_tables.sql
~~~

## 创建 table

~~~sql
create table user_bank_informations
(
  id serial primary key,
  user_id integer NOT NULL,
  account_name character varying(255) NOT NULL,
  account_address character varying(255) NOT NULL,
  account_number character varying(255) NOT NULL,
  bank_name character varying(255) NOT NULL,  
  bank_code character varying(255) NOT NULL,
  bank_address character varying(255) NOT NULL,
  swift_code character varying(255) NOT NULL,
  id_docs_received boolean default false NOT NULL,
  created_at timestamp without time zone,
  updated_at timestamp without time zone
)
~~~

~~~sql
create table accounts
(
  id serial primary key,
  owner character(255) NOT NULL,
  balance numeric(9, 2)
)
~~~

## 插入数据

~~~sql
insert into accounts (owner, balance) values ('Bob', 100);
insert into accounts (owner, balance) values ('Mary', 200);
~~~

## PL/pgSQL 编程

### create a function


~~~sql
CREATE OR REPLACE FUNCTION transfer(
  i_payer text,
  i_recipient text,
  i_amount numeric(15,2)
)
RETURNS text
AS
$$
DECLARE
  payer_bal numeric;
BEGIN

  SELECT balance INTO payer_bal
    FROM accounts
  WHERE owner = i_payer FOR UPDATE;
  IF NOT FOUND THEN
    RETURN 'Payer account not found';
  END IF;

  IF payer_bal < i_amount THEN
    RETURN 'Not enough funds';
  END IF;

  UPDATE accounts
         SET balance = balance + i_amount
	 WHERE owner = i_recipient;

  IF NOT FOUND THEN
    RETURN 'Recipient does not exist';
  END IF;

  UPDATE accounts
    SET blance = balance - i_amount
    WHERE owner = i_payer;
  RETURN 'OK';
END;
$$ LANGUAGE plpgsql;
~~~

使用刚刚创建的 function: transfer,

~~~sql
select * from transfer('Fred', 'Mary', 14.00);
~~~

### drop a function

~~~sql
DROP FUNCTION transfer(text, text, numeric(15, 2));
~~~
