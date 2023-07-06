---
layout: post
title: 一次生产OOM分析解决过程
categories: [Java]
description: 一次生产OOM分析解决过程
keywords: Java, JVM
---

上周，公司生产环境突然有用户返回IM服务连接不上导致无法收发消息。出现问题运维重启了IM的服务，之后恢复正常。事后分析事故原因，由于重启之后，日志被覆盖，所以无法从日志分析事故原因。只能从服务器运行状态，以及数据库的运行状态分析。从这段时间IM服务器和数据库服务器内存和CPU运行分析，除了其中一台IM服务器这段时间CPU有较大波动，其他的服务器CPU和内存使用状态都较为平稳。

由于我们在IM的Java应用启动时添加`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./`的启动参数，该参数的作用是在JVM出现OOM时生成内存快照。我们在CPU出现较大波动的服务的Java应用目录下找到生成JVM内存快照文件，并且内存快照的产生日志就是出现问题当天，所以IM服务连不上很有可能时OOM导致。我们从服务器上将内存快照下载下来，在本地使用MAT（Memory Analyzer Tools）对内存块进行分析。MAT是eclipse开发的一款用于优秀的JVM内存分析工具，官网地址[https://www.eclipse.org/mat/](https://www.eclipse.org/mat/)。

用MAT打开.hprof文件，选择`Leak Suspect Report`分析内存泄漏，生成楼层泄漏分析报告：

![内存泄漏分析报告](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F1.png?raw=true)

从上图看出有一个`org.tio.server.ServerTioConfig`类的实例占用90%的内存，占用3,538,806,520字节约3374M的内存，JVM最大内存设置为4G。再看详细信息：

![内存泄漏分析报告详细信息](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F2.png?raw=true)


从详细信息可以看出，`org.tio.server.ServerTioConfig`的实例中一个类型为`org.tio.core.maintain.Ips`属性占用内存。

![内存泄漏分析报告详细信息](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F4.png?raw=true)

从上图可以看出Ips中有一个HashSet，而HashSet是通过HashMap实现，这个HashMap中有262144个HashMap$Node对象，而每个HashMap$Node的key都是一个`org.tio.server.ServerChannelContext`对象。

`org.tio.server.ServerTioConfig`和`org.tio.core.maintain.Ips`为Tio的类。Tio是一个国产开源的I/O框架，官网地址是：[https://www.tiocloud.com](https://www.tiocloud.com/)。

分析Tio源码，`org.tio.server.ServerTioConfig`继承了`org.tio.core.TioConfig`，`org.tio.core.TioConfig`中有一个`org.tio.core.maintain.Ips`的属性：
```
public abstract class TioConfig extends MapWithLockPropSupport {
    ...
    public Ips									ips							= new Ips();
    ...
}
```
再看`org.tio.core.maintain.Ips`的源码：
```
public class Ips {

	/** The log. */
	private static Logger log = LoggerFactory.getLogger(Ips.class);

	/** 一个IP有哪些客户端
	 * key: ip
	 * value: SetWithLock<ChannelContext>
	 */
	private MapWithLock<String, SetWithLock<ChannelContext>>	ipmap	= new MapWithLock<>(new HashMap<String, SetWithLock<ChannelContext>>());
	private String												rwKey	= "_tio_ips__";

	/**
	 * 和ip绑定
	 * @param ip
	 * @param channelContext
	 * @author tanyaowu
	 */
	public void bind(ChannelContext channelContext) {
		if (channelContext == null) {
			return;
		}

		if (channelContext.tioConfig.isShortConnection) {
			return;
		}

		try {
			String ip = channelContext.getClientNode().getIp();
			if (ChannelContext.UNKNOWN_ADDRESS_IP.equals(ip)) {
				return;
			}

			if (StrUtil.isBlank(ip)) {
				return;
			}

			SetWithLock<ChannelContext> channelSet = ipmap.get(ip);
			if (channelSet == null) {
				LockUtils.runWriteOrWaitRead(rwKey + ip, this, () -> {
//					@Override
//					public void read() {
//					}

//					@Override
//					public void write() {
//						SetWithLock<ChannelContext> channelSet = ipmap.get(ip);
						if (ipmap.get(ip) == null) {
//							channelSet = new SetWithLock<>(new HashSet<ChannelContext>());
							ipmap.put(ip, new SetWithLock<>(new HashSet<ChannelContext>()));
						}
//					}
				});
				channelSet = ipmap.get(ip);
			}
			channelSet.add(channelContext);
		} catch (Exception e) {
			log.error(e.toString(), e);
		}
	}

	/**
	 * 一个ip有哪些客户端，有可能返回null
	 * @param tioConfig
	 * @param ip
	 * @return
	 * @author tanyaowu
	 */
	public SetWithLock<ChannelContext> clients(TioConfig tioConfig, String ip) {
		if (tioConfig.isShortConnection) {
			return null;
		}

		if (StrUtil.isBlank(ip)) {
			return null;
		}
		return ipmap.get(ip);
	}

	/**
	 * @return the ipmap
	 */
	public MapWithLock<String, SetWithLock<ChannelContext>> getIpmap() {
		return ipmap;
	}

	/**
	 * 与指定ip解除绑定
	 * @param ip
	 * @param channelContext
	 * @author tanyaowu
	 */
	public void unbind(ChannelContext channelContext) {
		if (channelContext == null) {
			return;
		}

		if (channelContext.tioConfig.isShortConnection) {
			return;
		}

		try {
			String ip = channelContext.getClientNode().getIp();
			if (StrUtil.isBlank(ip)) {
				return;
			}
			if (ChannelContext.UNKNOWN_ADDRESS_IP.equals(ip)) {
				return;
			}

			SetWithLock<ChannelContext> channelSet = ipmap.get(ip);
			if (channelSet != null) {
				channelSet.remove(channelContext);
				if (channelSet.size() == 0) {
					ipmap.remove(ip);
				}
			} else {
				log.info("{}, ip【{}】 找不到对应的SetWithLock", channelContext.tioConfig.getName(), ip);
			}
		} catch (Exception e) {
			log.error(e.toString(), e);
		}
	}
}

```
`org.tio.core.maintain.Ips`提供了两个方法bind和unbind，bind方法的作用是将channelContext的客户端IP作为key，存储放在这个对应的value中，这个value是`org.tio.utils.lock.SetWithLock<ChannelContext>`类型，channelContext是一个连接上下文对象，代表一个socket连接。再来看看`org.tio.utils.lock.SetWithLock`源码：
```
public class SetWithLock<T> extends ObjWithLock<Set<T>> {
	private static final long	serialVersionUID	= -2305909960649321346L;
	private static final Logger	log					= LoggerFactory.getLogger(SetWithLock.class);

	/**
	 * @param set
	 * @author tanyaowu
	 */
	public SetWithLock(Set<T> set) {
		super(set);
	}

	/**
	 * @param set
	 * @param lock
	 * @author tanyaowu
	 */
	public SetWithLock(Set<T> set, ReentrantReadWriteLock lock) {
		super(set, lock);
	}

	/**
	 *
	 * @param t
	 * @return
	 * @author tanyaowu
	 */
	public boolean add(T t) {
		WriteLock writeLock = this.writeLock();
		writeLock.lock();
		try {
			Set<T> set = this.getObj();
			return set.add(t);
		} catch (Throwable e) {
			log.error(e.getMessage(), e);
		} finally {
			writeLock.unlock();
		}
		return false;
	}

	/**
	 *
	 *
	 * @author tanyaowu
	 */
	public void clear() {
		WriteLock writeLock = this.writeLock();
		writeLock.lock();
		try {
			Set<T> set = this.getObj();
			set.clear();
		} catch (Throwable e) {
			log.error(e.getMessage(), e);
		} finally {
			writeLock.unlock();
		}
	}

	/**
	 *
	 * @param t
	 * @return
	 * @author tanyaowu
	 */
	public boolean remove(T t) {
		WriteLock writeLock = this.writeLock();
		writeLock.lock();
		try {
			Set<T> set = this.getObj();
			return set.remove(t);
		} catch (Throwable e) {
			log.error(e.getMessage(), e);
		} finally {
			writeLock.unlock();
		}
		return false;
	}

	/**
	 * 
	 * @return
	 * @author tanyaowu
	 */
	public int size() {
		ReadLock readLock = this.readLock();
		readLock.lock();
		try {
			Set<T> set = this.getObj();
			return set.size();
		} finally {
			readLock.unlock();
		}
	}
}
```
SetWithLock对HashSet进行了包装，使用ReentrantReadWriteLock对HashSet的读操作加读锁，写操作加写锁，保证对HashSet读写操作的线程安全。内存快照中占满内存的channelContext对象就是存在SetWithLock的HashSet里面。接着我们来看看channelContext是何时添加进去，并且在何时移除的。channelContext是通过org.tio.core.maintain.Ips#bind方法添加进去的，而这个方法只在org.tio.server.AcceptCompletionHandler#completed方法被调用过：
```
public class AcceptCompletionHandler implements CompletionHandler<AsynchronousSocketChannel, TioServer> {

	@Override
	public void completed(AsynchronousSocketChannel asynchronousSocketChannel, TioServer tioServer) {
		...
		ServerChannelContext channelContext = new ServerChannelContext(serverTioConfig, asynchronousSocketChannel);
			channelContext.setClosed(false);
			channelContext.stat.setTimeFirstConnected(SystemTimer.currTime);
			channelContext.setServerNode(tioServer.getServerNode());

			//			channelContext.traceClient(ChannelAction.CONNECT, null, null);

			//			serverTioConfig.connecteds.add(channelContext);
			serverTioConfig.ips.bind(channelContext);
		...
	}
}
```
从名字看，这个类应该是在服务接受客户端的socket请求的时候调用的，实际也正是如此：
```
public class ServerTioConfig extends TioConfig {

	public ServerTioConfig(String name, ServerAioHandler serverAioHandler, ServerAioListener serverAioListener, SynThreadPoolExecutor tioExecutor,
	        ThreadPoolExecutor groupExecutor) {
		super(tioExecutor, groupExecutor);
		this.ipBlacklist = new IpBlacklist(id, this);
		init(name, serverAioHandler, serverAioListener, tioExecutor, groupExecutor);
	}

	private void init(String name, ServerAioHandler serverAioHandler, ServerAioListener serverAioListener, SynThreadPoolExecutor tioExecutor, ThreadPoolExecutor groupExecutor) {
		this.name = name;
		this.groupStat = new ServerGroupStat();
		this.acceptCompletionHandler = new AcceptCompletionHandler();
		...
	}
}
```
`org.tio.server.AcceptCompletionHandler`是在`org.tio.server.ServerTioConfig`实例创建时被初始化的，而`org.tio.server.ServerTioConfig`是Tio的全局配置类，保存一些全局的配置，Ips也是作为全局的配置被其持有引用。`org.tio.server.AcceptCompletionHandler`又是何时被调用的呢？
```
public class TioServer {

	public void start(String serverIp, int serverPort) throws IOException {
		long start = System.currentTimeMillis();
		this.serverNode = new Node(serverIp, serverPort);
		channelGroup = AsynchronousChannelGroup.withThreadPool(serverTioConfig.groupExecutor);
		serverSocketChannel = AsynchronousServerSocketChannel.open(channelGroup);

		serverSocketChannel.setOption(StandardSocketOptions.SO_REUSEADDR, true);
		serverSocketChannel.setOption(StandardSocketOptions.SO_RCVBUF, 64 * 1024);

		InetSocketAddress listenAddress = null;

		if (StrUtil.isBlank(serverIp)) {
			listenAddress = new InetSocketAddress(serverPort);
		} else {
			listenAddress = new InetSocketAddress(serverIp, serverPort);
		}

		serverSocketChannel.bind(listenAddress, 0);

		AcceptCompletionHandler acceptCompletionHandler = serverTioConfig.getAcceptCompletionHandler();
		serverSocketChannel.accept(this, acceptCompletionHandler);
		...
	}
}
```
Tio底层使用的是AIO的API，`org.tio.server.AcceptCompletionHandler`实在服务端接受socket连接之后，会调用org.tio.server.AcceptCompletionHandler#completed方法。

再看org.tio.core.maintain.Ips#unbind方法调用：
```
public class MaintainUtils {
	public static void remove(ChannelContext channelContext) {
		TioConfig tioConfig = channelContext.tioConfig;
		if (!tioConfig.isServer()) {
			ClientTioConfig clientTioConfig = (ClientTioConfig) tioConfig;
			clientTioConfig.closeds.remove(channelContext);
			clientTioConfig.connecteds.remove(channelContext);
		}

		tioConfig.connections.remove(channelContext);
		tioConfig.ips.unbind(channelContext);
		tioConfig.ids.unbind(channelContext);

		close(channelContext);
	}
}
```
```
public class CloseRunnable extends AbstractQueueRunnable<ChannelContext> {

	private static Logger log = LoggerFactory.getLogger(CloseRunnable.class);

	public CloseRunnable(Executor executor) {
		super(executor);
		getMsgQueue();
	}
	//	long count = 1;

	@Override
	public void runTask() {
		if (msgQueue.isEmpty()) {
			return;
		}
		ChannelContext channelContext = null;
		while ((channelContext = msgQueue.poll()) != null) {
			if (isNeedRemove) {
							MaintainUtils.remove(channelContext);
						} else {
							ClientTioConfig clientTioConfig = (ClientTioConfig) channelContext.tioConfig;
							clientTioConfig.closeds.add(channelContext);
							clientTioConfig.connecteds.remove(channelContext);
							MaintainUtils.close(channelContext);
						}
			...
		}
	}
}
```
CloseRunnable的runTask方法中，从一个阻塞队列中取出channelContext判断是否需要移除。如果需要移除，调用org.tio.core.maintain.MaintainUtils#remove方法移除。org.tio.core.maintain.MaintainUtils#remove又调用了
org.tio.core.maintain.Ips#unbind方法。
org.tio.core.task.CloseRunnable的UML类图如下图所示：

![CloseRunnable类图](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/CloseRunnable%E7%B1%BB%E5%9B%BE.png?raw=true)

CloseRunnable继承AbstractQueueRunnable，AbstractQueueRunnable继承AbstractSynRunnable，AbstractSynRunnable实现了Runnable，重写run方法，在run方法中调用runTask方法。AbstractSynRunnable中还有一个execute方法，通过线程池调用当前实例，执行run方法。
```
public abstract class AbstractSynRunnable implements Runnable {

	/**
	 * 把本任务对象提交到线程池去执行
	 * @author tanyaowu
	 */
	public void execute() {
		executor.execute(this);
	}

	@Override
	public final void run() {
		if (isCanceled()) //任务已经被取消
		{
			return;
		}
		boolean tryLock = false;
		try {
			tryLock = runningLock.tryLock(1L, TimeUnit.SECONDS);
		} catch (InterruptedException e1) {
			log.error(e1.toString(), e1);
		}
		if (tryLock) {
			try {
				int loopCount = 0;
				runTask();
				while (isNeededExecute() && loopCount++ < 100) {
					runTask();
				}

			} catch (Throwable e) {
				log.error(e.toString(), e);
			} finally {
				executed = false;
				runningLock.unlock();
			}
		} else {
			executed = false;
		}

		//下面这段代码一定要在unlock()后面，别弄错了 ^_^
		if (isNeededExecute()) {
			execute();
		}

	}

}
```
CloseRunnable是在TioConfig初始化的时候创建的实例：
```
public abstract class TioConfig extends MapWithLockPropSupport {

	public TioConfig(SynThreadPoolExecutor tioExecutor, ThreadPoolExecutor groupExecutor) {
		...
		closeRunnable = new CloseRunnable(this.tioExecutor);
	}
}
```
而ServerTioConfig是TioConfig的子类，ServerTioConfig在Tio启动时创建实例。在org.tio.core.Tio#close(org.tio.core.ChannelContext, java.lang.Throwable, java.lang.String, boolean, boolean, org.tio.core.ChannelContext.CloseCode)方法中，将channelContext添加到CloseRunnable的阻塞启动中，并调用了CloseRunnable的execute()方法。Tio的close方法有多个重载方法，但最终都是调用的org.tio.core.Tio#close(org.tio.core.ChannelContext, java.lang.Throwable, java.lang.String, boolean, boolean, org.tio.core.ChannelContext.CloseCode)方法。

那Tio在什么情况情况下会调用close方法，分析源码，Tio在读取数据、写入数据、解码数据的时候出现异常，会主动调用close方法。并且Tio在发送数据和读取数据之前，都会调用org.tio.core.utils.TioUtils#checkBeforeIO(ChannelContext)方法，该方法的源码如下：
```
public class TioUtils {
	private static Logger log = LoggerFactory.getLogger(TioUtils.class);

	public static boolean checkBeforeIO(ChannelContext channelContext) {
		if (channelContext.isWaitingClose) {
			return false;
		}

		Boolean isopen = null;
		if (channelContext.asynchronousSocketChannel != null) {
			isopen = channelContext.asynchronousSocketChannel.isOpen();

			if (channelContext.isClosed || channelContext.isRemoved) {
				if (isopen) {
					try {
						Tio.close(channelContext,
						        "asynchronousSocketChannel is open, but channelContext isClosed: " + channelContext.isClosed + ", isRemoved: " + channelContext.isRemoved, CloseCode.CHANNEL_NOT_OPEN);
					} catch (Throwable e) {
						log.error(e.toString(), e);
					}
				}
				log.info("{}, isopen:{}, isClosed:{}, isRemoved:{}", channelContext, isopen, channelContext.isClosed, channelContext.isRemoved);
				return false;
			}
		} else {
			log.error("{}, 请检查此异常, asynchronousSocketChannel is null, isClosed:{}, isRemoved:{}, {} ", channelContext, channelContext.isClosed, channelContext.isRemoved,
			        ThreadUtils.stackTrace());
			return false;
		}

		if (!isopen) {
			log.info("{}, 可能对方关闭了连接, isopen:{}, isClosed:{}, isRemoved:{}", channelContext, isopen, channelContext.isClosed, channelContext.isRemoved);
			Tio.close(channelContext, "asynchronousSocketChannel is not open, 可能对方关闭了连接", CloseCode.CHANNEL_NOT_OPEN);
			return false;
		}
		return true;
	}

}
```
在这方法中，检查channelContext是否关闭，或者channelContext的isClosed或者isRemoved变量是否设置为true，如果是会调用Tio.close()方法。

至此，我们梳理完成了Ips的bind()和unbind()方法整个逻辑。Tio在建立新的连接成之后，会调用Ips的bind()方法，将连接对象channelContext添加Ips的容器中。在读写数据出现异常，或者将channelContext的isClosed或者isRemoved这是为true，会调用Ips的unbind()方法，将channelContext移除。

在我们的业务代码中，我们会在心跳检测的时候检查对应连接关联的用户信息是否存在，如果不存在，会将channelContext的isClosed设置为false，来关闭连接。查看ChannelContext的相关实现：
```
public abstract class ChannelContext extends MapWithLockPropSupport {

	public void setClosed(boolean isClosed) {
		this.isClosed = isClosed;
		if (isClosed) {
			if (clientNode == null || !UNKNOWN_ADDRESS_IP.equals(clientNode.getIp())) {
				String before = this.toString();
				assignAnUnknownClientNode();
				log.info("关闭前{}, 关闭后{}", before, this);
			}
		}
	}

	private void assignAnUnknownClientNode() {
		Node clientNode = new Node(UNKNOWN_ADDRESS_IP, UNKNOWN_ADDRESS_PORT_SEQ.incrementAndGet());
		setClientNode(clientNode);
	}

	public void setClientNode(Node clientNode) {
		if (!this.tioConfig.isShortConnection) {
			if (this.clientNode != null) {
				tioConfig.clientNodes.remove(this);
			}
		}

		this.clientNode = clientNode;
		if (this.tioConfig.isShortConnection) {
			return;
		}

		if (this.clientNode != null && !Objects.equals(UNKNOWN_ADDRESS_IP, this.clientNode.getIp())) {
			tioConfig.clientNodes.put(this);
			//			clientNodeTraceFilename = StrUtil.replaceAll(clientNode.toString(), ":", "_");
		}
	}
}
```
在这个方法中，将ChannelContext的isClosed设置为false，同时将ChannelContext的clientNode的IP设置为$UNKNOWN。但是在unbind方法中：
```
String ip = channelContext.getClientNode().getIp();
			if (StrUtil.isBlank(ip)) {
				return;
			}
			if (ChannelContext.UNKNOWN_ADDRESS_IP.equals(ip)) {
				return;
			}
```
会获取channelContext的ip，如果为\$UNKNOWN直接返回。并且通过MAT查看Ips中HashMap\$Node中key，找到channelContext的clientNode，clientNode的ip都是\$UNKNOWN。

![内存泄漏分析](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F5.png?raw=true)

通过分析内存快照中数据，也正式了我们的推测。内存泄漏无法释放的原因就是在心跳检测的时候，检测到连接对应连接的用户信息不存在时，将对应channelContext的isClosed设置为false时，将channelContext的clientNode的ip设置为\$UNKNOWN，导致channelContext无法从Ips中移除，从而导致内存泄漏而产生OOM。最后的解决方法是，服务端主动关闭连接时，使用Tio的close方法，就可以避免问题。