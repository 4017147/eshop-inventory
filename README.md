# eshop-inventory
解决数据库缓存双写不一致



更新数据的时候，根据数据的唯一标识，将操作路由之后，发送到一个jvm内部的队列中

读取数据的时候，如果发现数据不在缓存中，那么将重新读取数据+更新缓存的操作，根据唯一标识路由之后，也发送同一个jvm内部的队列中

一个队列对应一个工作线程

每个工作线程串行拿到对应的操作，然后一条一条的执行

这样的话，一个数据变更的操作，先执行，删除缓存，然后再去更新数据库，但是还没完成更新

此时如果一个读请求过来，读到了空的缓存，那么可以先将缓存更新的请求发送到队列中，此时会在队列中积压，然后同步等待缓存更新完成

这里有一个优化点，一个队列中，其实多个更新缓存请求串在一起是没意义的，因此可以做过滤，如果发现队列中已经有一个更新缓存的请求了，那么就不用再放个更新请求操作进去了，直接等待前面的更新操作请求完成即可

待那个队列对应的工作线程完成了上一个操作的数据库的修改之后，才会去执行下一个操作，也就是缓存更新的操作，此时会从数据库中读取最新的值，然后写入缓存中

如果请求还在等待时间范围内，不断轮询发现可以取到值了，那么就直接返回; 如果请求等待的时间超过一定时长，那么这一次直接从数据库中读取当前的旧值

int h;
return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);

(queueNum - 1) & hash

1、线程池+内存队列初始化

@Bean
public ServletListenerRegistrationBean servletListenerRegistrationBean(){
    ServletListenerRegistrationBean servletListenerRegistrationBean = new ServletListenerRegistrationBean();
    servletListenerRegistrationBean.setListener(new InitListener());
    return servletListenerRegistrationBean;
}

java web应用，做系统的初始化，一般在哪里做呢？

ServletContextListener里面做，listener，会跟着整个web应用的启动，就初始化，类似于线程池初始化的构建

spring boot应用，Application，搞一个listener的注册

2、两种请求对象封装

3、请求异步执行Service封装

4、请求处理的工作线程封装

5、两种请求Controller接口封装

6、读请求去重优化

如果一个读请求过来，发现前面已经有一个写请求和一个读请求了，那么这个读请求就不需要压入队列中了

因为那个写请求肯定会更新数据库，然后那个读请求肯定会从数据库中读取最新数据，然后刷新到缓存中，自己只要hang一会儿就可以从缓存中读到数据了

7、空数据读请求过滤优化

可能某个数据，在数据库里面压根儿就没有，那么那个读请求是不需要放入内存队列的，而且读请求在controller那一层，直接就可以返回了，不需要等待

如果数据库里都没有，就说明，内存队列里面如果没有数据库更新的请求的话，一个读请求过来了，就可以认为是数据库里就压根儿没有数据吧

如果缓存里没数据，就两个情况，第一个是数据库里就没数据，缓存肯定也没数据; 第二个是数据库更新操作过来了，先删除了缓存，此时缓存是空的，但是数据库里是有的

但是的话呢，我们做了之前的读请求去重优化，用了一个flag map，只要前面有数据库更新操作，flag就肯定是存在的，你只不过可以根据true或false，判断你前面执行的是写请求还是读请求

但是如果flag压根儿就没有呢，就说明这个数据，无论是写请求，还是读请求，都没有过

那这个时候过来的读请求，发现flag是null，就可以认为数据库里肯定也是空的，那就不会去读取了

或者说，我们也可以认为每个商品有一个最最初始的库存，但是因为最初始的库存肯定会同步到缓存中去的，有一种特殊的情况，就是说，商品库存本来在redis中是有缓存的

但是因为redis内存满了，就给干掉了，但是此时数据库中是有值得

那么在这种情况下，可能就是之前没有任何的写请求和读请求的flag的值，此时还是需要从数据库中重新加载一次数据到缓存中的

8、深入的去思考优化代码的漏洞

我的一些思考，如果大家发现了其他的漏洞，随时+我Q跟我交流一下

一个读请求过来，将数据库中的数刷新到了缓存中，flag是false，然后过了一会儿，redis内存满了，自动删除了这个额缓存

下一次读请求再过来，发现flag是false，就不会去执行刷新缓存的操作了

而是hang在哪里，反复循环，等一会儿，发现在缓存中始终查询不到数据，然后就去数据库里查询，就直接返回了

这种代码，就有可能会导致，缓存永远变成null的情况

最简单的一种，就是在controller这一块，如果在数据库中查询到了，就刷新到缓存里面去，以后的读请求就又可以从缓存里面读了

队列

对一个商品的库存的数据库更新操作已经在内存队列中了

然后对这个商品的库存的读取操作，要求读取数据库的库存数据，然后更新到缓存中，多个读

这多个读，其实只要有一个读请求操作压到队列里就可以了

其他的读操作，全部都wait那个读请求的操作，刷新缓存，就可以读到缓存中的最新数据了

如果读请求发现redis缓存中没有数据，就会发送读请求给库存服务，但是此时缓存中为空，可能是因为写请求先删除了缓存，也可能是数据库里压根儿没这条数据

如果是数据库中压根儿没这条数据的场景，那么就不应该将读请求操作给压入队列中，而是直接返回空就可以了

都是为了减少内存队列中的请求积压，内存队列中积压的请求越多，就可能导致每个读请求hang住的时间越长，也可能导致多个读请求被hang住

