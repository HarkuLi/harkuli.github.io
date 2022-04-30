---
layout: post
title: "MySQL charset"
date: 2022-04-30 18:32:00 +0800
categories: mysql
---

Character set setting is significant for the communication between MySQL client
and server. Unexpected result might occur when using worng charset settings.

In this article, we will see scopes, definition for charset system variables and
how they are used to interpret a query.

At the end, we will learn how charset settings affect a query result through an
example.

## Scope

Charset variables are a part of system variables. There are two scopes for
system variables:

*   Global
    *   Global system variables are initilized when **server starts up**.
    *   Used as fallback values for session variables.
*   Session
    *   Session system variables are initilized at **connection time**.
    *   Changes are independent for each session.

## Related variables

We can check global variables by following command:

```
mysql> SHOW GLOBAL VARIABLES LIKE "character%";
+--------------------------+--------------------------------+
| Variable_name            | Value                          |
+--------------------------+--------------------------------+
| character_set_client     | utf8mb4                        |
| character_set_connection | utf8mb4                        |
| character_set_database   | utf8mb4                        |
| character_set_filesystem | binary                         |
| character_set_results    | utf8mb4                        |
| character_set_server     | utf8mb4                        |
| character_set_system     | utf8mb3                        |
| character_sets_dir       | /usr/share/mysql-8.0/charsets/ |
+--------------------------+--------------------------------+
8 rows in set (0.01 sec)
```

Let's pick up some variables that will be mentioned in this article to know about their meaning.

*   [character_set_client](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_character_set_client)\
    The charset that the client uses.
