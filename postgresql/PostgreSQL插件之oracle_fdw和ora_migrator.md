# PostgreSQL插件之ora_migrator

ora_migrator基于oracle_fdw插件提供Oracle到PostgreSQL的迁移功能。该插件只能迁移序列和普通表以及普通表上的约束和索引。其他所有的对象，包括但不限于触发器、函数、存储过程等都需要迁移人员自己手工迁移。

除此之外，该插件还可以用来创建一些外部表和视图，以便在PostgreSQL中可以方便的访问Oracle。

### 编译安装

* 下载源码：

  ora_migrator:  https://github.com/cybertec-postgresql/ora_migrator/tree/ORA_MIGRATOR_0_9_1

  oracle_fdw:     https://github.com/laurenz/oracle_fdw/tree/ORACLE_FDW_2_3_0

* 编译安装oracle_fdw

  安装oracle客户端： 将instantclient-basic-linux.x64-21.1.0.0.0.zip和instantclient-sdk-linux.x64-21.1.0.0.0.zip拷贝到服务器上解压，然后设置环境变量。

  ```shell
  $ unzip instantclient-sdk-linux.x64-21.1.0.0.0.zip
  $ unzip instantclient-basic-linux.x64-21.1.0.0.0.zip
  $ export ORACLE_HOME=/home/pgcore12/instantclient_21_1
  ```

  编译安装oracle_fdw

  ```shell
  $ unzip oracle_fdw-ORACLE_FDW_2_3_0.zip
  $ cd oracle_fdw-ORACLE_FDW_2_3_0
  $ make && make install
  ```

* 编译安装ora_migrator

  ```shell
  $ unzip ora_migrator-ORA_MIGRATOR_0_9_1.zip
  $ cd ora_migrator-ORA_MIGRATOR_0_9_1
  $ make install
  ```

* 创建扩展

  ```sql
  $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME 
  $ pg_ctl restart
  postgres=# create extension oracle_fdw;
  CREATE EXTENSION
  postgres=# create extension ora_migrator;
  CREATE EXTENSION
  ```

  

###  使用示例

