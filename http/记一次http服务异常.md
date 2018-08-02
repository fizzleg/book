# 记一下http服务出现 curl: (18) transfer closed with outstanding read data remaining
## 环境：
* http 1.1
* spring-boot-starter-web 2.0.2.RELEASE
* spring-boot-starter-jetty 2.0.2.RELEASE（jetty 9.4.10）
## 问题描述：
* 只有这一个进程的这一个接口出现这个问题
* 使用http1.1版本访问 时会出现上述问题
## 问题排查过程
* 第一步当然是google一下 transfer closed with outstanding read data remaining 这个异常由于什么原因导致的
可以参考 https://blog.csdn.net/delphiwcdj/article/details/51095945 这篇文章
由上可以知道在http1.1版本默认使用 Connection: keep-alive
* 结合前面我们知道在http1.1默认是长连接，一次请求以 0\r\n\r\n 为结束标志
* 我们可以使用tcpdump抓包看一下这次http请求的过程，发现请求是由服务端主动关闭了连接，并且每次返回的数据和正常返回的数据比缺少 0\r\n\r\n
* 通过上面我们现在有一个临时解决的方案：
- 将http版本改为1.0，不适用长连接的方式
- 或者在请求头中加上 "Connection:close" 的方式来修改默认的长连接方式
* 然后我们接着往下查：
- 只有一个进程出现问题说明这个具有偶发性
- 只有一个接口出现问题说明应该和uri有关
- 我们把请求参数去掉发现还是有问题，这时候根本没有进入业务处理，和其他的url都一样直接返回了，这时候只能在本地debug跟踪一下处理流程了，看看哪块和uri有关
* 我们再本地debug一下，可以发现还没进入业务处理的逻辑，在业务处理之外发现了有两个过滤器，并且这两个过滤器和uri相关
其中一个过滤器的代码如下：
```
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
String uri = request.getRequestURI();
timeInMillis.putIfAbsent(uri, new CircularFifoQueue<Long>(MAX_LENGTH_RESPONSE_TIME_QUEUE));
Queue<Long> times = timeInMillis.get(uri);
Long beginTime = System.currentTimeMillis();
filterChain.doFilter(request, response);
Long endTime = System.currentTimeMillis();
times.add(endTime - beginTime);
}
```
其中我们发现有一个循环队列 CircularFifoQueue，点进去发现不是线程安全的
* 同时还有一个点，当时线上这个进程的内存占用比其他进程明显要高，并且访问统计的url（/status/dashboard）会直接抛出一个内存溢出的异常
* 这时候我们需要查看线上内存的具体情况，为了避免影响线上业务，我们需要先将当前进程的流量切掉（最好找运维操作）
* 迁移完流量后，我们可以用jmap -histo {pid} 查看当时java堆中具体是哪些对象占用内存较多
* 这时候发现有一个占用较多的 ConcurrentHashMap.Node，可能和我们的问题相关了
* 我们在本地模拟一下内存溢出看看是否会导致刚刚那个问题，修改一下过滤器的代码：
```
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
String uri = request.getRequestURI();
timeInMillis.putIfAbsent(uri, new CircularFifoQueue<Long>(MAX_LENGTH_RESPONSE_TIME_QUEUE));
Queue<Long> times = timeInMillis.get(uri);
Long beginTime = System.currentTimeMillis();
filterChain.doFilter(request, response);
Long endTime = System.currentTimeMillis();
times.add(endTime - beginTime);
throw new OutOfMemoryError();
}
```
直接在最后抛出一个内存溢出的错误，这时候我们可以看到客户端的表现就会出现 curl: (18) transfer closed with outstanding read data remaining 了。
* 到这里就基本可以确定是由于非现场安全的队列在某些并发场景下导致了内存溢出导致的。
### 最终解决方案
* 将循环队列替换为我们自己实现的数组，然后用AtomicLong实现一个数组的游标去设置值
### 扩展，深入springmvc的调用链查看一下具体情况
* 我们本地debug的时候跟一下jetty的调用链
* 在jtty中分发请求的HttpChannel这个类，我们一直可以跟踪到这个方法
```
private void minimalErrorResponse(Throwable failure)
{
try
{
Integer code=(Integer)_request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
_response.reset(true);
_response.setStatus(code == null ? 500 : code);
_response.flushBuffer();
}
catch (Throwable x)
{
failure.addSuppressed(x);
abort(failure);
}
}
```
我们可以看到在请求出现异常时jetty会将response重置，并修改状态码，那为什么我们还能收到正常的返回值，并且状态码是200呢？
* 跟踪到response的reset方法里面，我们会发现最终会调用到 HttpChannel的resetBuffer 方法中判断了当前channel的状态是否为commonited状态，如果commonited为true的话，直接抛出了异常，然后在后面将jetty对应的channel关闭了。
* 接下来我们看看channel的commonited状态什么时候置位true的，我们重新debug跟踪可以发现在springmvc 处理response返回值时的 AbstractMessageConverterMethodProcessor.writeWithMessageConverters 这个方法中调用了channel的write方法，这个时候将commonited设置为true的
* 综上我们可以发现springmvc和jetty结合的处理流程存在一定的问题，springmvc处理业务返回值时已经将数据写入了jetty的channel，并且将commonited设置为true了，后续拦截器的处理只要出现异常都会导致这种数据正常返回，连接异常关闭的情况。
### 总结
* 上线新的技术框架还是必须要到线上测试充分才推广，不然会有想不到的坑
* 业务中要充分考虑并发场景