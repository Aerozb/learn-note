# sharding-jdbc

## 基本概念
### 逻辑表

水平拆分的数据表的总称。例：订单数据表根据主键尾数拆分为10张表，分别是 t_order_0 、 t_order_1 到 t_order_9 ，他们的逻辑表名为 t_order 。

### 真实表

在分片的数据库中真实存在的物理表。即上个示例中的 t_order_0 到 t_order_9 。

### 数据节点

数据分片的最小物理单元。由数据源名称和数据表组成，例：ds_0.t_order_0 。

### 分片键

用于分片的数据库字段，是将数据库(表)水平拆分的关键字段。例：将订单表中的订单主键的尾数取模分片，则订单主键为分片字段。 SQL中如果无分片字段，将执行全路由，性能较差。 除了对单分片字段的支持，ShardingSphere也支持根据多个字段进行分片。

### 分片算法

通过分片算法将数据分片，支持通过`=`、`>=`、`<=`、`>`、`<`、`BETWEEN`和`IN`分片。分片算法需要应用方开发者自行实现，可实现的灵活度非常高。

目前提供4种分片算法。由于分片算法和业务实现紧密相关，因此并未提供内置分片算法，而是通过分片策略将各种场景提炼出来，提供更高层级的抽象，并提供接口让应用开发者自行实现分片算法。

- 精确分片算法

对应PreciseShardingAlgorithm，用于处理使用单一键作为分片键的=与IN进行分片的场景。需要配合StandardShardingStrategy使用。

- 范围分片算法

对应RangeShardingAlgorithm，用于处理使用单一键作为分片键的BETWEEN AND、>、<、>=、<=进行分片的场景。需要配合StandardShardingStrategy使用。

- 复合分片算法

对应ComplexKeysShardingAlgorithm，用于处理使用多键作为分片键进行分片的场景，包含多个分片键的逻辑较复杂，需要应用开发者自行处理其中的复杂度。需要配合ComplexShardingStrategy使用。

- Hint分片算法

对应HintShardingAlgorithm，用于处理使用Hint行分片的场景。需要配合HintShardingStrategy使用。

### 分片策略

包含分片键和分片算法，由于分片算法的独立性，将其独立抽离。真正可用于分片操作的是分片键 + 分片算法，也就是分片策略。目前提供5种分片策略。

具体算法实现，需要在配置中指定实现类，他会根据你的配置找到对应算法

- 标准分片策略

对应StandardShardingStrategy。提供对SQL语句中的=, >, <, >=, <=, IN和BETWEEN AND的分片操作支持。StandardShardingStrategy只支持单分片键，提供PreciseShardingAlgorithm和RangeShardingAlgorithm两个分片算法。PreciseShardingAlgorithm是必选的，用于处理=和IN的分片。RangeShardingAlgorithm是可选的，用于处理BETWEEN AND, >, <, >=, <=分片，如果不配置RangeShardingAlgorithm，SQL中的BETWEEN AND将按照全库路由处理。

`StandardShardingStrategy` 是一个配置工具，用于帮助你指定分片键和分片算法。你无需继承它，而是通过配置来指定具体的分片算法（如 `PreciseShardingAlgorithm` 和 `RangeShardingAlgorithm`）的实现。这两个算法类需要你自行实现，以处理具体的分片逻辑。

- 复合分片策略

对应ComplexShardingStrategy。复合分片策略。提供对SQL语句中的=, >, <, >=, <=, IN和BETWEEN AND的分片操作支持。ComplexShardingStrategy支持多分片键，由于多分片键之间的关系复杂，因此并未进行过多的封装，而是直接将分片键值组合以及分片操作符透传至分片算法，完全由应用开发者实现，提供最大的灵活度。

- 行表达式分片策略

