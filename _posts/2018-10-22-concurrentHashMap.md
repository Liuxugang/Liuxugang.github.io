---
layout:     post
title:      concurrentHashMap
subtitle:   concurrentHashMap 源码阅读
date:       2018-10-22
author:     刘旭刚
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - 源码阅读
    - java
    - 基础

## concurrentHashMap 源码阅读
#### 1.put方法
```
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)      //如果tab为空，对map进行初始化
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { //计算index位置，如果该位置为null，说明该位置没记录
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))     //则将数据存放在改index上，跳出
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)       //判断是否在扩容中
                tab = helpTransfer(tab, f);   
            else {            //最后将数据插入到map中
                V oldVal = null;
                synchronized (f) {    //对当前节点进行加锁，不影响其他节点
                    if (tabAt(tab, i) == f) {  //判断是链表还是红黑树
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {    //key和hash值相同，对该值进行替换
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {      //遍历完，发现不存在相同的，则对该值进行尾插法
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {   //如果是红黑树结构
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {   //查找或将数据存入该结构中
                                oldVal = p.val;
                                if (!onlyIfAbsent)  //如果找到了，则替换
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);    //如果大于8  则进行链表旋转
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount); //增加数量，如果数目超出则进行扩容
        return null;
    }
```
