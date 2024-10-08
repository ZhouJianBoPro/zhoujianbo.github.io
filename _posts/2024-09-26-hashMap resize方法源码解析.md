---
layout: post
title: hashMap resize方法解析
date: 2024-09-26
tags: [hash]
---

### resize方法源码
```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        
        // 数组扩容的基本条件
        if (oldCap > 0) {
            // 原数组容量大于hashMap数组最大阈值（1 << 30），不进行扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 扩容阈值设为最大，后续就不会走到扩容方法了
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            
            // 新数组容量 = oldCap << 1 （双倍扩容），同时新扩容阈值 = oldThr << 1
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) 
            // 初始化时直接设置了阈值（threshold），而没有指定初始容量
            newCap = oldThr;
        else {               
            // 当原数组长度和扩容阈值都为0时，初始化数组默认容量为16，默认扩容阈值为16 * 0.75 = 12
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        
        // 定义新数组，数组长度为新扩容容量
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            // 遍历原数组，将原数组中下标不为null的元素转移到新数组中，并且将原数组下标的元素置为null
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    // 原数组下标设为null
                    oldTab[j] = null;
                    // 当原数组下标中只有一个元素时，重新计算该元素在新数组中的下标，并将引用地址传递给新数组下标
                    // 数组双倍扩容之后，元素在新数组中的下标最多只有2个：1. newIndex = oldIndex（低位下标）  2. newIndex = oldIndex + oldCap（高位下标）
                    // 这是因为元素的hash值不变，但是数组长度翻倍了。理解：hash & (cap - 1) 位与运算
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // 当原数组下标的元素为红黑树结构时，遍历该树所有元素，并重新计算每个元素在新数组中的下标
                        // 该树所有元素重新计算出来的下标最多可能有2个，低位下标元素 + 高位下标元素
                        // 当(低位下标元素数量 <= 6) || (高位下标元素数量 <= 6) 时，则将这些元素以链表结构存储在新数组中，反之仍然以红黑树的结构存储
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else {
                        // 原数组下标元素为链表结构，遍历该链表元素元素，并重新计算每个元素在新数组的下标
                        // 低位下标元素 + 高位下标元素分别以链表的形式存储在新数组中
                        
                        // 低位头指针、低位尾指针
                        Node<K,V> loHead = null, loTail = null;
                        // 高位头指针，高位尾指针
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            // 低位头指针指向数组低位下标
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // 高位头指针指向数组高位下标
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

### 数组扩容过程
1. （当原数组长度 > 0） && （原数组长度 < (1 << 30)）时，新数组长度 = 原数组长度 << 1, 新数组扩容阈值 = 原数组扩容阈值 << 1
2. 当原数组长度 == 0 && 原数组扩容阈值 == 0时，进行数组的初始化，默认容量为16，默认扩容阈值为16 * 0.75 = 12
3. 初始化一个新数组，数组长度为新扩容容量。遍历原数组中的下标非null的元素，并将原数组下标置为null
4. 当原数组下标中元素没有下一个元素节点时，重新计算该元素hash值在新数组中的下标，并将该元素的引用地址传递给新数组下标
5. 当下标元素为红黑树结构时，遍历该树的所有元素节点，并重新计算每个元素在新数组中的下标。这些元素的下标最多只有两个，即低位下标（原下标）和高位下标（原下标 + 原数组长度）。当低位下标或高位下标中的元素数量<=6时，会将低位或高位下标元素拆分为链表结构存储在新数组中，反之仍然以红黑树的结构存储
6. 当下标元素为链表结构时，同样遍历链表中所有元素并重新计算下标，将低位和高位下标元素分别以两个链表结构存储在新数组中