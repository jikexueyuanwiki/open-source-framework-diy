# 分布式锁的简单实现

分布式锁在分布式应用当中是要经常用到的，主要是解决分布式资源访问冲突的问题。  一开始考虑采用ReentrantLock来实现，但是实际上去实现的时候，是有问题的，ReentrantLock的lock和unlock要求必须是在同一线程进行，而分布式应用中，lock和unlock是两次不相关的请求，因此肯定不是同一线程，因此导致无法使用ReentrantLock。 

接下来就考虑采用自己做个状态来进行锁状态的记录，结果发现总是死锁，仔细一看代码，能不锁死么。 

```
public synchronized void lock(){  
    while(lock){  
        Thread.sleep(1);  
    }  
    lock=true;  
...  
}  
  
public synchronized void unlock(){  
    lock=false;  
...  
}  
```

第一个请求要求获得锁，好么，给他个锁定状态，然后他拿着锁去干活了。 

这个时候，第二个请求也要求锁，OK,他在lock中等待解锁。 

第一个干完活了，过来还锁了，这个时候悲催了，因为，他进不了unlock方法了。 

可能有人会问，为什么采用while，而不是采用wait...notify?这个问题留一下，看看有人能给出来不？ 

总之，上面的方安案流产了。 

同样，不把synchronized 放在方法上，直接放在方法里放个同步对象可以不？？道理是一样的，也会发生上面一样的死锁。 

到此为止前途一片黑暗。 

@沈学良 同学的http://my.oschina.net/shenxueliang/blog/135865写了一个用zk做的同布锁，感觉还是比较复杂的且存疑。自己做不出来吧，又不死心。 

再来看看Lock的接口，想了一下，不遵守Lock的接口了。编写了下面的接口。 

```
public interface DistributedLock extends RemoteObject {  
  
    long lock() throws RemoteException, TimeoutException;  
  
    long tryLock(long time, TimeUnit unit) throws RemoteException, TimeoutException;  
  
    void unlock(long token) throws RemoteException;  
  
}  
```

呵呵，眼尖的同学可能已经发现不同了。 

lock方法增加了个long返回值，tryLock方法，返回的也不是boolean，也是long，unlock方法多了一个long参数型参数，呵呵，技巧就在这里了。 

```
public class DistributedLockImpl extends UnicastRemoteObject implements DistributedLock {  
    /** 
     * 超时单位 
     */  
    private TimeUnit lockTimeoutUnit = TimeUnit.SECONDS;  
    /** 
     * 锁的令牌 
     */  
    private volatile long token = 0;  
    /** 
     * 同步对象 
     */  
    byte[] lock = new byte[0];  
    /** 
     * 默认永不超时 
     */  
    long lockTimeout = 60 * 60;//默认超时3600秒  
    long beginLockTime;//获取令牌时间，单位毫秒  
  
  
    public DistributedLockImpl() throws RemoteException {  
        super();  
    }  
  
  
    /** 
     * @param lockTimeout 锁超时时间，如果加锁的对象不解锁，超时之后自动解锁 
     * @param lockTimeoutUnit  
     * @throws RemoteException 
     */  
    public DistributedLockImpl(long lockTimeout, TimeUnit lockTimeoutUnit) throws RemoteException {  
        super();  
        this.lockTimeout = lockTimeout;  
        this.lockTimeoutUnit = this.lockTimeoutUnit;  
    }  
    public long lock() throws TimeoutException {  
        return tryLock(0, TimeUnit.MILLISECONDS);  
    }  
    private boolean isLockTimeout() {  
        if (lockTimeout <= 0) {  
            return false;  
        }  
        return (System.currentTimeMillis() - beginLockTime) < lockTimeoutUnit.toMillis(lockTimeout);  
    }  
    private long getToken() {  
        beginLockTime = System.currentTimeMillis();  
        token = System.nanoTime();  
        return token;  
    }  
    public long tryLock(long time, TimeUnit unit) throws TimeoutException {  
        synchronized (lock) {  
            long startTime = System.nanoTime();  
            while (token != 0 && isLockTimeout()) {  
                try {  
                    if (time > 0) {  
                        long endTime = System.nanoTime();  
                        if (endTime - startTime >= unit.toMillis(time)) {  
                            throw new TimeoutException();  
                        }  
                    }  
                    Thread.sleep(1);  
                } catch (InterruptedException e) {  
                    //DO Noting  
                }  
            }  
            return getToken();  
        }  
    }  
    public void unlock(long token) {  
        if (this.token != 0 && token == this.token) {  
            this.token = 0;  
        } else {  
            throw new RuntimeException("令牌" + token + "无效.");  
        }  
    }  
}  
```