对应InlineShardingStrategy。使用Groovy的表达式，提供对SQL语句中的=和IN的分片操作支持，只支持单分片键。对于简单的分片算法，可以通过简单的配置使用，从而避免繁琐的Java代码开发，如: `t_user_$->{u_id % 8}` 表示t_user表根据u_id模8，而分成8张表，表名称为`t_user_0`到`t_user_7`。

- Hint分片策略

对应HintShardingStrategy。通过Hint指定分片值而非从SQL中提取分片值的方式进行分片的策略。

- 不分片策略

对应NoneShardingStrategy。不分片的策略。



分片策略是一个更高层次的概念，它定义了如何应用分片算法以及如何处理不同类型的分片需求。分片策略的主要任务是管理和协调分片算法的使用，以应对不同的查询模式和业务需求。它通常包含以下几个方面：

1. **分片键的选择**：
   - 确定用于分片的列，比如 `order_id` 或 `order_date`。
   - 分片键决定了分片算法如何执行。
2. **分片算法的选择和配置**：
   - 根据查询类型配置合适的分片算法，比如精确分片算法或范围分片算法。

# spring-boot 2.7.18整合sharding-jdbc-spring-boot-starter 4.1.1

5的整合不了暂时有问题

## 需求

**需求一**：根据创建时间的年月分表

**需求二**：根据 省份和创建时间的年月 分表

还需要自动创建表

## 代码

https://github.com/Aerozb/learn-project/tree/main/sharding/version4

## 创建表

```sql
CREATE TABLE `sharding_user` (
  `id` bigint NOT NULL COMMENT '主键',
  `username` varchar(32) CHARACTER SET utf8mb3 COLLATE utf8mb3_general_ci NOT NULL COMMENT '用户名',
  `province_abbreviation` varchar(8) CHARACTER SET utf8mb3 COLLATE utf8mb3_general_ci NOT NULL COMMENT '省份拼音缩写',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3 COMMENT='用户表';
```

## 依赖



使用`druid-spring-boot-starter` 会报错`Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required`

所以直接用druid

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
            <version>4.1.1</version>
        </dependency>
        <!--        使用druid-spring-boot-starter 会报错Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.2.23</version>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.5.6</version>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.4.0</version>
        </dependency>
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>2.1.0</version>
        </dependency>
            <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>5.8.28</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.26</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.2.11</version>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>2.7.18</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

## 配置

单片和多片分键都有写，用哪个启用那个，只能启用一个分片策略

```yaml
spring:
  shardingsphere:
    datasource:
      names: learn # 数据源名称，多数据源以逗号分隔
      learn:
        type: com.alibaba.druid.pool.DruidDataSource # 数据库连接池类名称
        driver-class-name: com.mysql.cj.jdbc.Driver # 数据库驱动类名
        url: jdbc:mysql://47.116.44.79:3306/learn?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai # 数据库url连接
        username: root # 数据库用户名
        password: 123456 # 数据库密码
    sharding:
      default-data-source-name: learn
      tables:
        sharding_user:
          # 会根据这里的表达式生成分表名，传入到各个分片算法的doSharding方法的Collection<String> availableTargetNames
          actual-data-nodes: learn.sharding_user
          # 分表策略
          table-strategy:
            # 用于单分片键的标准分片场景
#            standard:
#              sharding-column: create_time # 分片列名称
#              precise-algorithm-class-name: com.yhy.sharding.SinglColumnPreciseShardingAlgorithm # 精确分片算法类名称，用于=和IN。该类需实现PreciseShardingAlgorithm接口并提供无参数的构造器
#              range-algorithm-class-name: com.yhy.sharding.SinglColumnPreciseShardingAlgorithm # 范围分片算法类名称，用于BETWEEN，可选。该类需实现RangeShardingAlgorithm接口并提供无参数的构造器

              # 用于多分片键的复合分片场景
              complex:
                sharding-columns:  province_abbreviation,create_time # 分片列名称，多个列以逗号分隔
                algorithm-class-name: com.yhy.sharding.MultiColumnComplexKeysShardingAlgorithm # 复合分片算法类名称。该类需实现ComplexKeysShardingAlgorithm接口并提供无参数的构造器
    props:
      sql.show: true # 是否开启SQL显示，默认值: false


mybatis-plus:
  configuration:
    #开启驼峰命名自动映射
    map-underscore-to-camel-case: true
    #开启日志打印
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  type-aliases-package: com.yhy.entity
  #扫描mapper文件
  mapper-locations: classpath:mapper/*.xml

pagehelper:
  helperDialect: mysql
  reasonable: true
  supportMethodsArguments: true
  params: count=countSql
```

