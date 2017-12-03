---
title: 基于redis的发布订阅模式
date: 2016-08-20 18:41:42
tags: redis,sub/pub
---

#### 发布订阅模式

pub/sub（publish/subscribe）发布订阅模式是基于事件中广泛使用的通信模型，其中subscriber将注册到自己所监听的事件中，只要publisher有消息或者消息改变，所有注册的subscriber就会接收到消息通知。
    
发布订阅模式应用广泛，各种消息中间件（如rabbitmq，activemq）都是基于该模式开发，同时在zookeeper中的文件配置集中管理功能中就是用了发布订阅模式，而即时聊天也可以使用这种模式。

redis作为高性能的内存数据库也支持pub/sub模式，今天就说说基于redis的发布订阅模式。

#### 准备

首先要安装redis，因本人使用win7开发，所以就直接下载了window版redis，地址 [redis on windows][1] ,直接解压运行redis.exe就按默认的配置启动。

#### 编码
本实例是基于spring的所以获取redis的bean实例都是由spring注入，首先需先配置spring配置文件spring-redis-config.xml
<!--more-->
```

    <context:property-placeholder location="classpath:application.properties"/>
    <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="maxTotal">
            <value>${redis.pool.maxActive}</value>
        </property>
        <property name="maxIdle">
            <value>${redis.pool.maxIdle}</value>
        </property>
        <property name="testOnBorrow" value="true"/>
        <property name="testOnReturn" value="true"/>
    </bean>


    <bean id = "jedisPool" class="redis.clients.jedis.JedisPool">
        <constructor-arg index="0" ref="jedisPoolConfig"/>
        <constructor-arg index="1" value="${redis.host}"/>
        <constructor-arg index="2" value="${redis.port}" type="int"/>
        <constructor-arg index="3" value="${redis.timeout}" type="int"/>
       <!-- <constructor-arg index="4" value="${redis.password}"/>-->
    </bean>


    <bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory" destroy-method="destroy">
        <property name="poolConfig" ref="jedisPoolConfig"></property>
        <property name="hostName" value="${redis.host}"></property>
        <property name="port" value="${redis.port}"></property>
        <!--<property name="password" value="${redis.password}"></property>-->
        <property name="timeout" value="${redis.timeout}"></property>
        <property name="usePool" value="true"></property>
    </bean>


    <bean id="jedisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="jedisConnectionFactory"></property>

        <property name="keySerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer" />
        </property>
        <property name="valueSerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer" />
        </property>
    </bean>


```
接着封装publisher-MQPublisher
主要封装的是该方法，其中我们可以制定自己的消息格式，方便业务操作，在这里本人封装了一个MQMessage的一个消息实体类。
```
   public void pub(String channel,String message) {
        Jedis jedis = null;
        try {
            jedis = jedisPool.getResource();
            MQMessage mqMessage = new MQMessage();
            mqMessage.setChannelName(channel);
            mqMessage.setData(message);
            mqMessage.setMsgId(channel + UUID.randomUUID());
            String mqMsg = JSON.toJSONString(mqMessage);
            jedis.publish(channel, mqMsg);
        } catch (Exception e) {
            log.error("got exception :" + e);
        } finally {
            if(jedis!=null) {
                jedis.close();
            }
        }
    }
```
要实现redis的pub/sub需要定义一个监听器，该监听器继承redis的JedisPubSub。并重写onMessage等方法来消费接收消息，因为只是个demo，所以直接用log输出表示。
```
public class MQPubListener extends JedisPubSub{

    private static Logger log= LoggerFactory.getLogger(MQPubListener.class);

    /**
     * 订阅消息后的处理
     * @param channel
     * @param message
     */
    @Override
    public void onMessage(String channel, String message) {
        log.info(channel+"......"+message);
    }

    /**
     * 取得按表达式获取的消息后的处理
     * @param pattern
     * @param channel
     * @param message
     */
    public void onPMessage(String pattern, String channel, String message) {
        log.info(pattern+"...."+channel+"...."+message);
    }

    /**
     * 初始化订阅时处理
     * @param channel
     * @param subscribedChannels
     */
    public void onSubscribe(String channel, int subscribedChannels) {
        log.info(channel+"......"+"subscirbed count:"+subscribedChannels);
    }

    /**
     * 取消订阅时处理
     * @param channel
     * @param subscribedChannels
     */
    public void onUnsubscribe(String channel, int subscribedChannels) {
        log.info(channel+"have unsubscribed"+"current subscirbed channel count is:"+subscribedChannels);
    }

    public void onPUnsubscribe(String pattern, int subscribedChannels) {
    }

    public void onPSubscribe(String pattern, int subscribedChannels) {
    }
}

```
接下来是定义subscribe方的消费者-MQComsumer
由于redis的subscribe方法时阻塞的，并且消费者的注册需要在生产者的publish前进行，所以我们消费者利用多线程来实现。
```
public class MQComsumer {

    private static Logger log= LoggerFactory.getLogger(MQComsumer.class);

    private JedisPubSub listener;
    private Jedis jedis;
    public MQComsumer(JedisPubSub listener,Jedis jedis){
        this.jedis=jedis;
        this.listener=listener;
    }

    public SubTask subscribe(String channel){
        return new SubTask(channel);
    }
    public void unSubscribe(String channel){
        jedis.del(channel);
    }
    public class SubTask implements Runnable{
        private String routeKey;
        public SubTask(String routeKey){
            this.routeKey=routeKey;
        }

        public void run() {
            log.info("sub starting ,the key is:" +routeKey);
            jedis.subscribe(listener,routeKey);
            log.info("sub end");
        }
    }
}

```
最后使用MQEngine来触发启动comsumer线程
```
public class MQEngine {

    private static Logger log= LoggerFactory.getLogger(MQEngine.class);
    private MQComsumer comsumer;

    private ExecutorService service;
    public MQEngine(MQComsumer comsumer){
        this.comsumer=comsumer;
    }

    public void sub(String channel){
        log.info("subscribe channel:"+channel);
        service= Executors.newFixedThreadPool(1);
        service.execute(comsumer.subscribe(channel));
    }
```
#### 测试
```
@RunWith(SpringJUnit4ClassRunner.class)
@WebAppConfiguration
@ContextConfiguration({"/spring-beans.xml","/spring-redis-config.xml"})
public class appTest {
    @Autowired
    private JedisPool jedisPool;
    private static final String Channel="test";
    @Test
    public void test(){
        MQPubListener listener=new MQPubListener();
        MQComsumer comsumer=new MQComsumer(listener,jedisPool.getResource());
        MQEngine engine=new MQEngine(comsumer);
        engine.sub(Channel);
        MQPublisher publisher=new MQPublisher(jedisPool);
        publisher.pub(Channel,"hello message");
    }

}
```
测试结果

> test.......{"channelName":"test","data":"hello message","msgId":"testbe6de83c-ffbf-4bbb-8553-381e41dc133f"}
消费方成功获取消息

  [1]: https://github.com/MSOpenTech/redis