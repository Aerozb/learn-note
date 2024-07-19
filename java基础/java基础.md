# jar包指定额外配置启动

在jar包后面拼上 `--`+配置项+等号+双引号包裹的值

```sh
nohup java -jar xxljob.jar --spring.datasource.url="数据库地址" --lock.datasource.url="数据库地址" >/dev/null 2>/dev/null &
```