下面对代码进行一下讲解。 

上面的代码提供了，永远等待的获取锁的lock方法和如果在指定的时间获取锁失败就获得超时异常的tryLock方法，另外还有一个unlock方法。 

技术的关键点实际上就是在token上，上面的实现，有一个基本的假设，就是两次远程调用之间的时间不可能在1纳秒之内完成。因此，每次锁的操作都会返回一个长整型的令牌，就是当时执行时间的纳秒数。下次解锁必须用获得的令牌进行解锁，才可以成功。如此，解锁就不用添加同步操作了，从而解决掉上面死锁的问题。 

实际上，没有令牌也是可以的，但是那样就会导致a获取了锁，但是b执行unlock也会成功解锁，是不安全的，而加入令牌，就可以保证只有加锁者才可以解锁。 

下面是测试代码： 

```
public class TestDLock {  
    public static void main(String[] args) throws Exception {  
        RmiServer rmiServer = new LocalRmiServer();  
        DistributedLockImpl distributedLock = new DistributedLockImpl();  
        rmiServer.registerRemoteObject("lock1", distributedLock);  
        MultiThreadProcessor processor = new MultiThreadProcessor("aa");  
        for (int i = 0; i < 8; i++) {  
            processor.addProcessor(new RunLock("aa" + i));  
        }  
        long s = System.currentTimeMillis();  
        processor.start();  
        long e = System.currentTimeMillis();  
        System.out.println(e - s);  
        rmiServer.unexportObject(distributedLock);  
    }  
}  
  
class RunLock extends AbstractProcessor {  
    public RunLock(String name) {  
        super(name);  
    }  
  
    @Override  
    protected void action() throws Exception {  
        try {  
            RmiServer client = new RemoteRmiServer();  
            DistributedLock lock = client.getRemoteObject("lock1");  
            for (int i = 0; i < 1000; i++) {  
                long token = lock.lock();  
                lock.unlock(token);  
            }  
            System.out.println("end-" + Thread.currentThread().getId());  
        } catch (RemoteException e) {  
            e.printStackTrace();  
        }  
    }  
}  
```


运行情况： 

```
1    -0    [main] INFO   - 线程组<aa>运行开始,线程数8...
2    -3    [aa-aa0] INFO   - 线程<aa-aa0>运行开始...
3    -3    [aa-aa1] INFO   - 线程<aa-aa1>运行开始...
4    -3    [aa-aa2] INFO   - 线程<aa-aa2>运行开始...
5    -3    [aa-aa3] INFO   - 线程<aa-aa3>运行开始...
6    -3    [aa-aa4] INFO   - 线程<aa-aa4>运行开始...
7    -4    [aa-aa5] INFO   - 线程<aa-aa5>运行开始...
8    -4    [aa-aa6] INFO   - 线程<aa-aa6>运行开始...
9    -8    [aa-aa7] INFO   - 线程<aa-aa7>运行开始...
10   end-19
11   -9050 [aa-aa3] INFO   - 线程<aa-aa3>运行结束
12   end-17
13   -9052 [aa-aa1] INFO   - 线程<aa-aa1>运行结束
14   end-20
15   -9056 [aa-aa4] INFO   - 线程<aa-aa4>运行结束
16   end-16
17   -9058 [aa-aa0] INFO   - 线程<aa-aa0>运行结束
18   end-21
19   -9059 [aa-aa5] INFO   - 线程<aa-aa5>运行结束
20   end-26
21   -9063 [aa-aa7] INFO   - 线程<aa-aa7>运行结束
22   end-18
23   -9064 [aa-aa2] INFO   - 线程<aa-aa2>运行结束
24   end-22
25   -9065 [aa-aa6] INFO   - 线程<aa-aa6>运行结束
26   -9066 [main] INFO   - 线程组<aa>运行结束, 用时:9065ms
27   9069
```

也就是9069ms中执行了8000次锁定及解锁操作。 

## 小结： 
上面的分布式锁实现方案，综合考虑了实现简单，锁安全，锁超时等因素。实际测试，大概900到1000次获取锁和释放锁操作每秒，可以满足大多数应用要求。