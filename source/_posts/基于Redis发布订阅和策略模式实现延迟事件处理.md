---
title: 基于Redis发布订阅和策略模式实现延迟事件处理
date: 2023-03-10 15:48:40
categories:
  - "解决方案设计"
  - "设计模式"
tags:
  - "Redis"
  - "设计模式"
  - "策略模式"
---

### 需求

有这样的需求，用户发起一个活动(有人数限制和活动时间填写)，其他用户可以报名，报名人数满了之后就算成团，需要在活动开始之前提醒用户，活动开始时有对应业务处理，活动结束的时候有业务逻辑处理。这个功能如何设计？

### 设计

主要是基于监听 Redis 的 Key 过期事件，加上策略模式实现的。

首先活动开始时间是已知的，主要是求得活动开始之前的时间并触发事件，以及活动结束的时间并触发事件。
- 活动开始之前的业务主要是给用户发提醒，所以在定在活动时间前两三个小时之前触发。如果用户提交活动的时间和活动开始的时间相差小于1H，拒绝用户提交；如果大于6H，则在活动开始之前3H提醒；如果时间差在1～6H，则 时间差/2;
- 活动结束之后的时间程序无法把控，因为是用户的线下行为，所以直接定在了开始时间+3H。

主要实现思路
1. 在活动提交的时候向 Redis 添加一个 Key (Key的组成是：固定前缀+活动ID+策略实现 Handler 类名)，过期时间依据当前时间和活动开始时间计算而出。
2. 监听到上一步的 Key 的过期事件，给用户提醒之后，再向 Redis 中添加活动开始对应的 Key ，过期时间根据当前时间和活动开始时间计算而得。
3. 监听到上一步的 Key 过期事件，做对应的业务处理，同时再向 Redis 添加对应的 Key ，过期时间根据当前时间+3H。
4. 监听到上一部的 Key 的过期事件，做对应的业务处理，此时一个活动的流程算是结束了。

Key的组成包含三个部分
- 固定前缀，使用 `Spring MessageListener` 组件可以监听到所有 Key 过期事件，只需要以活动前缀的 Key 做业务处理就可以。
- 活动ID不用多说，每一步业务处理都需要。
- 策略实现 Handler 类名，方便在234步骤中直接找到对应的处理类，不需要再写对应的类去找处理类了。

有两个问题
Redis key 过期删除是定时+惰性，当 key 过多时，删除会有延迟，回调通知同样会有延迟。但此时有时间延迟是业务可允许的。
项目架构是集群，多个服务同时监听Redis，一个 Key 过期，所有的服务都会监听到，实际上一台服务处理就可以，所以这块使用了 Redis 分布式锁。

### 实现

- SpringBoot 和 Redis相关的版本

  |SpringBoot|Redisson|
  |--|--|
  |2.2.4.RELEASE|3.12.0|

- 定义事件处理的接口以及三个实现，实现中时间为了测试快速看到结果，直接设置为了10s

```java
interface AbstractActivityEventHandle {
    void handle(Long type);
}

@Component("activityRemindHandle")
@Slf4j
class ActivityRemindHandle implements AbstractActivityEventHandle {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate

    @Override
    void handle(Long type) {
        redisTemplate.opsForValue().set("activity_expire_event:123:activityStartHandle", "1", 10, TimeUnit.SECONDS)
        log.info("activity remind handle...{}", type)
    }
}

@Slf4j
@Component("activityStartHandle")
class ActivityStartHandle implements AbstractActivityEventHandle {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate

    @Override
    void handle(Long type) {
        redisTemplate.opsForValue().set("activity_expire_event:123:activityEndHandle", "1", 10, TimeUnit.SECONDS)
        log.info("activity start handle...{}", type)
    }
}


@Slf4j
@Component("activityEndHandle")
class ActivityEndHandle implements AbstractActivityEventHandle {
    @Override
    void handle(Long type) {
        log.info("activity end handle...{}", type)
    }
}


```

- 创建消息监听器

```java
@Component
class ActivityKeyExpiredListener extends KeyExpirationEventMessageListener {
    @Autowired
    private RedissonClient redissonClient

    @Autowired
    private Map<String, AbstractActivityEventHandle> activityHandlerMap;

    ActivityKeyExpiredListener(RedisMessageListenerContainer listenerContainer) {
        super(listenerContainer)
    }

    @Override
    void onMessage(Message message, @Nullable byte[] pattern) {
        super.onMessage(message, pattern)
        // 获取到失效的 key，进行取消订单业务处理

        final String key = message.toString()
        if (key.startsWith("activity_expire_event:")) {
            final String activityId = key.replace("activity_expire_event:", "");
            final String[] strings = activityId.split(":");
            //使用分布式锁，防止业务重复处理
            final RLock lock = this.redissonClient.getLock("activity_handler_lock_" + activityId);
            final boolean b = lock.tryLock();
            try {
                if (b) {
                    //策略模式处理对应的事件
                    this.activityHandlerMap.get(strings[1]).handle(Long.valueOf(strings[0]));
                }
            } catch (final Exception e) {
                e.printStackTrace();
            } finally {
                if (b) {
                    lock.unlock();
                }
            }
        }
    }
}
```

- 创建监听器容器以及 Redis 模板

```java
@Configuration
class RedisListenerConfig {
    @Bean
    RedisMessageListenerContainer redisMessageListenerContainer(RedisConnectionFactory redisConnectionFactory) {
        final RedisMessageListenerContainer redisMessageListenerContainer = new RedisMessageListenerContainer()
        redisMessageListenerContainer.setConnectionFactory(redisConnectionFactory)
        return redisMessageListenerContainer
    }
}

@EnableCaching
@Configuration
@AutoConfigureBefore(value = [RedisAutoConfiguration.class, RedissonAutoConfiguration.class])
class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(final RedisConnectionFactory redisConnectionFactory) {
        final RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new JdkSerializationRedisSerializer());
        redisTemplate.setHashValueSerializer(new JdkSerializationRedisSerializer());
        redisTemplate.setEnableTransactionSupport(true);
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }
}
```

- 创建测试入口

```java
@RestController
@RequestMapping("/groovy")
class TestGroovyController {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate

    @GetMapping("/submit")
    String submit() {
        // key的组成有三部分
        redisTemplate.opsForValue().set("activity_expire_event:123:activityRemindHandle", "1", 10, TimeUnit.SECONDS)
        return "success"
    }
}
```

- 执行结果

![](https://cdn.jsdelivr.net/gh/Killy412/killy412.github.io@hexo/source/images/2023-03-11_01-14-05.png)


### 实现参考

- [Spring Boot 监听 Redis Key 失效事件实现定时任务](https://mp.weixin.qq.com/s/tMR0-Oxw99XtF1KBTGrnAg)


