# 创建一个 MySQL 测试数据库

为了测试目的，我们在本书的源代码中提供了一个 SQL 文件，其中包含了 [https://github.com/dbierer/php7cookbook](https://github.com/dbierer/php7cookbook) 的示例数据。这本书的示例中使用的数据库的名称是 php7cookbook。

## 怎么做...

1.定义一个MySQL数据库 php7cookbook。同时将新数据库的权限分配给一个名为 cook 的用户，密码为book。下表总结了这些设置。

| 项目 | 注释 |
| :--- | :--- |
| 数据库名称 | `php7cookbook` |
| 数据库用户 | `cook` |
| 数据库用户密码 | `book` |

2.这是创建数据库所需的SQL示例：

```sql
CREATE DATABASE IF NOT EXISTS dbname DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
CREATE USER 'user'@'%' IDENTIFIED WITH mysql_native_password;
SET PASSWORD FOR 'user'@'%' = PASSWORD('userPassword');
GRANT ALL PRIVILEGES ON dbname.* to 'user'@'%';
GRANT ALL PRIVILEGES ON dbname.* to 'user'@'localhost';
FLUSH PRIVILEGES;
```

3.将样本值导入到新数据库中。 导入文件`php7cookbook.sql`位于[https://github.com/dbierer/php7cookbook/blob/master/php7cookbook.sql](https://github.com/dbierer/php7cookbook/blob/master/php7cookbook.sql)。

