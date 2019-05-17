---
title: Java多线程设计模式-Worker Thread
date: 2019-05-17 09:55:29
tags: 
- Java
- 设计模式
- 多线程
type: "设计模式"
---
Worker Thread 模式又称工人线程模式，是线程池使用的关键模式之一。本文参考《图解Java多线程设计模式》一书，
从模式中的角色入手，总结、分析了该模式的特点，结合书后习题，对比了该模式与Thread-per-Message模式在吞吐量上的
差别，解答了**线程池的优势**这一常见面试题。
<!-- more -->
# 理论
## 模式中的角色：

* Client：发出请求的一方，向channel put request
* Channel：请求的中继，提供同步的put/take接口
* Worker：获取并处理请求的一方，从channel take request
* Request：请求本身，提供执行的具体实现

## 实现示例

以下实现来源于书中，未使用concurrent包中的线程池。
* Request角色：保存着自己的序号，重写了`toString`方法，打印出自己对应的序号
```java
import java.util.Random;
/**
 * Request defines the work done by the worker (handleRequest method)
 */
public class Request {
	private final String name;
	private final int number;
	private static final Random random = new Random();

	public Request(String name, int number) {
		this.name = name;
		this.number = number;
	}

	// unblocking the thread to evaluate throughput
	public void handleRequest() {
		System.out.println(Thread.currentThread().getName() + " executes " + this);
		/*try {
			//Thread.sleep(random.nextInt(1000));
		} catch (InterruptedException e) {
			//System.out.println(name + ": Interrupted");
		}*/
	}

	public String toString() {
		return "[ Request from " + name + " No." + number + " ]";
	}
}
```
* Client角色：在作出请求时，将序号传递给`Request`实例，
*注意terminated应该在方法中和catch块中都要设置为true，这样才能保证线程能够被终止*
*catch块可能很长时间不会被执行（why），仅在catch中设置terminated值，线程就不能及时退出run方法*
```java
import java.util.Random;

/**
 * Thread that sends request to Channel
 * acts as producer role
 * invokes putRequest on given channel object at construction
 */
public class ClientThread extends Thread {
	private final Channel channel;
	private static final Random random = new Random();
	private boolean terminated = false;

	public ClientThread(String name, Channel channel) {
		super(name);
		this.channel = channel;
	}

	// put request to channel
	public void run() {
		try {
			// continuously put request into channel
			for (int i = 0; !terminated; i++) {
				Request req = new Request(getName(), i);
				channel.putRequest(req);
				//Thread.sleep(random.nextInt(1000));
			}
		} catch (InterruptedException e) {
			System.out.println(getName() + ": Interrupted");
			terminated = true; // set true after interrupted
		}
	}

	public void terminateClient() {
		this.terminated = true; // interrupt is NOT always caught!
		this.interrupt();
		System.out.println("terminated client");
	}
}
```
* Channel角色 & Worker角色：
```java
public class Channel {
	private static final int MAX_REQUESTS = 100;
	private final Request[] reqQueue;
	private int tail; 	// tail, head and count of reqQueue
	private int head;
	private int count;
	
	/**
	 * threads that take requests from channel
	 * and invoke handleRequest of reqest objects
	 */
	private class WorkerThread extends Thread {
		final Channel channel;
		boolean terminated = false; // exercise 8-6

		public WorkerThread(String name, Channel channel) {
			super(name);
			this.channel = channel;
		}

		public void run() {
			while (!terminated) {
				try {
					Request request = channel.takeRequest();
					request.handleRequest();
				} catch (InterruptedException e) {
					System.out.println(getName() + ": Interrupted");
					terminated = true;
				}
			}
		}

		public void terminateWorker() {
			this.terminated = true; // interrupt is NOT always caught!
			this.interrupt();
		}
	}

	// preserve the worker threadpool
	private final WorkerThread[] threadPool;

	public Channel(int threadNum) {
		this.reqQueue = new Request[MAX_REQUESTS];
		this.tail = 0;
		this.head = 0;
		this.count = 0;

		this.threadPool = new WorkerThread[threadNum];
		for (int i = 0; i < threadNum; i++) {
			threadPool[i] = new WorkerThread("Worker-" + i, this);
		}
	}

	public void startWorkers() {
		for (WorkerThread worker : threadPool) {
			worker.start();
		}
	}

/*
    for exercise8-6
 */
	public void terminateWorkers() {
		for (WorkerThread worker : threadPool) {
			worker.terminateWorker();
		}
		System.out.println("terminated workers");
	}

	public synchronized void putRequest(Request req) throws InterruptedException {
		while (count >= reqQueue.length) {
			wait();
		}
		reqQueue[tail] = req;
		tail = (tail + 1) % reqQueue.length;
		count++;
		notifyAll();
	}

	public synchronized Request takeRequest() throws InterruptedException {
		while (count <= 0) {
			wait();
		}
		Request req = reqQueue[head];
		head = (head + 1) % reqQueue.length;
		count--;
		notifyAll();
		return req;
	}
}
```

## java中对该模式的应用

* ThreadPoolExecutor类
* Executors工具类可创建的线程池
```
Executors.newFixedThreadPool
Executors.newCachedThreadPool
Executors.newScheduledThreadPool
```

* Swing事件分发线程

是单个worker的情况: `event dispatching thread`是唯一的worker，当然在其调用的`actionPerformed`方法可以启动新线程来达到非串行执行

## 模式特点