## 表操作mapper类

用来创建表和判断表是否存在

```java
@Mapper
public interface TableMapper {
    void createTable(@Param("tableName") String tableName, @Param("templateTableName") String templateTableName);

    int existTable(@Param("tableSchema") String tableSchema, @Param("tableName") String tableName);

    List<String> getShardingTableName(@Param("tableSchema") String tableSchema, @Param("templateTableName") String templateTableNam);
}
```

## 表操作mapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.yhy.mapper.TableMapper">
    <select id="existTable" resultType="int">
        SELECT COUNT(*)
        FROM information_schema.tables
        WHERE table_schema = #{tableSchema}
          AND table_name = #{tableName}
    </select>
    <update id="createTable">
        CREATE TABLE ${tableName} like ${templateTableName}
    </update>
        <select id="getShardingTableName" resultType="String">
        SELECT table_name
        FROM information_schema.tables
        WHERE table_schema = #{tableSchema}
          AND table_name like CONCAT(#{templateTableName}, '_%')
    </select>
</mapper>
```

## 常量

```java
public interface Constant {
    /**
     * 需要分片的库
     */
    String SHARDING_DB_NAME = "learn";

    /**
     * 需要分片的表
     */
    String SHARDING_TABLE_NAME = "sharding_user";
}
```



## 分片工具类

```java
public class ShardingUtil {
    private static final Map<String, Integer> tableExistMap = new ConcurrentHashMap<>();
    private static TableMapper tableMapper;

    public static void setTableMapper(TableMapper tableMapper) {
        ShardingUtil.tableMapper = tableMapper;
    }

    public static void setTableExistMap(List<String> shardingTableNameList) {
        for (String shardingTableName : shardingTableNameList) {
            int shardingYearMonth = getShardingYearMonth(shardingTableName);
            tableExistMap.put(shardingTableName, shardingYearMonth);
        }
    }

    public static boolean isExistTableCache(String tableName) {
        return tableExistMap.containsKey(tableName);
    }

    /**
     * 不存在表，则创建
     * 日期大于上个月的，不创建
     *
     * @param tableName         要创建的表名-分片后的表名
     * @param templateTableName 未分片的表
     */
    public static void createTable(String tableName, String templateTableName) {
        if (isExistTableCache(tableName)) {
            return;
        }

        int shardingYearMonth = getShardingYearMonth(tableName);
        int nowYearMonth = Integer.parseInt(DateUtil.format(new Date(), DatePattern.SIMPLE_MONTH_PATTERN));
        if (shardingYearMonth > nowYearMonth) {
            throw new RuntimeException("创建表失败超出当前年月:" + shardingYearMonth);
        }

        int row = tableMapper.existTable(Constant.SHARDING_DB_NAME, tableName);
        //表不存在则创建并塞入缓存
        if (row <= 0) {
            tableMapper.createTable(tableName, templateTableName);
            tableExistMap.put(tableName, shardingYearMonth);
        } else {
            tableExistMap.put(tableName, shardingYearMonth);
        }
    }

    /**
     * 获取日期最新的表
     *
     * @param provinceAbbreviation 省份缩写
     * @return 最新日期的表对应的日期
     */
    public static Date getExistLatestDate(String provinceAbbreviation) {
        return tableExistMap.entrySet().stream()
                .filter(entry -> entry.getKey().contains(provinceAbbreviation))
                .map(Map.Entry::getValue)
                .max(Integer::compareTo)
                .map(latestYearMonth -> DateUtil.parse(String.valueOf(latestYearMonth), DatePattern.SIMPLE_MONTH_PATTERN))
                .orElseThrow(() -> new RuntimeException("不存在此省份分表:" + provinceAbbreviation));
    }

    private static int getShardingYearMonth(String tableName) {
        return Integer.parseInt(StrUtil.subSufByLength(tableName, DatePattern.SIMPLE_MONTH_PATTERN.length()));
    }
}
```

## 给工具类注入表操作mapper

```java
/**
 * 项目启动后
 * 1.分表工具类注入属性
 * 2.把已有的真实表加载进缓存，否则项目重启缓存不见
 */
@Slf4j
@Order(value = 1) // 数字越小 越先执行
@Component
public class ShardingTablesLoadRunner implements CommandLineRunner {

    @Resource
    private TableMapper tableMapper;

    @Override
    public void run(String... args) {
        // 给 分表工具类注入属性
        ShardingUtil.setTableMapper(tableMapper);
        //加载真实表缓存
        List<String> shardingTableNameList = tableMapper.getShardingTableName(Constant.SHARDING_DB_NAME, Constant.SHARDING_TABLE_NAME);
        ShardingUtil.setTableExistMap(shardingTableNameList);
    }
}
```

## 单个分片键策略-用于精确和范围查询（解决需求一）

```java
/**
 * 单个分片键-根据年月分表
 */
public class SingleColumnPreciseShardingAlgorithm implements PreciseShardingAlgorithm<Date>, RangeShardingAlgorithm<Date> {

    /**
     * 这个方法用于处理精确分片，即处理单个分片键值的分片场景。例如，处理一个具体的订单 ID，决定它应该被路由到哪个数据源或表。
     *
     * @param availableTargetNames 可用的目标数据源或表的集合
     * @param shardingValue        包含分片键值以及逻辑表名等信息的对象
     * @return 单张表
     */
    @Override
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Date> shardingValue) {
        Date createTime = shardingValue.getValue();
        String shardingName = DateUtil.format(createTime, DatePattern.SIMPLE_MONTH_PATTERN);
        //要分片的表
        String logicTableName = shardingValue.getLogicTableName();
        //实际要查询的表名
        String tableName = logicTableName + "_" + shardingName;
        ShardingUtil.createTable(tableName, logicTableName);
        return tableName;
    }

    /**
     * 这个方法用于处理范围分片，即处理一段分片键值范围的分片场景。例如，处理一个范围查询（如查询某一段时间内的订单），决定这些记录应该被路由到哪些数据源或表。
     *
     * @return 多张表
     */
    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, RangeShardingValue<Date> shardingValue) {
        Range<Date> range = shardingValue.getValueRange();
        String logicTableName = shardingValue.getLogicTableName();
        List<DateTime> dateRange = DateUtil.rangeToList(range.lowerEndpoint(), range.upperEndpoint(), DateField.MONTH);
        List<String> tableNameList = new ArrayList<>();
        //生成年月的分表名
        for (DateTime dateTime : dateRange) {
            String tableName = logicTableName + "_" + DateUtil.format(dateTime, DatePattern.SIMPLE_MONTH_PATTERN);
            //不存在此表，则不返回表名，否则查询时表不存在会报错
            if (ShardingUtil.isExistTableCache(tableName)) {
                tableNameList.add(tableName);
            }
        }
        return tableNameList;
    }

}
```

## 自定义多片键（解决需求二）

```java
/**
 * 多列复杂分片算法，根据省份缩写和年月分表
 */
