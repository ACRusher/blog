---
layout: post
title: zookeeper实现分布式锁 
categories: [blog ]
tags: [软件开发  ]
description: zookeeper实现分布式锁 
---

# zookeeper实现分布式锁 


## 前言

zookeeper可以理解为一个高可靠的分布式文件系统，想了解更多可以参考：https://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html 。

## 分布式锁实现

### 伪代码

    1. 判断锁目录是否存在，不存在则创建锁目录（忽略并发导致的创建失败）
	2. 在锁目录下创建临时序列化节点，节点的data为唯一性token（jvm+threadId）
	3. 判断自己的节点名称是否是最小，如果是则获取到锁跳转到5，否则到4
	4. 监听锁目录子节点的变化，（防止丢失event，sleep前主动测试下是否可到5），如果发生改变，则跳转到3，如果等待过程中超时则抛超时异常
	5. 执行获取到锁后的业务代码，然后删除自己的临时节点（通过token查找）
	6. 如果锁的子节点为空，则尝试删除锁目录（如果由于并发导致子节点不空，忽略掉删除失败）

### java实现

```java

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.List;
import java.util.PriorityQueue;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.LockSupport;
import java.util.concurrent.locks.ReentrantLock;
import java.util.regex.Pattern;

/**
 * zookeeper 实现的分布式锁
 * 本类只是用来示范如何正确实现分布式锁，如果线上要用请继承本类并复写Zookeeper的初始化
 * <p>
 * Created by zhouxiliang on 2016/8/17.
 */
public class ZKLock {
    private static final Logger logger = LoggerFactory.getLogger(ZKLock.class);

    //会话超时时间
    private static final int SESSION_TIMEOUT = 30000;
    //zookeeper session
    protected static ZooKeeper zooKeeper = null;
    //文件根路径
    static String ROOT = null;
    //默认监控器
    private static Watcher defaultWatcher = null;
    //zookeeper集群
    private static String connectionUrl = null;

    static {
        init();
    }

    /**
     * 请不要使用默认的zookeeper，这儿只是作为例子
     * 请自行重写zooKeeper即可。
     */
    private static void init() {
        try {
            connectionUrl = "127.0.0.1:2181";
            ROOT = "/boss-config/lock";
            defaultWatcher = new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    System.out.println(watchedEvent);
                    logger.info("Receive WatchedEvent : " + watchedEvent.toString());
                }
            };
            zooKeeper = new ZooKeeper(connectionUrl, SESSION_TIMEOUT, defaultWatcher);
            zooKeeper.addAuthInfo("digest", "boss-config:boss-config".getBytes());
        } catch (Exception e) {
            logger.error("ZKLock Initializing zookeeper failed. fatal error!", e);
        }
    }

    private static int cnt = 0;

    public static void main(String[] args) throws KeeperException, InterruptedException {
        /*
         *测试的时候 去掉jvm本地锁
         */
        List<String> list = zooKeeper.getChildren("/boss-config/lock", false);
        System.out.println(list.toString());
        final CountDownLatch countDownLatch = new CountDownLatch(10);
        final CountDownLatch start = new CountDownLatch(1);
        for (int i = 0; i < 10; ++i) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        start.await();
                        lock("test_lock");
                        System.out.println("Thread-" + Thread.currentThread().getId() + " 获取到锁");
                        System.out.println(Thread.currentThread().getId() + " is doing some biz...");
                        int t = cnt;
                        Thread.sleep(1000);
                        cnt = t + 1;
                    } catch (Exception e) {
                        e.printStackTrace();
                    } finally {
                        unlock("test_lock");
                        countDownLatch.countDown();
                    }
                }
            }).start();

        }
        long startT = System.currentTimeMillis();
        start.countDown();
        countDownLatch.await();
        long endT = System.currentTimeMillis();
        System.out.println("耗时：（ms） " + (endT - startT));
        list = zooKeeper.getChildren("/boss-config/lock", false);
        System.out.println(list.toString());
        System.out.println("cnt = " + cnt);
    }

    /**
     * 本机锁
     */
    private static ConcurrentHashMap<String, ReentrantLock> localLock = new ConcurrentHashMap<String, ReentrantLock>();

    /**
     * 获取分布式锁
     *
     * @param lockName
     * @throws LockException
     */
    public static void lock(String lockName) throws LockException, KeeperException, InterruptedException {
        //验证参数
        if (!isValidLockName(lockName)) {
            throw new LockException("锁名称非法");
        }
        lockName = ROOT + "/" + lockName;
        //获取jvm锁
        if (!localLock.containsKey(lockName)) {
            localLock.putIfAbsent(lockName, new ReentrantLock());
        }
        Lock jvmLock = localLock.get(lockName);
        jvmLock.lock();
        System.out.println("Thread-" + Thread.currentThread().getId() + " 获取 jvm锁成功");

        try {
            Stat stat = zooKeeper.exists(lockName, false);
            if (stat == null || stat.getCzxid() <= 0) {
                System.out.println("Thread-" + Thread.currentThread().getId() + " 尝试创建锁 ： " + lockName);
                zooKeeper.create(lockName, NetworkUtil.getLocalIP().getBytes("utf-8"), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }
        } catch (Exception e) {
            //ignore concurrent create lock node exception
        }
        String path = lockName + "/node";
        try {
            path = zooKeeper.create(path, (NetworkUtil.getLocalIP() + "_" + Thread.currentThread().getId()).getBytes("utf-8"),
                    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        } catch (Exception e) {
            throw new LockException("Thread-" + Thread.currentThread().getId() + " 创建zk临时节点失败 path=" + path, e);
        }
        System.out.println("Thread-" + Thread.currentThread().getId() + " 临时节点创建成功 ： " + path);
        //快速尝试，是否可以获取锁
        List<String> list = zooKeeper.getChildren(lockName, false);
        if (isSmallestNode(path, list)) {
            //获取锁成功
            return;
        }
        //loop until get the lock
        while (!isSmallestNode(lockName, path)) {
            final Thread currentThread = Thread.currentThread();
            zooKeeper.getChildren(lockName, new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    LockSupport.unpark(currentThread);
                }
            });
            // 在sleep前再次验证，防止漏掉zookeeper事件
            if (!isSmallestNode(lockName, path)) {
                LockSupport.park();
            }
        }
    }

    /**
     * 获取分布式锁
     *
     * @param lockName
     * @param timeout  超时时间（ms)
     * @throws LockException
     */
    public static void lock(String lockName, long timeout) throws LockException, InterruptedException, KeeperException {
        long start = System.currentTimeMillis();
        //验证参数
        if (!isValidLockName(lockName)) {
            throw new LockException("锁名称非法");
        }
        lockName = ROOT + "/" + lockName;
        //获取jvm锁
        if (!localLock.containsKey(lockName)) {
            localLock.putIfAbsent(lockName, new ReentrantLock());
        }
        Lock jvmLock = localLock.get(lockName);
        boolean f = jvmLock.tryLock(timeout, TimeUnit.MILLISECONDS);
        if (!f) {
            throw new LockException("获取本地锁超时, lock=" + lockName);
        }
        System.out.println("Thread-" + Thread.currentThread() + " 获取 jvm锁成功");

        try {
            Stat stat = zooKeeper.exists(lockName, false);
            if (stat == null || stat.getCzxid() <= 0) {
                System.out.println("Thread-" + Thread.currentThread().getId() + " 创建zk锁 : " + lockName);
                zooKeeper.create(lockName, NetworkUtil.getLocalIP().getBytes("utf-8"),
                        ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }
        } catch (Exception e) {
            //ignore
        }
        String path = lockName + "/node";
        try {
            path = zooKeeper.create(path, (NetworkUtil.getLocalIP() + "_" + Thread.currentThread().getId()).getBytes("utf-8"),
                    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        } catch (Exception e) {
            throw new LockException("Thread-" + Thread.currentThread().getId() + " 创建zk临时节点失败", e);
        }
        System.out.println("Thread-" + Thread.currentThread().getId() + " 临时节点创建成功 ： " + path);
        //快速尝试，是否可以获取锁
        List<String> list = zooKeeper.getChildren(lockName, false);
        if (isSmallestNode(path, list)) {
            //获取锁成功
            return;
        }
        timeout -= System.currentTimeMillis() - start;
        if (timeout <= 0) throw new LockException("Thread-" + Thread.currentThread().getId() + " 获取ZK锁超时");
        start = System.currentTimeMillis();
        //loop until get the lock
        while (!isSmallestNode(lockName, path)) {
            timeout -= System.currentTimeMillis() - start;
            if (timeout <= 0) throw new LockException("Thread-" + Thread.currentThread().getId() + " 获取ZK锁超时");
            start = System.currentTimeMillis();
            final Thread currentThread = Thread.currentThread();
            zooKeeper.getChildren(lockName, new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    LockSupport.unpark(currentThread);
                }
            });
            // 在sleep前再次验证，防止漏掉zookeeper事件
            if (!isSmallestNode(lockName, path)) {
                LockSupport.parkNanos(TimeUnit.MILLISECONDS.toNanos(timeout));
            }
        }
    }

    /**
     * 释放分布式锁
     *
     * @param lockName
     * @throws LockException
     */
    public static void unlock(String lockName) {
        //验证参数
        if (!isValidLockName(lockName)) {
            return;
        }
        lockName = ROOT + "/" + lockName;
        try {
            List<String> list = zooKeeper.getChildren(lockName, false);
            if (list != null && !list.isEmpty()) {
                PriorityQueue<String> priorityQueue = new PriorityQueue<String>(list);
                String smallest = priorityQueue.poll();
                byte[] data = zooKeeper.getData(lockName + "/" + smallest, false, null);
                String token = NetworkUtil.getLocalIP() + "_" + Thread.currentThread().getId();
                if (new String(data, "utf-8").equals(token)) {
                    System.out.println("Thread-" + Thread.currentThread().getId() + " unlock 删除zk节点： " + lockName + "/" + smallest);
                    zooKeeper.delete(lockName + "/" + smallest, -1);
                } else {
                    System.out.println("Thread-" + Thread.currentThread().getId() + " unlock 拒绝删除ZK节点 ： " + lockName + " data=" + new String(data, "utf-8"));
                    //node的个数最大为服务器集群个数，所以这儿性能不会损耗太大
                    for (String node : priorityQueue) {
                        String path = lockName + "/" + node;
                        try {
                            data = zooKeeper.getData(path, false, null);
                            if (new String(data, "utf-8").equals(token)) {
                                System.out.println("Thread-" + Thread.currentThread().getId() + " unlock 删除zk节点: " + path);
                                zooKeeper.delete(path, -1);
                                break;
                            }
                        } catch (Exception e) {
                            //ignore no node exception
                        }
                    }
                }

            }
            Stat stat = new Stat();
            zooKeeper.getData(lockName, false, stat);
            if (stat.getNumChildren() == 0) {
                System.out.println("Thread-" + Thread.currentThread().getId() + " unlock 删除ZK锁： " + lockName);
                zooKeeper.delete(lockName, stat.getVersion());
            }
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }

        //释放jvm锁
        ReentrantLock jvmLock = localLock.get(lockName);
        if (jvmLock != null) {
            System.out.println("Thread-" + Thread.currentThread().getId() + " 释放 jvm锁");
            jvmLock.unlock();
            if (!jvmLock.hasQueuedThreads()) {
                System.out.println("Thread-" + Thread.currentThread().getId() + " 删除 jvm锁");
                localLock.remove(lockName);
            }
        }
    }

    private static boolean isSmallestNode(String lockName, String nodeName) throws KeeperException, InterruptedException {
        List<String> list = zooKeeper.getChildren(lockName, false);
        if (isSmallestNode(nodeName, list)) {
            return true;
        }
        return false;
    }

    private static boolean isSmallestNode(String node, List<String> nodes) {
        String[] arr = node.split("/");
        node = arr[arr.length - 1];
        for (String s : nodes) {
            if (s.compareTo(node) < 0) return false;
        }
        return true;
    }

    //znode节点命名有限制，这儿限制的比zookeeper更严一些
    private static Pattern pattern = Pattern.compile("\\w+");

    private static boolean isValidLockName(String lock) {
        return pattern.matcher(lock).matches();
    }


    public static class LockException extends java.lang.Exception {
        public LockException() {
        }

        public LockException(String message) {
            super(message);
        }

        public LockException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}

```
	