* 使用oracle_fdw操作oracle数据库

  ```sql
  postgres=# CREATE SERVER oracle FOREIGN DATA WRAPPER oracle_fdw OPTIONS (dbserver 'serverip:port/orcl');
  CREATE SERVER
  postgres=# 
  postgres=# CREATE USER MAPPING FOR PUBLIC SERVER oracle OPTIONS (user 'username', password 'password');
  CREATE USER MAPPING
  postgres=# 
  postgres=# CREATE FOREIGN TABLE typetest1 (
  postgres(#    id  integer OPTIONS (key 'yes') NOT NULL,
  postgres(#    q   double precision,
  postgres(#    c   character(10),
  postgres(#    nc  character(10),
  postgres(#    vc  character varying(10),
  postgres(#    nvc character varying(10),
  postgres(#    lc  text,
  postgres(#    r   bytea,
  postgres(#    u   uuid,
  postgres(#    lb  bytea,
  postgres(#    lr  bytea,
  postgres(#    b   boolean,
  postgres(#    num numeric(7,5),
  postgres(#    fl  float,
  postgres(#    db  double precision,
  postgres(#    d   date,
  postgres(#    ts  timestamp with time zone,
  postgres(#    ids interval,
  postgres(#    iym interval
  postgres(# ) SERVER oracle OPTIONS (table 'TYPETEST1');
  CREATE FOREIGN TABLE
  postgres=# 
  postgres=# CREATE FOREIGN TABLE shorty (
  postgres(#    id  integer OPTIONS (key 'yes') NOT NULL,
  postgres(#    c   character(10)
  postgres(# ) SERVER oracle OPTIONS (table 'TYPETEST1');
  CREATE FOREIGN TABLE
  postgres=# 
  postgres=# CREATE FOREIGN TABLE longy (
  postgres(#    id  integer OPTIONS (key 'yes') NOT NULL,
  postgres(#    c   character(10),
  postgres(#    nc  character(10),
  postgres(#    vc  character varying(10),
  postgres(#    nvc character varying(10),
  postgres(#    lc  text,
  postgres(#    r   bytea,
  postgres(#    u   uuid,
  postgres(#    lb  bytea,
  postgres(#    lr  bytea,
  postgres(#    b   boolean,
  postgres(#    num numeric(7,5),
  postgres(#    fl  float,
  postgres(#    db  double precision,
  postgres(#    d   date,
  postgres(#    ts  timestamp with time zone,
  postgres(#    ids interval,
  postgres(#    iym interval,
  postgres(#    x   integer
  postgres(# ) SERVER oracle OPTIONS (table 'TYPETEST1');
  CREATE FOREIGN TABLE
  postgres=# 
  postgres=# ALTER FOREIGN TABLE typetest1 DROP q;
  ALTER FOREIGN TABLE
  postgres=# 
  postgres=# INSERT INTO typetest1 (id, c, nc, vc, nvc, lc, r, u, lb, lr, b, num, fl, db, d, ts, ids, iym) VALUES (
  postgres(#    1,
  postgres(#    'fixed char',
  postgres(#    'nat''l char',
  postgres(#    'varlena',
  postgres(#    'nat''l var',
  postgres(#    'character large object',
  postgres(#    bytea('\xDEADBEEF'),
  postgres(#    uuid('055e26fa-f1d8-771f-e053-1645990add93'),
  postgres(#    bytea('\xDEADBEEF'),
  postgres(#    bytea('\xDEADBEEF'),
  postgres(#    TRUE,
  postgres(#    3.14159,
  postgres(#    3.14159,
  postgres(#    3.14159,
  postgres(#    '1968-10-20',
  postgres(#    '2009-01-26 15:02:54.893532 PST',
  postgres(#    '1 day 2 hours 30 seconds 1 microsecond',
  postgres(#    '-6 months'
  postgres(# );
  INSERT 0 1
  postgres=# 
  postgres=# INSERT INTO shorty (id, c) VALUES (2, NULL);
  INSERT 0 1
  postgres=# INSERT INTO typetest1 (id, c, nc, vc, nvc, lc, r, u, lb, lr, b, num, fl, db, d, ts, ids, iym) VALUES (
  postgres(#    3,
  postgres(#    E'a\u001B\u0007\u000D\u007Fb',
  postgres(#    E'a\u001B\u0007\u000D\u007Fb',
  postgres(#    E'a\u001B\u0007\u000D\u007Fb',
  postgres(#    E'a\u001B\u0007\u000D\u007Fb',
  postgres(#    E'a\u001B\u0007\u000D\u007Fb ABC' || repeat('X', 9000),
  postgres(#    bytea('\xDEADF00D'),
  postgres(#    uuid('055f3b32-a02c-4532-e053-1645990a6db2'),
  postgres(#    bytea('\xDEADF00DDEADF00DDEADF00D'),
  postgres(#    bytea('\xDEADF00DDEADF00DDEADF00D'),
  postgres(#    FALSE,
  postgres(#    -2.71828,
  postgres(#    -2.71828,
  postgres(#    -2.71828,
  postgres(#    '0044-03-15 BC',
  postgres(#    '0044-03-15 12:00:00 BC',
  postgres(#    '-2 days -12 hours -30 minutes',
  postgres(#    '-2 years -6 months'
  postgres(# );
  INSERT 0 1
  postgres=#          
  postgres=# INSERT INTO typetest1 (id, c, nc, vc, nvc, lc, r, u, lb, lr, b, num, fl, db, d, ts, ids, iym) VALUES (
  postgres(#    4,
  postgres(#    'short',
  postgres(#    'short',
  postgres(#    'short',
  postgres(#    'short',
  postgres(#    'short',
  postgres(#    bytea('\xDEADF00D'),
  postgres(#    uuid('0560ee34-2ef9-1137-e053-1645990ac874'),
  postgres(#    bytea('\xDEADF00D'),
  postgres(#    bytea('\xDEADF00D'),
  postgres(#    NULL,
  postgres(#    0,
  postgres(#    0,
  postgres(#    0,
  postgres(#    NULL,
  postgres(#    NULL,
  postgres(#    '23:59:59.999999',
  postgres(#    '3 years'
  postgres(# );
  INSERT 0 1
  postgres=#         
  postgres=# SELECT id, c, nc, vc, nvc, length(lc), r, u, length(lb), length(lr), b, num, fl, db, d, ts, ids, iym, x FROM longy ORDER BY id;
  WARNING:  column number 19 of foreign table "longy" does not exist in foreign Oracle table, will be replaced by NULL
   id |          c           |          nc          |        vc        |       nvc        | length |     r      |                  u                   | length | length | b |   num    |     fl      |    db    |       d       |            
     ts                |          ids          |       iym        | x 
  ----+----------------------+----------------------+------------------+------------------+--------+------------+--------------------------------------+--------+--------+---+----------+-------------+----------+---------------+------------
  ---------------------+-----------------------+------------------+---
    1 | fixed char           | nat'l char           | varlena          | nat'l var        |     22 | \xdeadbeef | 055e26fa-f1d8-771f-e053-1645990add93 |      4 |      4 | t |  3.14159 |  3.14159012 |  3.14159 | 1968-10-20    | 2009-01-27 
  07:02:54.893532+08   | 1 day 02:00:30.000001 | -6 mons          |  
    2 |                      |                      |                  |                  |        |            |                                      |        |        |   |          |             |          |               |            
                       |                       |                  |  
    3 | a\x1B\x07\r\x7Fb     | a\x1B\x07\r\x7Fb     | a\x1B\x07\r\x7Fb | a\x1B\x07\r\x7Fb |   9010 | \xdeadf00d | 055f3b32-a02c-4532-e053-1645990a6db2 |     12 |     12 | f | -2.71828 | -2.71828008 | -2.71828 | 0044-03-15 BC | 0044-03-15 
  12:00:43+08:05:43 BC | -2 days -12:30:00     | -2 years -6 mons |  
    4 | short                | short                | short            | short            |      5 | \xdeadf00d | 0560ee34-2ef9-1137-e053-1645990ac874 |      4 |      4 |   |  0.00000 |           0 |        0 |               |            
                       | 23:59:59.999999       | 3 years          |  
  (4 rows)
  
  postgres=# 
  postgres=# WITH upd (id, c, lb, d, ts) AS
  postgres-#    (UPDATE longy SET c = substr(c, 1, 9) || 'u',
  postgres(#                     lb = lb || bytea('\x00'),
  postgres(#                     lr = lr || bytea('\x00'),
  postgres(#                      d = d + 1,
  postgres(#                     ts = ts + '1 day'
  postgres(#    WHERE id < 3 RETURNING id + 1, c, lb, d, ts)
  postgres-# SELECT * FROM upd ORDER BY id;
  WARNING:  column number 19 of foreign table "longy" does not exist in foreign Oracle table, will be replaced by NULL
   id |     c      |      lb      |     d      |              ts               
  ----+------------+--------------+------------+-------------------------------
    2 | fixed chau | \xdeadbeef00 | 1968-10-21 | 2009-01-28 07:02:54.893532+08
    3 |            |              |            | 
  (2 rows)
  
  postgres=#          
  postgres=# CREATE FOREIGN TABLE qtest (
  postgres(#    id  integer OPTIONS (key 'yes') NOT NULL,
  postgres(#    vc  character varying(10),
  postgres(#    num numeric(7,5)
  postgres(# ) SERVER oracle OPTIONS (table '(SELECT id, vc, num FROM typetest1)');
  CREATE FOREIGN TABLE
  postgres=# 
  postgres=# 
  postgres=# INSERT INTO qtest (id, vc, num) VALUES (5, 'via query', -12.5);
  INSERT 0 1
  postgres=# SELECT * FROM qtest ORDER BY id;
   id |        vc        |    num    
  ----+------------------+-----------
    1 | varlena          |   3.14159
    2 |                  |          
    3 | a\x1B\x07\r\x7Fb |  -2.71828
    4 | short            |   0.00000
    5 | via query        | -12.50000
  (5 rows)
  
  postgres=#          
                    
  --- 下面对应oracle数据库中的查询验证
  SQL> select count(*) from all_tables where table_name =upper('typetest1');
  
    COUNT(*)
  ----------
  	 1
  
  SQL> 
  ```

  

