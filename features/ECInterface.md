# 事件中心接口
事件中心最核心的模块为ec-api
```xml
<dependency>
    <groupId>io.eventcenter</groupId>
    <artifactId>ec-api</artifactId>
    <version>${latestVersion}</version>
</dependency>
```
最核心的接口为EventCenter，如下：
```java
public interface EventCenter {

	/**
	 * 通过事件名称，查找到事件注册者
	 * @param name
	 * @return
	 */
	EventRegister findEventRegister(String name);

	/**
	 * 触发事件，和方法{@link #fireEvent(Object, EventInfo)}类似，这个方法只需要传递target，事件名称和参数
	 * @param target
	 * @param eventName
	 * @param args
	 * @return
	 */
	Object fireEvent(Object target, String eventName, Object... args);

	/**
	 * 触发事件，将事件通知给对应的事件订阅者
	 * @param target 触发事件的目标
	 * @param eventInfo 事件体，包含了事件名称，事件参数，事件编号
	 * @return
	 */
	Object fireEvent(Object target, EventInfo eventInfo);
	
	/**
	 * 触发事件，将事件通知给对应的事件订阅者
	 * @param target 触发事件的目标
	 * @param eventInfo 事件体，包含了事件名称，事件参数，事件编号
	 * @param result 事件触发之后产生的回执数据
	 * @return 通过listener返回数据
	 */
	Object fireEvent(Object target, EventInfo eventInfo, Object result);
}
```
接口主要包含两个功能：
 * findEventRegister 查找事件的注册者，事件中心默认的实现是不需要去管理这个方法，默认会将事件注册为CommonEventSource，也就是在EventListener中onObserved的方法传递进来的事件类型。如果有特殊需要，这里可以定制化事件源的类型，比如做成TradeEvent，这个时候需要注入自定义的EventRegister，目前版本尚未完全开放这个功能，后续版本将会支持；
 * fireEvent 事件触发的方法，包含了三个方法，最终会调用到最下面那个。
 
## 构建事件中心
事件中心提供了构建工厂的API，Maven引用如下：
```xml
<dependency>
    <groupId>io.eventcenter</groupId>
    <artifactId>ec-builder</artifactId>
    <version>${latestVersion}</version>
</dependency>
```
构建器关联了事件中心的多个核心模块，所以基本上在项目工程中只需要依赖ec-builder即可传递依赖其他ec的模块了。

构建很简单，使用`EventCenterBuilder`就可以创建事件中心，如果使用Spring，可以使用`@Configuration`或者`@Bean`构建出EventCenter实例，如果还不觉得满足？可以使用Spring Schema的方式构建，如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans p http://www.springframework.org/schema/context/spring-context-3.2.xsd">
​
    <!-- 这个是最基础的配置，默认初始化DefaultEventCenter实例 -->
    <conf xmlns="http://code.eventcenter.com/schema/ec" xsi:schemaLocation="http://code.eventcenter.com/schema/ec http://code.eventcenter.com/schema/ec/eventcenter.xsd">
​
    </conf>
</beans>
```
关于参数的设置，可以参考[配置和示例](/examples/README.md)

## 引入事件中心
WEB系统中大多数都会使用Spring框架，基于Spring架构的代码，可以通过`@Resource`或者`@Autowired`的方式植入`EventCenter`接口
```java
    @Resource
    EventCenter eventCenter;

    public void created(String tid){
        // ...业务代码省略
        eventCenter.fireEvent(this, "trade.created", tid);
    }
```

以下是使用了注解的方式：
```java
    
    @EventPoint("trade.created")
    public void created(String tid){
        // ...业务代码省略
    }
```
两种方式各有利弊，直接代码引入eventCenter使用比较灵活，能够方便的传入事件的参数，可以在代码中任何一处触发事件；使用注解的优势在于不会入侵代码，代码可读性和维护性高，缺点也比较明显，参数不容易传递，会比较被动，未来版本会改善参数的传递的方式，关于注解方式的配置方式请参考[事件注解的说明](/examples/Chapter4.8.md)。

## 订阅事件
### 使用EventCenterBuilder注册事件
首先我们先看下如何实现一个事件的监听器，也就是事件消费者：
```java
/**
* 这里需要添加一个注解，用于描述这个监听器所监听的事件，可以支持多个事件同时监听
*/
public class SimpleEventListener implements EventListener {
    
    @Override
    public void onObserved(CommonEventSource source) {
        // source中传递了事件触发时所带的参数，这里可以原封不动的按照顺序取出
        System.out.println("Hello " + source.getArg(0, String.class));
    }
}
```

然后，将他添加到构建器中：
```java
    DefaultEventCenter eventCenter = new EventCenterBuilder()
                    // 这里需要将监听器注入进来
                    .addEventListener("example.saidHello", new SimpleEventListener())
                    .build();
```

事件中心支持在一个节点下，一个事件能够同时被多个订阅器消费，如下：
```java
    DefaultEventCenter eventCenter = new EventCenterBuilder()
                    // 这里需要将监听器注入进来
                    .addEventListener("example.saidHello", new SimpleEventListener())
                    .addEventListener("example.saidHello", new SimpleEventListener2())
                    .addEventListener("example.saidHello", new SimpleEventListener3())
                    .build();
```

`SimpleEventListener`类本身没有关联任何事件，而是在构建器时关联了事件，这种方式对于数量较小的事件消费者规模来看比较容易管理，但是一旦系统比较大，涉及到不同的业务模块时，这个构建器管理监听器会有些复杂，因为他要承担所有消费者的关联关系，所以可以把这个管理职责转移到监听器上。监听器是最了解他自己本身需要什么样的事件，事件中心提供了`@ListenerBind`注解，这个只需要修饰在监听器上即可，构建器只需要加入这个监听器，如下：
```java
@ListenerBind("example.saidHello")
public class SimpleEventListener implements EventListener {
    // 业务代码省略.....
}
```

```java
DefaultEventCenter eventCenter = new EventCenterBuilder()
                    // 这里需要将监听器注入进来
                    .addEventListener(new SimpleEventListener())
                    .build();