*   [character_set_connection](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_character_set_connection)\
    It is used for literals specified without a
    [charset introducer](https://dev.mysql.com/doc/refman/8.0/en/charset-introducer.html).
    The corresponding variable `collation_connection` is important for
    comparisons of **literal strings**.
*   [character_set_results](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_character_set_results)\
    The charset that is used to respond query result.
*   [character_set_database](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_character_set_database)\
    The charset used by the
    [default database](https://dev.mysql.com/doc/refman/8.0/en/use.html).
*   [character_set_server](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_character_set_server)\
    The default charset of server.

## Column charset

The charset of a column is determined by following priority:

1.  Column charset\
    e.g.

    ```sql
    CREATE TABLE t1
    (
        col1 CHAR(10) CHARACTER SET utf8mb4
    );
    ```

2.  Table charset\
    e.g.

    ```sql
    CREATE TABLE t1
    (
        col1 CHAR(10)
    ) CHARACTER SET utf8mb4;
    ```

3.  Server charset
    *   The value of system variable `character_set_server`.

---

**NOTE**

If the collation isn't specified, default collation of the charset will be used.\
By contrast, if collation is specified without charset, the associated charset
will be used.

---

##  Query translation

A MySQL query is translated in following flow:

```
+--------+
| client |
+--------+
    |
    | Interpreted as "character_set_client" and
    | translated to "character_set_connection"
    v
+--------+
| server |
+--------+
    |
    | Translated to column charset
    v
+--------+
| column |
+--------+
```

## Case study

`docker-compose.yaml` for MySQL server:

```yaml
version: '3.9'

services:
  mysql:
    image: mysql:5.7
    ports:
      - 3306:3306
    environment:
      MYSQL_DATABASE: test
      MYSQL_ROOT_PASSWORD: password
```

Table creation command:

```sql
CREATE TABLE test.t1
(
    col1 VARCHAR(10)
) CHARACTER SET utf8mb4;
```

PHP script in **UTF-8** encoding:

```php
<?php

$pdo = new PDO(
    'mysql:dbname=test;host=127.0.0.1',
    'root',
    'password'
);

$pdo->exec('INSERT INTO t1 (col1) VALUES ("🍺🍺🍺");');
```

Executing above script will result in an PDOException:

```
PHP Fatal error:  Uncaught PDOException: SQLSTATE[22001]: String data, right truncated: 1406 Data too long for column 'col1' at row 1 in /app/index.php:9
Stack trace:
#0 /app/index.php(9): PDO->exec('INSERT INTO t1 ...')
#1 {main}
  thrown in /app/index.php on line 9

Fatal error: Uncaught PDOException: SQLSTATE[22001]: String data, right truncated: 1406 Data too long for column 'col1' at row 1 in /app/index.php:9
Stack trace:
#0 /app/index.php(9): PDO->exec('INSERT INTO t1 ...')
#1 {main}
  thrown in /app/index.php on line 9
```

The query for inserting a 3-character string "🍺🍺🍺" into a 10-character column
is failed. The result is surprising, isn't it?

To understand the cause, let's check charset settings step by step.

First, the column charset isn't specified, so it uses default encoding of the
table, which is `utf8mb4`.

Next step, let's check the server charset. According to
[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/charset-server.html),
the default value for server charset is `latin1` if it isn't set explicitly at
server startup.\
We can verify this by checking global charset variables:

```
mysql> SHOW GLOBAL VARIABLES LIKE "character%";
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | latin1                     |
| character_set_connection | latin1                     |
| character_set_database   | latin1                     |
| character_set_filesystem | binary                     |
| character_set_results    | latin1                     |
| character_set_server     | latin1                     |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)
```

Then, let's check the client charset. Because we don't specify the client
charset explicitly, the server charset will be used as client charset.
We can check it through the created PDO instance:

```php
<?php

$pdo = new PDO(
    'mysql:dbname=test;host=127.0.0.1',
    'root',
    'password'
);

$result = $pdo
    ->query('SHOW VARIABLES LIKE "character_set_client"', PDO::FETCH_ASSOC)
    ->fetch();
print_r($result);
```

The value of `character_set_client` is `latin1`.

```
Array
(
    [Variable_name] => character_set_client
    [Value] => latin1
)
```

Now, we can know how the failed query is executed by MySQL:

1.  "🍺" is a 4-byte character in **UTF-8** encoding, so the "🍺🍺🍺" string is
    **12 bytes**.
2.  The 12-byte string will be interpreted as a **12-character** string in
    `latin1` encoding by MySQL server according to the `character_set_client`
    system variable.
3.  The string is then translated as a **12-character** string in the encoding
    defined by the system variable `character_set_connection`.
4.  Finally, the string is translated as a **12-character** string in `utf8mb4`
    encoding according to the column charset. As a result, it exceeds the length
    limitation of the column which is 10 characters.

---

**NOTE**

The system variable `character_set_connection` doesn't matter here.\
Because the number of characters won't change after translations.

---
\
We can come to the conclusion that the cause is wrong value for the
system variable `character_set_client`.

Here are some solutions to fix it:

1.  Specify client charset in the data source name (DSN).

    ```
    mysql:host=mysql;dbname=test;charset=utf8mb4
    ```

2.  Specify client charset after the connection is built.

    ```sql
    SET NAMES utf8mb4;
    ```

3.  Specify server charset as `utf8mb4` when starting mysql server.

    ```yaml
    version: '3.9'

    services:
      mysql:
        image: mysql:5.7
        ports:
          - 3306:3306
        environment:
          MYSQL_DATABASE: test
          MYSQL_ROOT_PASSWORD: password
        command:
          - --character-set-server=utf8mb4
    ```

## Good practice

Although the charset problem can be solved by changing the server charset in
above case, it isn't a good idea. Because the database server might be shared by
several clients, changing the setting of database server will affect other
clients.

It is recommended to explicitly specify charset for each connection so that we
won't get wrong results because of unexpected default charset setting.

---

**NOTE**

In Laravel, we can add charset option in the
[connection setting](https://laravel.com/docs/9.x/database).

```php
'mysql' => [
    'read' => [...],
    'write' => [...],
    'driver' => 'mysql',
    'database' => 'test',
    'charset' => 'utf8mb4',
],
```

If the charset option is set,
[it will be used to set client charset](https://github.com/laravel/framework/blob/9.x/src/Illuminate/Database/Connectors/MySqlConnector.php#L76)
right after the connection is built.

---

## Reference

* [MySQL 8.0 Reference Manual  /  Connection Character Sets and Collations](https://dev.mysql.com/doc/refman/8.0/en/charset-connection.html)
* [MySQL 8.0 Reference Manual  /  Server Character Set and Collation](https://dev.mysql.com/doc/refman/8.0/en/charset-server.html)
