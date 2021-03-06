# 事件触发机制
事件中心本身是一个区中心化的架构风格，事件被触发后，可能会路由到本地容器中，也有可能其他模块订阅了事件，会将他传播到其他模块中消费。传播成功之后，同样在其他节点中以同样的方式消费事件。

这一节将会分三个部分讲解事件触发机制，一个是本地事件的触发和消费；另一个是事件路由的机制；最后一个是事件传播的机制。

## 本地事件触发机制
<img src="/images/eventLocalFiredChain.png" alt="本地事件触发机制" align="center" />

这个图主要描述了事件EventInfo在本地节点中，从fired开始，经历了一系列的流程之后的一个生命周期。他分为两个阶段，第一个阶段是入队列，第二个阶段是出队列并在容器内找出监听器消费。

图中包含了两种监听器：`ListenerFilter`和`EventFireFilter`，fireEvent的性能首先在于入队列这里，其次就是监听器了，监听器的调用是阻塞调用，所以在使用监听器时需要考虑下fireEvent的性能。

`EventContainer`在启动时开启线程，实现观察者的模式监听消息队列。

## 事件路由机制

## 事件传播机制