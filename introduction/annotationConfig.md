# 使用注解

上一章节我们学习了Boss类中通过引用EventCenter组件，Boss具备了发送事件的能力，这一章节，将会使用@EventPoint的注解的方式加在Boss.submitTask方法中，也可以起到同样的效果，如下是Boss类代码：
```java
/**
 * 领域模型：老板 (Boss)
 **/
@Component
public class Boss implements Serializable {
    private static final long serialVersionUID = 1905777591870557841L;

    @Value("大老板")
    String name;

    private final Logger logger = Logger.getLogger(this.getClass());

    /**
     * <pre>
     * 老板提交任务的方法。
     * @EventPoint注解可以拦截方法完成之前或者之后，触发一次fireEvent的操作，也就是发事件的操作。
     * 老板任务创建完成之后，将会触发boss.task.submit的事件，事件的参数将会和submitTask方法中的参数保持一致，方法返回的result将会放入事件信息的result属性中
     * </pre>
     * @param content
     */
    @EventPoint(value = "boss.task.submit")
    public String submitTask(String content){
        logger.info(name + "创建了任务了:" + content);
        // 返回一些结果，这些结果将会被带入到事件信息的result字段中
        return name;
    }
}
```

接下来改下ManagerEventListener：
```java
/**
 * 这个是管理员监听器，目前监听了老板的任务，注意：这个类需要加上Spring Scan的注解，例如{@link Component}注解是必要的
 **/
@Component
@ListenerBind("boss.task.submit")
public class ManagerEventListener implements EventListener {

    @Resource
    Manager manager;

    @Override
    public void onObserved(CommonEventSource source) {
        // 这里和section1_2不同的是，boss的name是通过result传递过来
        manager.manageTask(source.getResult(String.class), source.getArg(0, String.class));
    }
}
```

@EventPoint的使用有优点，也有缺点。他能够减少代码的入侵性，能够比较方便的将发送事件的能力植入到领域模型中，缺点也是有的，当事件中需要传递丰富的参数时，这个注解会有些困难，对于参数比较难传递的，尽可能使用方法的返回对象中进行传递，他将会设置到CommonEventSource中的result参数中，监听器可以通过获取result得到事件的结果。

如下是spring-ec.xml的配置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xmlns:aop="http://www.springframework.org/schema/aop"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

	<context:component-scan base-package="eventcenter.tutorial.section1_3"></context:component-scan>
	<!-- 如果要使用@EventPoint，需要配置这个扫描器，并开启aop -->
	<context:component-scan base-package="eventcenter.api"></context:component-scan>

	<aop:aspectj-autoproxy proxy-target-class="true" />

	<!-- 这个是最基础的配置，默认初始化DefaultEventCenter实例 -->
	<conf xmlns="http://code.eventcenter.io/schema/ec" xsi:schemaLocation="http://code.eventcenter.io/schema/ec http://code.eventcenter.io/schema/ec/eventcenter.xsd">

	</conf>
</beans>
```

因为使用了Spring的AOP机制，所以需要开启aop的代理，同时需要实例化事件中心的切面实例，需要加入```<context:component-scan base-package="eventcenter.api"></context:component-scan>```这段代码

Manager（管理领域）不需要做任何修改，同样由监听器监听boss.task.submit的事件来调用manager的业务。@EventPoint的调用顺序可以控制，默认是在被切面的方法调用成功之后，触发事件，之后触发的能够传递方法的参数和方法响应的内容到事件中；同时也可以设置listenOrder，在被切面的方法之前调用，那么触发的事件中只会传递方法参数。具体的设置请参考注解配置这一章节。