- 也称为`Background Thread`模式，运用了`Producer-Consumer`模式，`worker`线程作为`consumer`，不断从`channel`获取任务并执行，与`Thread-per-message`不同，`worker`线程是可以在创建后复用的
- 与`Thread-per-message`模式相比可能实际提高了吞吐量（不做任何处理，单纯的无业务逻辑吞吐量），原因是`worker-thread`模式会复用工人线程而无需花费时间启动更多的新线程
- 可以进行`Worker`线程数量的控制，即服务的数量（容量）
- 也可以控制`request`的数量，当挤压的`request`过多时可以通过`channel`的`putrequest`方法进行`client`的减速

## 关键：调用与执行的分离

- 相当于异步执行，可以提高系统响应速度
- 请求调度：即可以控制任务执行的顺序（通过`channel`中的`blockingqueue`控制`worker`角色向外拿任务时的顺序，可以有优先级，例如`PriorityBlockingQueue`以及其子类）
- 可以取消和反复执行
- 分布式：负责调用的计算机和负责执行的计算机可以分离，通过网络传递`request`角色！！

# 实战
## Worker Thread vs. Thread-per-Message (exercise 8-3 throughput)
为测试两个模式的吞吐量，我们需要Main函数和几个Client来发起请求，首先是Worker Thread对应的`main`，注意这里只启动了5个worker
```java
public class WorkerThreadMain {
	// class to test implementation of worker-thread design pattern
	public static void main(String[] args) {
		System.out.println(Thread.currentThread().getName() + ": BEGIN");
		Channel channel = new Channel(5);
		channel.startWorkers();
		ClientThread[] clients = new ClientThread[]{
			new ClientThread("Alice", channel),
			new ClientThread("Bob", channel),
			new ClientThread("Ellie", channel),
		};

		for (ClientThread c : clients) {
			c.start();
		}

		try {
			Thread.sleep(5000);
		} catch (InterruptedException e) {
			System.out.println(Thread.currentThread().getName() + ": INTERRUPTED");
		} finally {
			// stop client threads after 30 secs
			for (ClientThread c : clients) {
				c.terminateClient();
			}
			channel.terminateWorkers();
		}
	}
}
```
该`main`首先创建`channel`和`client`，初始化并启动`worker`后等待30秒，最后关闭所有的`worker, client`线程。类似的，Thread-per-Message模式对应的Main函数如下
首先，由于先前的channel采用worker thread模式，因此不能重复使用，我们编写一个简单的`Channel2`来表示Thread-per-Message模式的实现
```java
/**
 * Channel2 directly handle requests by creating new Thread
 * it act as channel and consumer roles (create new thread to handle it)
 */
public class Channel2 {
	public void sendReq(Request req) {
		new Thread() {
			public void run() {
				req.handleRequest();
			}
		}.start();
	}
}
```
这个channel提供`sendReq`方法给client，对于每一个传递进来的request，启动一个新线程来处理。接下来是main函数，与之前的main不同，它直接退出，并未终止所有线程。
```java
public class ThreadperMessageMain {
	// class to implement thread-per-message pattern
	private static class Client extends Thread {
		Channel2 host;
		
		Client(String name, Channel2 host) {
			super(name);
			this.host = host;
		}

		public void run() {
			for (int i = 0; true; i++) {
				host.sendReq(new Request(getName(), i));
			}
		}
	}

	public static void main(String[] args) {
		System.out.println(Thread.currentThread().getName() + ": BEGIN");
		Channel2 host = new Channel2();
		Client[] clients = new Client[3];
		clients[0] = new Client("Alice", host);
		clients[1] = new Client("Bob", host);
		clients[2] = new Client("Ellie", host);

		for (Client client : clients) {
			client.start();
		}

		try {
			Thread.sleep(30000);
		} catch (InterruptedException e) {
			System.out.println(Thread.currentThread().getName() + ":INTERRUPTED");
		}

		//host.shutdown();
		System.exit(0);
	}
}
```
内部类`Client`替代了先前worker thread模式中的client。

### 测试结果

测试共花费了两个30秒，直接对比处理的request数目（即request序号），可以看出在本机上，同样时长处理的non-blocking请求差距有三倍多。
```
Thread-per-message:
Thread-315383 executes [ Request from Alice No.97196 ]
Thread-315385 executes [ Request from Bob No.100932 ]
Thread-315501 executes [ Request from Ellie No.117369 ]

Worker Thread：(5 workers)
Worker-0 executes [ Request from Alice No.495789 ]
Worker-0 executes [ Request from Ellie No.485608 ]
Worker-0 executes [ Request from Bob No.475884 ]
```

### 推论

从本地测试结果来看，当request的处理不做任何事时，两种设计模式有很明显的差距，导致这种差距的原因可能是创建线程的花销与处理request时长的trade-off。
当处理request时间变长后，这种差距会变化，可以通过修改Request再测试一下：

将`Request`中的`handleRequest`修改成
```java
    // ....
	public void handleRequest() {
		System.out.println(Thread.currentThread().getName() + " executes " + this);
		try {
		    // do silly things
		    int counter = 0;
		    while (counter < 10000) { 
		        counter++;
		    }
		    // then sleep for a while
			Thread.sleep(random.nextInt(1000));
		} catch (InterruptedException e) {
		}
	}
	// ....
```

再次用先前的两个main函数测试一下：

```
Thread-per-Message:
Thread-164056 executes [ Request from Ellie No.54819 ]
Thread-164070 executes [ Request from Alice No.54097 ]
Thread-164093 executes [ Request from Bob No.55172 ]

Worker Thread: (50 workers)
Worker-33 executes [ Request from Ellie No.1104 ]
Worker-5 executes [ Request from Bob No.1039 ]
Worker-27 executes [ Request from Alice No.884 ]
```
差距还挺明显的，当然这里的测试有很多变量，只能做直观的感受。

