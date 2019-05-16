**一、算法思想**
```
// 方案A:基于临时顺序节点
1、在某父节点下创建临时有序节点
2、判断创建的节点是否是当前父节点下所有子节点中序号最小的
3、是序号最小的成功获取锁，否则监听比自己小的那个节点，进行watch，当该节点被删除的时候通知当前节点，重新获取锁
4、解锁的时候删除当前节点

// 方案B:基于名称唯一的临时节点
// 由于这个方案会出现惊群效应，所以很少会使用这种方式来实现分布式锁。
// 这种思路往往被用来做选举策略，例如kafka的controller选举。
```
**二、算法实现**

**1、基于Apache Curator实现的可重入锁**
```
package com.cagam.lock;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.zookeeper.KeeperException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

// Reentrant In JVM
public class ZkCuratorReentrantLock implements DistributedLock {
    private static final Logger logger = LoggerFactory.getLogger(ZkCuratorReentrantLock.class);
    public static final String ROOT_PATH = "/ROOT_LOCK/"; // Zk锁根目录
    private static final long delayTimeForClean = 1000; // 延迟清理时间
    private static final ScheduledExecutorService executorService = Executors.newScheduledThreadPool(5);

    private CuratorFramework client;
    private String lockPath;
    private InterProcessMutex interProcessMutex;

    public ZkCuratorReentrantLock(CuratorFramework client, String lockPath) {
        if (client == null || lockPath == null
                || "".equals(lockPath.trim()))
            throw new IllegalArgumentException("client and lockPath are not null...");

        this.client = client;
        this.lockPath = ROOT_PATH + lockPath;
        this.interProcessMutex = new InterProcessMutex(client, this.lockPath);
    }

    public boolean tryLock(long timeOut, TimeUnit timeUnit) throws Exception {
        return interProcessMutex.acquire(timeOut, timeUnit);
    }

    public void lock() throws Exception {
        interProcessMutex.acquire();
    }

    public void unlock() {
        try {
            interProcessMutex.release();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            executorService.schedule(new Cleaner(client, lockPath), delayTimeForClean, TimeUnit.MILLISECONDS);
        }
    }

    private static class Cleaner implements Runnable {
        CuratorFramework client;
        String path;

        public Cleaner(CuratorFramework client, String path) {
            this.client = client;
            this.path = path;
        }

        public void run() {
            try {
                List list = client.getChildren().forPath(path);
                if (list == null || list.isEmpty()) {
                    client.delete().forPath(path);
                }
            } catch (KeeperException.NoNodeException e) {
                // do not anything
            } catch (KeeperException.NotEmptyException e1) {
                // do not anything
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
        }
    }
}

```
**2、基于Apache Curator实现的读写锁**
```
package com.cagam.lock;

import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.locks.InterProcessReadWriteLock;
import com.cagam.utils.SysConstants;

// 生产环境下使用，需要考虑下读与写锁释放的问题
public class ZkCuratorReentrantReadWriteLock {  
  private String lockPath;
  private CuratorFramework client;
  private InterProcessReadWriteLock interProcessReadWriteLock;
  private InterProcessMutex readLock;
  private InterProcessMutex writeLock;

  public ZkCuratorReentrantReadWriteLock(CuratorFramework client, String lockPath) {
    this.client = client;
    this.lockPath = SysConstants.ZK_LOCK_ROOT_PATH + lockPath;
    this.interProcessReadWriteLock = new InterProcessReadWriteLock(this.client, this.lockPath);
    this.readLock = interProcessReadWriteLock.readLock();
    this.writeLock = interProcessReadWriteLock.writeLock();
  }

  public InterProcessMutex readLock() {
    return this.readLock;
  }

  public InterProcessMutex writeLock() {
    return this.writeLock;
  }
}
```
**三、附录**
* [可重入锁](https://github.com/c-agam/distributed-toolkit/blob/master/src/main/java/com/cagam/lock/ZkCuratorReentrantLock.java)
* [读写锁](https://github.com/c-agam/distributed-toolkit/blob/master/src/main/java/com/cagam/lock/ZkCuratorReentrantReadWriteLock.java)
