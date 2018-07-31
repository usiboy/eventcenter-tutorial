# 加强事件的传送

事件在传送的过程中可能会出现如下几个故障：
 * 接收端未正常启动，无法接收事件，导致发送端无法将事件传送过来；
 * 接收端成功启动之后，需要重新接收之前发送端发送失败的事件，没有这个机制会导致事件漏发的问题；
 * 当某个接收端的事件量过高，不能及时处理事件时，需要将事件的流量进行负载均衡；
 
事件中心支持Store And Forward机制，离线推送机制，能够解决第一和第二个问题，事件中心消费端的发现机制是基于dubbo的发现机制封装的，第三个问题也能够支持，如下是发布端的配置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xmlns:aop="http://www.springframework.org/schema/aop"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

	<!-- 这里只会初始化Boss类 -->
	<context:component-scan base-package="eventcenter.tutorial.section1_6.boss"></context:component-scan>
	<!-- 如果要使用@EventPoint，需要配置这个扫描器，并开启aop -->
	<context:component-scan base-package="eventcenter.api"></context:component-scan>

	<aop:aspectj-autoproxy proxy-target-class="true" />

	<!-- 这个是最基础的配置，默认初始化DefaultEventCenter实例 -->
	<!-- group是用来确定不同事件中心的节点的分组的标示，未来在发现事件中心消费点时，事件只会在同一个group下的节点中进行传播 -->
	<conf group="example" xmlns="http://code.eventcenter.com/schema/ec" xsi:schemaLocation="http://code.eventcenter.com/schema/ec http://code.eventcenter.com/schema/ec/eventcenter.xsd">
		<!-- dubbo节点需要配置dubbo的application、registry和protocol基本信息，由于这个节点不需要接收其他模块的事件，所以不需要设置protocol前缀的属性 -->
		<dubbo registryAddress="multicast://224.5.6.7:1236?unicast=false" applicationName="example-boss" applicationOwner="jueming">
			<dubboPublish>
				<!-- 定义个 -->
				<dubboPublishGroup remoteEvents="boss.task.submit">
					<eventTransmission version="manager-example-1" />
				</dubboPublishGroup>
			</dubboPublish>
		</dubbo>
		<!-- 要支持离线推送功能，只需要添加如下的配置 -->
		<saf>
			<leveldbSaf checkInterval="10000" storeOnSendFail="true"></leveldbSaf>
		</saf>
	</conf>
</beans>
```

SAF既Store and Forward的缩写，事件发送端支持离线推送的功能，这里有两种实现方式，最可靠的是leveldb的存储，事件发送失败后，会通过leveldb落地到磁盘中，防止应用重启或者系统宕机导致事件丢失的问题。参数checkInterval表示检查接收端的心跳间隔，每次检查如果发现接收端可用，则会立即从离线队列中获取事件，并推送到接收端中。