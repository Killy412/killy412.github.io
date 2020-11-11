---
title: SpringBoot 手动控制事务
date: 2020-11-11 18:41:20
categories: 
  - "SpringBoot"
tags:
  - "SpringBoot"
  - "事物"
---

### 前言
今天进行项目开发时遇到一个事物相关问题,有一块业务使用策略模式实现的,使用注释式事物是不生效的.个人猜测这一块应该是`Red.handler()`方法不是被spring所管理的bean,所以注解式事物是无效的. 所以把这块改成了手动控制事物.
<!--more-->
```java
public enum ColorEnum {
    Red("红色") {
        @Override
        @Transactional(rollbackFor = Exception.class)
        public int handler(boolean bool) {
            //...
        }
    };
    public String name;
    ColorEnum(String name) {
        this.name = name;
    }
    public static ColorEnum getByName(String name) {
        return Arrays.stream(ColorEnum.values()).filter(d -> d.getName().equals(name)).findFirst().orElse(Red);
    }
    public String getName() {
        return this.name;
    }
    public abstract int handler(boolean bool);
}
```

### 实现方式
其实实现起来也很简单,首先注入两个事物相关的bean

```java
@Autowird
private DataSourceTransactionManager dataSourceTransactionManager;
@Autowird
private TransactionDefinition transactionDefinition;
```

然后在对应代码处进行**事物开始/提交/回滚**
```java
public int handler(boolean bool,Student student,Teacher teacher) {
    // 开始一个事物
    TransactionStatus transactionStatus = dataSourceTransactionManager.getTransaction(transactionDefinition);
    try {
        final int insert = studentMapper.insert(student);
        if (bool) {
            throw new RuntimeException("throw exception");
        }
        final int result = teacherMapper.insert(teacher);
        // 事物提交
        dataSourceTransactionManager.commit(transactionStatus);
        return insert + result;
    } catch (Exception e) {
        // 事物回滚
        dataSourceTransactionManager.rollback(transactionStatus);
        return -1;
    }
}
```

但是因为我使用的是枚举策略,所以无法使用属性注入的方法. 改造了下,写了一个控制事物的工具类,在对应代码处直接调用即可.

```java
public class TransactionUtils {
    private static final ThreadLocal<TransactionStatus> THREAD_LOCAL = new ThreadLocal<>();

    private static final DataSourceTransactionManager DATA_SOURCE_TRANSACTION_MANAGER = SpringUtils.getBean(DataSourceTransactionManager.class);
    private static final TransactionDefinition TRANSACTION_DEFINITION = SpringUtils.getBean(TransactionDefinition.class);

    /**
     * 开始事物
     */
    public static void start() {
        THREAD_LOCAL.set(DATA_SOURCE_TRANSACTION_MANAGER.getTransaction(TRANSACTION_DEFINITION));
    }

    /**
     * 事物提交
     */
    public static void commit() {
        DATA_SOURCE_TRANSACTION_MANAGER.commit(THREAD_LOCAL.get());
        THREAD_LOCAL.remove();
    }

    /**
     * 事物回滚
     */
    public static void rollback() {
        DATA_SOURCE_TRANSACTION_MANAGER.rollback(THREAD_LOCAL.get());
        THREAD_LOCAL.remove();
    }
}
```