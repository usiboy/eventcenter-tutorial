# 加强事件运行容器

事件中心的所有消费者都会由事件运行容器进行分配和管理，目前容器的实现有两种，第一种是默认的运行容器：eventcenter.api.async.simple.SimpleQueueEventContainer，第二种是使用leveldb封装的queue，eventcenter.leveldb.LevelDBContainer。

 * SimpleQueueEventContainer使用的队列是内存中的队列，存储速度快，缺点是，没有持久事件，并且当消息量较大时，队列元素随之会增加，导致内存增大引起内存溢出的问题，这个队列适合小型的应用或者是开发环境下使用；
 * LevelDBContainer使用的是leveldb作为事件的存储，并通过queue的封装，使它成为一个具有持久化的队列工具，leveldb存取速度快，高效，能够保障事件的收发一致性，即使服务器宕机了，事件依然不会丢失；并且能够在事件洪流中起到限流功能，换着一部分的事件放在持久化队列中，由容器按照并发能力依次消费；
 
 如何配置leveldb的持久化运行容器？
 ```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd">

	<context:component-scan base-package="eventcenter.tutorial.section1_4"></context:component-scan>

	<!-- 这个是最基础的配置，默认初始化DefaultEventCenter实例 -->
	<conf xmlns="http://code.eventcenter.io/schema/ec" xsi:schemaLocation="http://code.eventcenter.io/schema/ec http://code.eventcenter.io/schema/ec/eventcenter.xsd">
		<queue>
			<!--
			 corePoolSize		初始化运行容器中的线程池的最小活跃线程数
			 maximumPoolSize	初始化运行容器中的线程池的最大活跃线程数
			 openTxn			加强事件运行时的数据一致性，防止系统异常导致事件丢失的问题，默认为false
			 -->
			<leveldbQueueContainer corePoolSize="1" maximumPoolSize="4" openTxn="true" ></leveldbQueueContainer>
		</queue>
	</conf>
</beans>
```

这个示例配置中的类是按照章节一中的示例演示。配置中多了queue的设置，这里只需要改下配置，事件中心便可支持持久化的功能。

以下是使用构建器的方式创建，如下：
```java
/**
 * 增加了leveldb的事件中心运行容器的配置
 **/
public class BuilderMain {

    public static void main(String[] args) throws Exception {
        // 配置LevelDB的运行容器
        LevelDBContainerFactory levelDBContainerFactory = new LevelDBContainerFactory();
        // 初始化运行容器中的线程池的最小活跃线程数
        levelDBContainerFactory.setCorePoolSize(1);
        // 初始化运行容器中的线程池的最大活跃线程数
        levelDBContainerFactory.setMaximumPoolSize(Runtime.getRuntime().availableProcessors());
        // 加强事件运行时的数据一致性，防止系统异常导致事件丢失的问题，默认为false
        levelDBContainerFactory.setOpenTxn(true);
        DefaultEventCenter eventCenter = new EventCenterBuilder()
                // 这里需要将监听器注入进来，后续章节会介绍如何自动注入监听器
                .addEventListener(new SimpleEventListener())
                // 将leveldb的容器注入进来
                .queueContainerFactory(levelDBContainerFactory)
                .build();
        // 启动事件中心容器
        eventCenter.startup();
        // 接着开始触发事件，打一个招呼
        eventCenter.fireEvent("Main", "example.saidHello", "World");
        // 跟着观察下控制台的输出，这里暂停500ms
        Thread.sleep(500);
        // 最后关闭掉容器
        eventCenter.shutdown();
    }
}
```

运行后的效果和Spring的配置方式一样，使用构建器的方式会比较自由的控制配置，不会受限于spring schema的约束，如果系统中使用到Spring Configuration的方式初始化Bean，那可以将这个构建器的逻辑写到Java类中进行控制。