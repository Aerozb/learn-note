# jar包指定额外配置启动

在jar包后面拼上 `--`+配置项+等号+双引号包裹的值

```sh
nohup java -jar xxljob.jar --spring.datasource.url="数据库地址" --lock.datasource.url="数据库地址" >/dev/null 2>/dev/null &
```

配置项作为 Spring Boot 应用的命令行参数传递，而不是作为 JVM 参数，应该将它放在 `xxljob.jar` jar包后面

放在`java`后面是`jvm`参数

