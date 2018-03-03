# miaosha
Java秒杀系统

# 初始情况

## 初始项目情况

**后端**：SpringBoot，MyBatis，JSR303

**前端**：Thymeleaf，Bootstrap，Jquery

**中间件**：Druid，Redis，RabbitMQ

## 服务器情况

**服务器类型**：阿里云轻量应用服务器

**CPU核心数**：单核

**内存**：2G

**磁盘**：40G

## 初始QPS

用一段代码往数据库里插入了1500条用户信息，并生成相应的用户Token，保存在tokens.txt中

使用JMeter来进行压力测试，由于我的阿里云服务器配置较低，压测的时候出现很多问题，而且不能测试太多线程

所以，我就改用我的的Windows电脑进行测试，采用的也是本地MySQL和本地Redis

QPS：是指每秒内查询次数,比如执行了select操作,相应的qps会增加。QPS = 总请求数 / ( 进程总数 *   请求时间 )

### 压测商品列表

1500个线程*10次，平均QPS = 1076/sec

### 压测秒杀接口

自定义变量模拟多用户

1500个线程*10次，平均QPS = 652/sec

# 开始优化

并发的瓶颈在数据库，如何减少对数据库的访问呢？最有效的办法就是加缓存

## 页面优化

- ### 页面缓存 + 对象缓存

   （**优化了商品列表页面**）我们访问一个页面的时候不是直接让系统帮我们渲染，而是首先从缓存里面取，如果说找到了，直接返回给客户端，如果说没有，那么我们手动来渲染这个模版，渲染出来后把结果返回给客户端，同时把结果缓存到Redis里面，下次可以直接从这里面取。**更新缓存**：先把数据存到数据库中，成功后，再让缓存失效。
   
   ```java
   //手动渲染
   SpringWebContext springWebContext = new SpringWebContext(request, response, request.getServletContext(),
           request.getLocale(), model.asMap(), applicationContext);
   String html = thymeleafViewResolver.getTemplateEngine().process("goods_list", springWebContext);
   if (!StringUtils.isEmpty(html)) {
       redisService.set(GoodsKey.getGoodsList, "", html);
   }
   ```

- ### 页面静态化，前后端分离
 
   （**优化了商品详情和订单详情页面**）页面都是纯HTML代码，通过JS和Ajax来请求服务端，拉取数据渲染页面，浏览器可以把HTML缓存在客户端，这样页面数据就不用重复下载，只需要下载动态的数据就可以。

- ### 静态资源优化（未实现）

   优化方式：1. JS和CSS压缩，减少流量。2. 多个JS和CSS组合，减少连接数。

- ### CDN优化（未实现）

   CDN即内容分发网络。整个网络上有很多节点，它会把每个数据在每个节点上做一份缓存，如果用户请求的时候，它会根据用户的位置，把请求定位到离用户最近的CDN镜像中，这就导致用户总是访问离自己最近的那个镜像。
   
## 解决超卖

- ### 数据库加唯一索引：防止用户重复购买

- ### SQL加库存数量判断：防止库存变成负数

## 接口优化

- ### Redis预减库存减少数据库访问

- ### 内存标记减少Redis访问

- ### RabbitMQ队列缓冲，异步下单，增强用户体验

  - 系统初始化时，把商品库存数量加载到Redis。
  - 收到请求，Redis预减库存，库存不足，直接返回，否则继续。（例如：库存有10个，过来10个请求，把库存减为0，第11个请求过来就不用往下走，直接返回秒杀失败）。 
  - 如果有库存，请求入队，立即返回排队中

## 安全优化

- ### 秒杀接口地址隐藏

- ### 数学公式验证码

- ### 接口防刷

## 其他项目亮点

- ### 使用JSR303参数检验

- ### 使用Redis实现分布式Session

# 优化后压测

下面的压测都是在安全优化之前进行的

### 压测商品列表

1500个线程*10次，平均QPS = 2315/sec

优化后QPS提高为原来的2倍，但仍然有一些不足

由于我将商品列表的HTML页面缓存到Redis，并设置了有效期为60秒，也就是说，60秒之内该页面的商品数量是不会发生改变的

### 压测秒杀接口

自定义变量模拟多用户

1500个线程*10次，平均QPS = 652/sec

