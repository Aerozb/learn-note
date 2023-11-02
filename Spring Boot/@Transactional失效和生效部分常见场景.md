# @Transactional失效和生效部分常见场景

## 1.不生效

### 1.1 没有带事务注解的方法调带事务注解的方法

```java
public void initUser1(User user) {
    save(user);
    initPoint1(user.getId());
}

@Transactional
public void initPoint1(Integer userId) {
    UserPoint userPoint = new UserPoint();
    userPoint.setUserId(userId);
    userPoint.setPoint(0);
    userPointService.save(userPoint);
    int i = 5 / 0;
}

```

```java
@Test
void test1() {
    User user = new User();
    user.setId(1);
    user.setName("1");
    userService.initUser1(user);
}
```

### 1.2 捕获异常，不抛出

```java
@Transactional
public void initUser3(User user) {
    save(user);
    initPoint3(user.getId());
}

public void initPoint3(Integer userId) {
    UserPoint userPoint = new UserPoint();
    userPoint.setUserId(userId);
    userPoint.setPoint(0);
    userPointService.save(userPoint);
    try {
        int i = 5 / 0;
    } catch (Exception e) {
        System.out.println("不抛出异常，事务不生效");
    }
}
```

```java
@Test
void test3() {
   User user = new User();
   user.setId(3);
   user.setName("3");
   userService.initUser3(user);
}
```

### 1.3 不带事务方法调带事务方法

```java
public void initUser4(User user) {
    save(user);
    initUser4_1(user);
}

@Transactional
public void initUser4_1(User user) {
    user.setId(user.getId()+1);
    save(user);
    int i = 5 / 0;
}
```

```java
@Test
void test4() {
   User user = new User();
   user.setId(4);
   user.setName("4");
   userService.initUser4(user);
}
```

## 2.生效

### 2.1 带事务注解的方法调无事务注解的私有(公有)方法

```java
@Transactional
public void initUser2(User user) {
    save(user);
    initPoint2(user.getId());
}

private void initPoint2(Integer userId) {
    UserPoint userPoint = new UserPoint();
    userPoint.setUserId(userId);
    userPoint.setPoint(0);
    userPointService.save(userPoint);
    int i = 5 / 0;
}
```

```java
@Test
void test2() {
    User user = new User();
    user.setId(2);
    user.setName("2");
    userService.initUser2(user);
}
```

### 2.2  @Transactional写在类上，公有调用私有

```java
@Service
@Transactional
public class UserService2 extends ServiceImpl<UserMapper, User> {

    @Autowired
    private UserPointService userPointService;

    public void initUser6(User user) {
        save(user);
        initPoint6(user.getId());
    }

    private void initPoint6(Integer userId) {
        UserPoint userPoint = new UserPoint();
        userPoint.setUserId(userId);
        userPoint.setPoint(0);
        userPointService.save(userPoint);
        int i = 6 / 0;
    }
    
}
```

```java
@Test
void test6() {
   User user = new User();
   user.setId(6);
   user.setName("6");
   userService2.initUser6(user);
}
```