---
title: ts-spring-session-oome
date: 2017-08-24 16:20:28
tags:
---

# [Trouble Shooting] spring-session OOME

## 증상
1. 2번 서버가 OutOfMemoryError로 갑자기 죽어버렸음
2. KT 로드밸런서가 2번 서버로 계속 요청을 보냄
3. 사용자의 요청이 1번 서버로만 간 경우 정상적으로 서비스를 이용
4. 사용자의 요청이 2번으로 하나라도 가면 API의 응답을 받지 못함

```log
2017-08-04 19:40:51 [gajago-front] [redisMessageListenerContainer-1] ERROR org.springframework.data.redis.listener.RedisMessageListenerContainer:handleSubscriptionException:648 - SubscriptionTask aborted with exception:
java.lang.OutOfMemoryError: unable to create new native thread
        at java.lang.Thread.start0(Native Method)
        at java.lang.Thread.start(Thread.java:714)
        at org.springframework.core.task.SimpleAsyncTaskExecutor.doExecute(SimpleAsyncTaskExecutor.java:213)
        at org.springframework.core.task.SimpleAsyncTaskExecutor.execute(SimpleAsyncTaskExecutor.java:171)
        at org.springframework.core.task.SimpleAsyncTaskExecutor.execute(SimpleAsyncTaskExecutor.java:151)
        at org.springframework.data.redis.listener.RedisMessageListenerContainer.dispatchMessage(RedisMessageListenerContainer.java:958)
        at org.springframework.data.redis.listener.RedisMessageListenerContainer.access$1400(RedisMessageListenerContainer.java:71)
        at org.springframework.data.redis.listener.RedisMessageListenerContainer$DispatchMessageListener.onMessage(RedisMessageListenerContainer.java:949)
        at org.springframework.data.redis.connection.jedis.JedisMessageListener.onPMessage(JedisMessageListener.java:43)
        at redis.clients.jedis.BinaryJedisPubSub.process(BinaryJedisPubSub.java:109)
        at redis.clients.jedis.BinaryJedisPubSub.proceedWithPatterns(BinaryJedisPubSub.java:75)
        at redis.clients.jedis.BinaryJedis.psubscribe(BinaryJedis.java:2953)
        at org.springframework.data.redis.connection.jedis.JedisConnection.pSubscribe(JedisConnection.java:2959)
        at org.springframework.data.redis.listener.RedisMessageListenerContainer$SubscriptionTask.eventuallyPerformSubscription(RedisMessageListenerContainer.java:773)
        at org.springframework.data.redis.listener.RedisMessageListenerContainer$SubscriptionTask.run(RedisMessageListenerContainer.java:740)
        at java.lang.Thread.run(Thread.java:745)
Exception in thread "http-apr-10090-Acceptor-0" java.lang.OutOfMemoryError: unable to create new native thread
        at java.lang.Thread.start0(Native Method)
        at java.lang.Thread.start(Thread.java:714)
        at java.util.concurrent.ThreadPoolExecutor.addWorker(ThreadPoolExecutor.java:950)
        at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1368)
        at org.apache.tomcat.util.threads.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:161)
        at org.apache.tomcat.util.threads.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:141)
        at org.apache.catalina.core.StandardThreadExecutor.execute(StandardThreadExecutor.java:169)
        at org.apache.tomcat.util.net.AprEndpoint.processSocketWithOptions(AprEndpoint.java:909)
        at org.apache.tomcat.util.net.AprEndpoint$Acceptor.run(AprEndpoint.java:1086)
        at java.lang.Thread.run(Thread.java:745)
```

## 원인
1. [spring-session](https://github.com/spring-projects/spring-session)에서 [spring-data-redis]를 사용할 때 [RedisMessageListenerContainer의 taskExecutor](https://github.com/spring-projects/spring-data-redis/blob/master/src/main/java/org/springframework/data/redis/listener/RedisMessageListenerContainer.java#L97)에 별도의 값을 설정하지 않아서 [SimpleAsyncTaskExecutor](https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/task/SimpleAsyncTaskExecutor.java)를 사용하고 있음
2. [SimpleAsyncTaskExecutor 문서](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/core/task/SimpleAsyncTaskExecutor.html)에서 이 클래스는 쓰레드를 재사용하지 않으니 실행시간이 짧은 많은 수의 작업에는 사용하지 말라고 권장하고 있음
    - ``NOTE: This implementation does not reuse threads! Consider a thread-pooling TaskExecutor implementation instead, in particular for executing a large number of short-lived tasks.``
3. spring-session에서 세션정보에 접근하기 위해서 redis를 사용할 때 마다 Thread가 매번 생성됨
4. 접속량이 많이지게 되면 ``OutOfMemoryError`` 발생

## 해결책
1. SimpleAsyncTaskExecutor 대신에 ThreadPool을 사용할 수 있도록 지정한다
2. 하지만 지금 사용중인 ``spring-session-1.0.2.RELEASE`` 버전에서는 taskExecutor를 별도로 지정할 수 있는 방법이 없음
3. 결국 ``spring-session-1.3.1.RELEASE`` 버전으로 업데이트
4. [패치된 내용](https://github.com/tsachev/spring-session/commit/6604bf180221028aff47f999418c0d48db72fb92)을 바탕으로 ``springSessionRedisTaskExecutor``를 지정
5. ``springSessionRedisTaskExecutor``를 지정하면 일어나는 일
  1. ``springSessionRedisTaskExecutor``를 지정하면 ``RedisHttpSessionConfiguration.redisTaskExecutor``에 지정한 ``springSessionRedisTaskExecutor``가 할당 됨 [소스코드](https://github.com/spring-projects/spring-session/blob/master/spring-session-data-redis/src/main/java/org/springframework/session/data/redis/config/annotation/web/http/RedisHttpSessionConfiguration.java#L203)
  2. ``RedisMessageListenerContainer.taskExecutor``에 ``RedisHttpSessionConfiguration.redisTaskExecutor``가 할당 됨 [소스코드](https://github.com/spring-projects/spring-session/blob/master/spring-session-data-redis/src/main/java/org/springframework/session/data/redis/config/annotation/web/http/RedisHttpSessionConfiguration.java#L93)
  3. spring-session에서 redis를 사용할 때 ``springSessionRedisTaskExecutor``를 사용하게 됨 [소스코드](https://github.com/spring-projects/spring-data-redis/blob/master/src/main/java/org/springframework/data/redis/listener/RedisMessageListenerContainer.java#L961)

```java
@Configuration
public class FrontSessionConfig {
    @Bean
    public Executor springSessionRedisTaskExecutor() {
        ThreadFactory threadFactory = new ThreadFactory() {
            private int threadCount = 0;

            @Override
            public Thread newThread(Runnable runnable) {
                return new Thread(runnable, String.format("gajago-session-redis-%d", ++threadCount));
            }
        };

        return Executors.newCachedThreadPool(threadFactory);
    }
}
```


## 남아있는 과제
1. 왜 1번 서버는 안 죽고 2번 서버만 죽었을까?
2. KT 로드밸런서가 왜 2번 서버로 계속 요청을 보냈을까?
3. 캐시로 사용하고 있는 redis의 taskExecutor는?

## 관련자료
* https://github.com/spring-projects/spring-session
* https://github.com/tsachev/spring-session/commit/6604bf180221028aff47f999418c0d48db72fb92
* https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/task/SimpleAsyncTaskExecutor.java
* https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/core/task/SimpleAsyncTaskExecutor.html
