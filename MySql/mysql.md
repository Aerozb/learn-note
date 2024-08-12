# MySql8 去除ONLY_FULL_GROUP_BY

允许查询group by和order之外的字段

可以编辑 MySQL 配置文件 `my.cnf` 或 `my.ini`，并在 `[mysqld]` 部分下添加或修改以下行：

```ini
[mysqld]
sql_mode = "STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION"
```

通过这样做，你可以去除 `ONLY_FULL_GROUP_BY` 模式，允许在 `ORDER BY` 子句中使用不在 `SELECT` 列表中的列。