@Slf4j
public class MultiColumnComplexKeysShardingAlgorithm implements ComplexKeysShardingAlgorithm<Comparable<?>> {

    /**
     * 处理多列复杂分片
     *
     * @param availableTargetNames 可用的目标表名集合
     * @param shardingValue        包含分片键值和逻辑表名等信息
     * @return 匹配的表名集合
     */
    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, ComplexKeysShardingValue<Comparable<?>> shardingValue) {
        String logicTableName = shardingValue.getLogicTableName();
        Map<String, Collection<Comparable<?>>> columnNameAndShardingValuesMap = shardingValue.getColumnNameAndShardingValuesMap();
        Map<String, Range<Comparable<?>>> columnNameAndRangeValuesMap = shardingValue.getColumnNameAndRangeValuesMap();

        //精确查询的省份
        Collection<Comparable<?>> provinceAbbreviationList = columnNameAndShardingValuesMap.get("province_abbreviation");
        //精确查询的创建时间
        Collection<Comparable<?>> createTimeList = columnNameAndShardingValuesMap.get("create_time");
        //范围查询的创建时间
        Range<Comparable<?>> createTimeRange = columnNameAndRangeValuesMap.get("create_time");

        //校验分片键
        validateInputs(provinceAbbreviationList, createTimeList, createTimeRange);

        //获取省份
        String provinceAbbreviation = getProvinceAbbreviation(provinceAbbreviationList);

        //精确查询
        if (CollUtil.isNotEmpty(createTimeList)) {
            return handlePreciseSharding(logicTableName, provinceAbbreviation, createTimeList);
        }

        //范围查询
        return handleRangeSharding(logicTableName, provinceAbbreviation, createTimeRange);
    }

    private void validateInputs(Collection<Comparable<?>> provinceAbbreviationList, Collection<Comparable<?>> createTimeList, Range<Comparable<?>> createTimeRange) {
        if (CollUtil.isEmpty(provinceAbbreviationList)) {
            throw new RuntimeException("路由表失败,province_abbreviation不能为空");
        }
        if (CollUtil.isEmpty(createTimeList) && createTimeRange == null) {
            throw new RuntimeException("路由表失败,create_time不能为空");
        }
        if (provinceAbbreviationList.size() > 1) {
            throw new RuntimeException("路由表失败,province_abbreviation精确查询只能查1个");
        }
        if (CollUtil.isNotEmpty(createTimeList) && createTimeList.size() > 1) {
            throw new RuntimeException("路由表失败,create_time精确查询只能查1个");
        }
    }

    private String getProvinceAbbreviation(Collection<Comparable<?>> valueList) {
        return valueList.stream()
                .map(String::valueOf)
                .findFirst()
                .orElseThrow(() -> new RuntimeException("获取province_abbreviation异常"));
    }

    private Collection<String> handlePreciseSharding(String logicTableName, String provinceAbbreviation, Collection<Comparable<?>> createTimeList) {
        String tableName = logicTableName + "_" + provinceAbbreviation + "_" + createTimeList.stream()
                .map(comparable -> DateUtil.format((Date) comparable, DatePattern.SIMPLE_MONTH_PATTERN))
                .findFirst()
                .orElseThrow(() -> new RuntimeException("获取create_time异常"));
        ShardingUtil.createTable(tableName, logicTableName);
        return Collections.singleton(tableName);
    }

    private Collection<String> handleRangeSharding(String logicTableName, String provinceAbbreviation, Range<Comparable<?>> createTimeRange) {
        //必传
        Date startDate = getStartDate(createTimeRange);
        //不传或者大于上个月，就改为存在此省份表的最新日期
        Date endDate = getEndDate(createTimeRange, provinceAbbreviation);

        List<DateTime> dateRange = DateUtil.rangeToList(startDate, endDate, DateField.MONTH);
        List<String> tableNameList = new ArrayList<>();

        //返回已存在表的表名
        for (DateTime dateTime : dateRange) {
            String tableName = logicTableName + "_" + provinceAbbreviation + "_" + DateUtil.format(dateTime, DatePattern.SIMPLE_MONTH_PATTERN);
            if (ShardingUtil.isExistTableCache(tableName)) {
                tableNameList.add(tableName);
            }
        }
        return tableNameList;
    }

    private Date getStartDate(Range<Comparable<?>> createTimeRange) {
        try {
            return (Date) createTimeRange.lowerEndpoint();
        } catch (IllegalStateException e) {
            throw new RuntimeException("请传入开始日期");
        }
    }

    private Date getEndDate(Range<Comparable<?>> createTimeRange, String provinceAbbreviation) {
        try {
            Date date = (Date) createTimeRange.upperEndpoint();
            String shardingYearMonth = DateUtil.format(date, DatePattern.SIMPLE_MONTH_PATTERN);
            String nowYearMonth = DateUtil.format(new Date(), DatePattern.SIMPLE_MONTH_PATTERN);
            if (Integer.parseInt(shardingYearMonth) > Integer.parseInt(nowYearMonth)) {
                date = ShardingUtil.getExistLatestDate(provinceAbbreviation);
            }
            return date;
        } catch (IllegalStateException e) {
            return ShardingUtil.getExistLatestDate(provinceAbbreviation);
        }
    }
}
```

## 编写测试请求

先请求保存，在请求多片键multiColumnList

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/singleColumnList")
    public PageInfo<User> singleColumnList() {
        PageHelper.startPage(1, 10);
        LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
        wrapper.ge(User::getCreateTime, DateUtil.parse("202302", DatePattern.SIMPLE_MONTH_PATTERN));
        wrapper.le(User::getCreateTime, new Date());
        List<User> list = userService.list(wrapper);
        return new PageInfo<>(list);
    }

    @GetMapping("/multiColumnList")
    public PageInfo<User> multiColumnList() {
        PageHelper.startPage(1, 10);
        LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
        wrapper.eq(User::getProvinceAbbreviation, "wh");
//        wrapper.ge(User::getCreateTime,DateUtil.parse("202302", DatePattern.SIMPLE_MONTH_PATTERN));
//        wrapper.le(User::getCreateTime,new Date());
        wrapper.eq(User::getCreateTime, DateUtil.parse("202403", DatePattern.SIMPLE_MONTH_PATTERN));
        List<User> list = userService.list(wrapper);
        return new PageInfo<>(list);
    }


    @PostMapping("/save")
    public void save() {
        List<User> users = generateUsers(5);
        for (User user : users) {
            userService.save(user);
        }
    }

    public static List<User> generateUsers(int count) {
        List<User> users = new ArrayList<>();
        List<Date> dates = new ArrayList<>();
        List<String> provinceAbbreviation = new ArrayList<>();
        provinceAbbreviation.add("fz");
        provinceAbbreviation.add("bj");
        provinceAbbreviation.add("wh");
        provinceAbbreviation.add("hn");
        provinceAbbreviation.add("sh");
        dates.add(DateUtil.parse("202401", DatePattern.SIMPLE_MONTH_PATTERN));
        dates.add(DateUtil.parse("202402", DatePattern.SIMPLE_MONTH_PATTERN));
        dates.add(DateUtil.parse("202403", DatePattern.SIMPLE_MONTH_PATTERN));
        dates.add(DateUtil.parse("202404", DatePattern.SIMPLE_MONTH_PATTERN));
        dates.add(DateUtil.parse("202405", DatePattern.SIMPLE_MONTH_PATTERN));
        for (int i = 0; i < count; i++) {
            User user = new User();
            long snowflakeNextId = IdUtil.getSnowflakeNextId();
            user.setId(snowflakeNextId);
            user.setUsername("user" + snowflakeNextId);
            user.setProvinceAbbreviation(provinceAbbreviation.get(i));
            user.setCreateTime(dates.get(i));
            users.add(user);
        }
        return users;
    }

}
```
