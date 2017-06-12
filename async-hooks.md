# async hooks

虽然说做后端开发也有好几年了，但是真正熟悉的只是`node.js`，对于其它的语言都只停留在初级阶段。第一次了解到`thread local`是在系统接入`zipkin`的时候，另外一个后端开发的同事在抱怨不知道在`node.js`中怎么可以方便的传递`traceId`，以及和我讲解了在其它语言中`thread local`的使用。于是为了避免傻傻的将`traceId`一直当参数的传递，上网搜索了一下，居然还真有相关的实现，使用`domain(已废弃)`来模拟，我们就去研究了一下`domain`的实现，那代码还真古怪加古怪，后来还是决定放弃了，性能的统计只做关键的记录统计，不做太细化，参数的形式也还是可以接受的。

项目上线稳定运行，随着用户量的增长，慢慢日志越来越多了，这个时间才发现没有一个规范的日志简直就是一个深坑，用户反馈出问题的时候，没办法准确定位到相应的日志，只能通过把客户的所有日志都查出来，再根据时间人眼一个个排查，这实在是太累了。而`zipkin`使用得太简陋，只知道整个请求慢，但是究竟慢在哪一部分，这也没办法去精确定位，一直想解决但没有想到好的办法去解决，因此一直拖着拖着。

之后一直有关注相关的`node.js`实现，`AsyncWrap`的实现我以为还有一段时间才会真正出来，没想到`8.0`版本的更新的时候，`async_hooks`已经成为正式的内置实现模块，虽然还是处理`Experimental`的阶段，但经过试验已经完全可以使用。

## 使用场景

由于系统是`REST`服务，因此在请求发起端（浏览器或其它客户端）会生成一个`X-Request-Id(traceId)`，怎么将方便将这个`traceId`从各函数调用中取出则是解决问题的关键。

- 在每一个`io`处理中，都获取`traceId`，根据开始结束时间记录当前处理的时间，生成调用链的`Timing`

- 在每个日志输出增加前缀，输出当前`traceId`，可以根据`traceId`直接筛选出该请求处理对应的所有日志（不再需要肉眼筛选了）
