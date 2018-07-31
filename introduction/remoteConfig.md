# 远程传送事件

远程传送事件可以把它理解为分布式事件，两个独立部署的A、B两个系统，可以直接将A --> B，也可以间接的由B订阅A的事件，动态接收事件。这里使用了[dubbo](https://dubbo.incubator.apache.org/#!/?lang=zh-cn)来完成了自动发现和RPC通讯的功能。动态订阅能够极大的降低A、B两个模块的耦合度，统一由事件中心进行管理。

这一章节依然使用第三章节使用到的两个领域，Boss和Manager，只是这两个领域分别放在两个不同的进程中启动，如下是Boss模块下的Spring配置:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xmlns:aop="http://www.springframework.org/schema/aop"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

	<!-- 这里只会初始化Boss类 -->
	<context:component-scan base-package="eventcenter.tutorial.section1_5.boss"></context:component-scan>
	<!-- 如果要使用@EventPoint，需要配置这个扫描器，并开启aop -->
	<context:component-scan base-package="eventcenter.api"></context:component-scan>

	<aop:aspectj-autoproxy proxy-target-class="true" />

	<!-- 这个是最基础的配置，默认初始化DefaultEventCenter实例 -->
	<!-- group是用来确定不同事件中心的节点的分组的标示，未来在发现事件中心消费点时，事件只会在同一个group下的节点中进行传播 -->
	<conf group="example" xmlns="http://code.eventcenter.com/schema/ec" xsi:schemaLocation="http://code.eventcenter.com/schema/ec http://code.eventcenter.com/schema/ec/eventcenter.xsd">
		<!-- dubbo节点需要配置dubbo的application、registry和protocol基本信息，由于这个节点不需要接收其他模块的事件，所以不需要设置protocol前缀的属性 -->
		<dubbo registryProtocol="multicast" registryAddress="224.5.6.7:1234" applicationName="example-boss" applicationOwner="jueming">
			<dubboPublish>
				<!-- 定义个 -->
				<dubboPublishGroup remoteEvents="boss.task.submit">
					<eventTransmission version="manager-example-1" />
				</dubboPublishGroup>
			</dubboPublish>
		</dubbo>
	</conf>
</beans>
```

配置中定义了dubbo模块，设置了dubbo两个关键配置，registry和application，同时设置了dubboPublish，一个dubboPublish可以对应多个dubboPublishGroup，一个group中对应着一个远程端的reference，其中的version是表示provider的version，remoteEvents表示当前模块触发的事件将会发送到关联的provider。

接下来看下manager模块下的Spring配置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xmlns:aop="http://www.springframework.org/schema/aop"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

	<!-- 这里只会初始化Manager和ManagerEventListener两个类 -->
	<context:component-scan base-package="eventcenter.tutorial.section1_5.manager"></context:component-scan>
	<!-- 如果要使用@EventPoint，需要配置这个扫描器，并开启aop -->
	<context:component-scan base-package="eventcenter.api"></context:component-scan>

	<aop:aspectj-autoproxy proxy-target-class="true" />

	<!-- 这个是最基础的配置，默认初始化DefaultEventCenter实例 -->
	<!-- group是用来确定不同事件中心的节点的分组的标示，未来在发现事件中心消费点时，事件只会在同一个group下的节点中进行传播 -->
	<conf group="example" xmlns="http://code.eventcenter.com/schema/ec" xsi:schemaLocation="http://code.eventcenter.com/schema/ec http://code.eventcenter.com/schema/ec/eventcenter.xsd">
		<!-- dubbo节点需要配置dubbo的application、registry和protocol基本信息，由于这个节点不需要接收其他模块的事件，所以不需要设置protocol前缀的属性 -->
		<dubbo registryProtocol="multicast" registryAddress="224.5.6.7:1234" applicationName="example-manager" applicationOwner="jueming">
			<!-- 这个点主要是manager监听老板模块的boss.task.submit，所以创建一个dubboSubscribe节点，并打上版本，boss模块的publish的version会与之对应 -->
			<dubboSubscribe version="manager-example-1" ></dubboSubscribe>
		</dubbo>
	</conf>
</beans>
```

manager模块需要订阅boss模块的事件，这里配置了dubboSubscribe，并设置了version，这里将会初始化一个dubbo的provider，group为example，version为manager-example-1，用于开启dubbo协议的端口，接收boss端的事件。


