# 使用Spring配置事件中心

前面章节构建事件中心使用的是EventCenterBuilder，并手动注入到构建器中，EventCenterBuilder支持函数式编程，在构建复杂配置的事件中心时，具有较高的可读性和维护性，同时也能够较容易的支持Spring的Configuration的Bean的管理。

事件中心框架基于Spring的schema实现了一套配置管理，其xsi的值为: http://code.eventcenter.com/schema/ec/eventcenter.xsd 。

配置如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd">

	<context:component-scan base-package="eventcenter.tutorial.section1_2"></context:component-scan>

	<!-- 这个是最基础的配置，默认初始化DefaultEventCenter实例 -->
	<conf xmlns="http://code.eventcenter.com/schema/ec" xsi:schemaLocation="http://code.eventcenter.com/schema/ec http://code.eventcenter.com/schema/ec/eventcenter.xsd">

	</conf>
</beans>
``` 

这里只需要加一个conf节点，其中还有其他属性可以设置，默认不设置，将会创建一个和前面章节所创建的事件中心实例一样。

Spring context扫描了"eventcenter.tutorial.section1_2"，这个包中按照老板与管理层的关系，做了一个简单的领域设计，并结合事件中心的监听器将之关联起来，我们先来看看Boss这个类：
```java
/**
* 领域模型：老板 (Boss)
*/
@Component
public class Boss implements Serializable {
    private static final long serialVersionUID = 1905777591870557841L;

    /**
    * 在Boss类中，引用eventCenter，让Boss具备发送事件的能力，后续章节，可以通过注解的方式发送事件，减少代码的入侵性 
    */
    @Resource
    EventCenter eventCenter;

    @Value("大老板")
    String name;

    private final Logger logger = Logger.getLogger(this.getClass());

    /**
     * 老板提交任务的方法
     * @param content
     */
    public void submitTask(String content){
        logger.info(name + "创建了任务了:" + content);
        // 老板任务创建完成之后，触发了boss.task.submit的事件，并将老板的name放在第一个参数中，content放在第二个参数中
        eventCenter.fireEvent(this, "boss.task.submit", name, content);
    }
}
```

接下来看看，Manager这个领域：
```java
/**
 * 管理层的员工
 **/
@Component
public class Manager implements Serializable {
    private static final long serialVersionUID = -5397461665049816937L;

    private final Logger logger = Logger.getLogger(this.getClass());

    @Value("管理A")
    String name;

    /**
     * 安排老板的任务
     * @param content
     */
    public void manageTask(String bossName, String content){
        logger.info(name + "开始安排" + bossName + "分配的任务:" + content);
    }
}
```

从这两个类中，Boss和Manager没有直接的关联，通过实现事件中心的监听器，监听老板发出的boss.task.submit事件，让老板间接的关联到相应的Manager中：
```java
/**
 * 这个是管理员监听器，目前监听了老板的任务，注意：这个类需要加上Spring Scan的注解，例如{@link Component}注解是必要的
 * @author liumingjian
 * @date 2018/7/29
 **/
@Component
@ListenerBind("boss.task.submit")
public class ManagerEventListener implements EventListener {

    @Resource
    Manager manager;

    @Override
    public void onObserved(CommonEventSource source) {
        manager.manageTask(source.getArg(0, String.class), source.getArg(1, String.class));
    }
}
```

这个Listener被标记了Component注解，他将会被托管在Spring的容器中管理，事件监听器（消费者）可以自由的订阅相关的事件，然后聚合相关的领域完成事件后的业务。

这一章节主要讲解了如何通过Spring的配置方式引入事件中心，下一章节将会继续使用这些领域讲解如何使用注解的方式触发事件。