```

这样是不是更简单了？

### 使用Spring Schema注册事件
除了配置事件中心本身，加载监听器可以交给Spring的context的扫描进行处理，以及事件的过滤器也可以使用这种方式进行处理，我们可以在监听器上标记Spring的注解，如下:
```java
@Component
@ListenerBind("example.saidHello")
public class SimpleEventListener implements EventListener {
    // 业务代码省略.....
}
```
加入了`@Component`后，并在spring配置中加入contenxt-scan的配置，就可以自动的将监听器注入到事件中心中：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans p http://www.springframework.org/schema/context/spring-context-3.2.xsd">
​
    <!-- 假设监听器存放在这个package中 -->
    <context:component-scan base-package="eventcenter.listeners"></context:component-scan>

    <!-- 这个是最基础的配置，默认初始化DefaultEventCenter实例 -->
    <conf xmlns="http://code.eventcenter.com/schema/ec" xsi:schemaLocation="http://code.eventcenter.com/schema/ec http://code.eventcenter.com/schema/ec/eventcenter.xsd">
​
    </conf>
</beans>
```


## 事件过滤器
过滤器是在处理监听器之前或者监听器处理完之后的一个过滤器的功能。过滤器的功能请查看如下图所示：
<img src="/images/ecfilter.png" alt="事件中心监听器的过滤器" align="center" />

如下是过滤器的接口：
```java
public interface ListenerFilter {

    /**
     * execute filter before listener invoked
     * @param listener
     * @param evt
     * @return 如果过滤器执行处理正常，应该返回true，如果返回false，那么事件和后置过滤器都不会执行
     */
    boolean before(EventListener listener, CommonEventSource evt);

    /**
     * execute filter after listener invoked
     */
    void after(ListenerReceipt receipt);
}
```

实现时可以继承`ListenerFilterAdapter`类，事件中心设计多种过滤器，还有`EventFireFilter`、`PublishFilter`和`SubscribFilter`，每种Filter是如何被调用，可以参考[事件触发机制](ECFireInterface.md)。

事件监听器过滤器目前支持两种拦截，一种是所有事件的拦截，事件日志的监控是基于全局过滤器实现的，另一种是关联某些事件进行拦截。

### 使用EventCenterBuilder注册过滤器
定义一个事件过滤器，然后将它设置到`EventCenterBuilder`中：
```java
DefaultEventCenter eventCenter = new EventCenterBuilder()
                .addEventListeners(InitBuilder.buildEventListeners())
                .addGlobleFilter(new ListenerFilter() {
                    @Override
                    public boolean before(EventListener listener, CommonEventSource evt) {
                        System.out.println("我是全局过滤前置器:" + evt.getSource()==null?"":evt.getSource().getClass());
                        return true;
                    }

                    @Override
                    public void after(ListenerReceipt receipt) {
                        System.out.println("我是全局过滤后置器,took:" + receipt.getTook() + " ms");
                    }
                })
                .addListenerFilter("example.manual", new ListenerFilter() {
                    @Override
                    public boolean before(EventListener listener, CommonEventSource evt) {
                        System.out.println("我是单个过滤前置器:" + evt.getSource()==null?"":evt.getSource().getClass());
                        return true;
                    }

                    @Override
                    public void after(ListenerReceipt receipt) {
                        System.out.println("我是单个过滤后置器,took:" + receipt.getTook() + " ms");
                    }
                })
                .addEventFireFilter(new EventFireFilter() {
                    @Override
                    public void onFired(Object target, EventInfo eventInfo, Object result) {
                        System.out.println(target.getClass() + "准备触发事件:" + eventInfo.getName());
                    }
                }).build();
```

这个示例定义了全局过滤器、example.manual事件的过滤器和触发过滤器。全局过滤器和example.manual过滤器不冲突，这两个会依次执行，也就是说一个事件能够同时被多个过滤器过滤。

这里有一点需要注意，拦截器是阻塞调用，实现拦截器需要考虑程序的性能问题，需谨慎使用。

### 使用Spring Schema注册过滤器
对于事件监听器过滤，这里可以使用过滤器的注解：`@EventFilterable`：
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface EventFilterable {

    /**
     * 声明这个过滤器需要和哪些事件进行关联，如果要关联多个事件，请使用','分割。
     * @return
     */
    String value() default "";

    /**
     * 是否为全局过滤器，全局过滤器将会为每个事件都执行一遍过滤，如果isGlobal为false，并且value为空字符串，那么启动事件中心将会报错
     * @return
     */
    boolean isGlobal() default false;
}
```

例如实现一个example.manual的事件监听过滤器：
```java
@EventFilterable("example.manual")
@Component    // 如果使用Spring，需要设置spring的context-scan，并将这个包配置到扫描包中
public class SingleFilter extends ListenerFilterAdapter {
    
}
```

实现一个全局的事件监听过滤器：
```java
@EventFilterable(isGlobal = true)
@Component    // 如果使用Spring，需要设置spring的context-scan，并将这个包配置到扫描包中
public class GlobalFilter extends ListenerFilterAdapter {
    
}
```

然后只需要将这两个过滤加入到spring的context-scan中，即可实现自动注入到事件中心的过滤器注册表中，是不是So Easy?

如果想进一步了解一个事件是如何从fireEvent到达EventListener的，可以阅读下一小结。

