---
title: Netty+Redis开发高并发应用的一些思考(一)
date: 2016-07-27 21:40:22
tags:
- Netty
- Redis
- 并发
---

一个开发中的高并发应用原来部署在tomcat上，但这个应用基于HTTP协议，但并非tomcat所擅长的web服务；在启用了tomcat自带的nio模式后，效率还是不高，所以选择了尝试Netty。

在缓存方面，一直以来都是使用Redis，为了满足高并发的需求，Redis也需要作一些优化。

下面就简单总结一下在开发过程中的一些想法：

1. 对于Redis读写，有很大一部分的耗时是在网络IO上，尤其是Redis(集群)与应用不在一台服务器上时；此时，对于一些连续的操作，尽量使用pipeline批处理，当然前提是这一系列操作对先后顺序没有要求，因为pipeline是将命令打包一起发送，执行顺序可能没有保证的。若批量的命令对执行顺序有要求，建议用redis事务，效率还是比pipeline低很多。

2. 灵活利用lua脚本，减少Redis的网络IO。Redis尽管对Lua脚本有很多限制，但的确能提高效率，对于一些Redis原生API不能满足的批量操作，比如读取多个key再进行简单计算，如果将这些key的值分别读取到本地，再进行计算，会发生多次网络IO，那么可以用上面的pipeline，而效率更高的方法是将这些计算写成Lua脚本，使用其SHA（可以在应用初始化的时候加载所有用到的Lua脚本，保存SHA，在线计算时直接拿SHA）调用直接返回计算结果。

3. 对于我们的应用，Netty相比Tomcat更为轻量化，毕竟只是一个NIO框架，省去了不必要的中间层。值得注意的是，协议处理和业务逻辑应该尽量解耦，协议处理由Netty完成，包括TCP拆包粘包处理、HTTP协议处理、业务应用的底层协议处理，都可以编写成Netty的Handler进行处理；但业务逻辑本身的处理不建议放在Handler中，一来逻辑上架构上不清晰，耦合度太高，二来一些耗时长的业务逻辑（往往需要数据库IO）会阻塞Eventloop，阻塞后面的channel。

4. 对于Netty中的业务逻辑，我的做法是在Handler中将解析出来的请求以及一个DefaultPromise实例封装成对象，压入业务处理的等待队列中，并在Handler中增加Promise的Listener监听器监听业务处理完成的情况，完成则写入响应；
```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
	if (msg instanceof BidRequest) {
		//创建一个Promise
		DefaultPromise<BidResponse> promise = new DefaultPromise<BidResponse>(ctx.executor()) ;
		//打包成任务对象并加入处理队列
		bidQueueStack.offer(new BidMission((BidRequest)msg, promise));

		//增加监听器，等任务处理完成之后将BidResponse写入响应
		promise.addListener(new PromiseNotifier<BidResponse,DefaultPromise<BidResponse>>(){
			@Override
			public void operationComplete(DefaultPromise<BidResponse> future) throws Exception {
				if(future.isSuccess()){
					ctx.writeAndFlush(future.get());
				}
				ctx.channel().close();
			}
		});
	}
}
```
另外使用线程池管理CPU内核数个业务处理线程，从业务等待队列中获取任务对象，进行业务逻辑处理；处理完成之后通过Promise通知任务完成，并放入任务处理结果（响应）：
```java
public class BidHandleThread implements Runnable {
	private LinkedBlockingQueue<BidMission> bidQueueStack;
	private static final int DEFAULT_RANGE_FOR_SLEEP = 50; // 随机休眠时间

	public BidHandleThread(LinkedBlockingQueue<BidMission> bidQueueStack) {
		super();
		this.bidQueueStack = bidQueueStack;
	}

	@Override
	public void run() {
		try {
			while (true) {
				Random r = new Random();

				// 从队列弹出数据
				BidMission mission = null;
				if (bidQueueStack.size() > 0) {
					mission = bidQueueStack.poll();
				} else {
					Thread.sleep(r.nextInt(DEFAULT_RANGE_FOR_SLEEP));
					continue;
				}

				if (null != mission) {
					/**
          * 此处为具体的业务处理过程
          */
          //通过Promise通知任务完成
					mission.getPromise().setSuccess(adxResp);
					// 打印数据
					System.out.println("队列剩余数据数量：" + bidQueueStack.size());
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
```

5. 至于业务处理的线程池内，线程之间对数据库的访问应该还有进一步优化的空间。之前的一个设想是一个业务线程发起Redis访问的时候，把当前线程休眠，让其他线程进行数据库访问以外的业务处理（计算）；等待Redis响应后才苏醒，参与到其他线程之间对时间片的争夺。这样保证数据库IO是饱和的（应该也是业务逻辑处理中耗时最多的部分）。但还没实现。
或者将所有数据库访问都放在一个任务队列中，也是通过Promise监听-通知的方法，实现数据库的异步访问。