* 使用ora_migrator迁移oracle数据库

  oracle数据库授权

  ```sql
  SQL> grant SELECT ANY DICTIONARY to SCOTT;
  ```

  迁移oracle数据库

  ```sql
  postgres=# SELECT oracle_migrate(server => 'oracle', only_schemas => '{SCOTT}');
  NOTICE:  Creating staging schemas "ora_stage" and "pgsql_stage" ...
  NOTICE:  Creating Oracle metadata views in schema "ora_stage" ...
  NOTICE:  Copying definitions to PostgreSQL staging schema "pgsql_stage" ...
  NOTICE:  Creating schemas ...
  NOTICE:  Creating sequences ...
  NOTICE:  Creating foreign tables ...
  NOTICE:  Migrating table scott.gis ...
  NOTICE:  Migrating table scott.test1 ...
  NOTICE:  Migrating table scott.typetest1 ...
  NOTICE:  Creating UNIQUE and PRIMARY KEY constraints ...
  NOTICE:  Creating FOREIGN KEY constraints ...
  NOTICE:  Creating CHECK constraints ...
  NOTICE:  Creating indexes ...
  NOTICE:  Setting column default values ...
  NOTICE:  Dropping staging schemas ...
  NOTICE:  Migration completed with 0 errors.
   oracle_migrate 
  ----------------
                0
  (1 row)
  
  postgres=# 
  postgres=# select count(*) from scott.typetest1;
   count 
  -------
       5
  (1 row)
  
  postgres=# 
  postgres=# select id, c, nc from scott.typetest1;
   id |          c           |          nc          
  ----+----------------------+----------------------
    3 | a\x1B\x07\r\x7Fb     | a\x1B\x07\r\x7Fb    
    4 | short                | short     
    5 |                      | 
    1 | fixed chau           | nat'l char
    2 |                      | 
  (5 rows)
  
  postgres=# 
  ```

  

  

  
