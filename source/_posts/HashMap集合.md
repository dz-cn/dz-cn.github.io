---
title: HashMap集合
tags:
  - HashMap
description: |
  文章缩略，支持 `Markdown` **文档格式**
pinned: 0
lang: zh
categories:
  - Java 基础篇
type: post
wordCount: 396
charCount: 5404
imgCount: 0
vidCount: 0
wsCount: 0
cbCount: 2
readTime: 约3分钟
abbrlink: 43ade6e4
date: 2025-10-27 22:26:27
---
<!-- toc -->
### 1、HashMap 的底层数据结构是什么?

​      在JDK 8 之前，主要通过数组 + 链表的方式实现。在JDK 8之后，主要通过数组 + 链表 + 红黑树的方式实现。核心变量为Node 类型的数组 table 变量。由 transient 修饰，并重写了readObject 和 writeObject 方法实现序列化相关操作。

### 2、HashMap 是如何解决哈希冲突的?

​      由于通过键哈希值的运算来确定元素在table中的位置，当出现不同键的哈希值相同时，会产生哈希冲突，主要通过链地址法来解决。也就是在数组对应的位置形成链表，而对于链表结点的插入采用的是尾插法，防止在多线程模式下出现循环链表的问题。

### 3、HashMap 的 put 方法的过程

- **首先**，是计算hash值，即：通过将键的hash值的低16异或上高16位。

- **其次**，计算该元素在哈希桶中的位置，即：将数组的长度 - 1 & 上二次哈希值作为该元素在数组中的下标。

- **然后**：判断下标位置的元素情况，即：

    - 如果当前位置没有元素，则直接创建node对象并存储hash值，防止二次计算，然后将当前元素储存在当前下标的位置；

    - 如果当前位置有元素，就以hash值、key的引用地址，key的equales方法的比较顺序来判断key是否相同；

        - 如果有一个成立，则表示key值相同，即触发替换效果，将原位置的元素的value值替换为当前元素的value 值。

        - 如果判断 key 不相同，（省略：就先判断是否为树结构，如果是）继续按照之前的比较顺序遍历整个链表。

            - 当发现键相同时，也是执行替换操作。

            - 当遍历到最后一个时，回在最后一个结点的尾部创建一个新的Node结点。（省略：并判断当前位置的链表的长度是否大于8，当大于 8 时，则尝试树形化，在树形化的内部，会判断哈希桶的长度是否大于64，如果不是，则直接扩容，否则进行树形化）

- **最后**：都会检查size是否超过扩容阈值，如果超过，则进行扩容。而put方法的返回值是当发成替换时，返回的是旧值，发生插入时，返回的是原位置的值，也就是null。

- 源码展示：

  ```java
  /*添加元素源码*/
  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
          Node<K,V>[] tab; Node<K,V> p; int n, i;
          if ((tab = table) == null || (n = tab.length) == 0) //判断是否需要扩容
              n = (tab = resize()).length; //执行扩容
          if ((p = tab[i = (n - 1) & hash]) == null)  //判断要插入的位置是否存在元素
              tab[i] = newNode(hash, key, value, null); //如果不存在，直在该数组的该位置上添加
          else { //该位置上存在元素，判断该元素的键是否与要插入的元素的键相同
              Node<K,V> e; K k;
              //1、先比较hash值，如果hash不相同，则键一定不相同
              //2、再比较键
              	//1、先比较键的地址
              	//2、最后在比较键的内容
              //最后得出，键是否相同
              if (p.hash == hash 
                  && ((k = p.key) == key || (key != null && key.equals(k))))
                  e = p; //如果键相同
              else if (p instanceof TreeNode) //如果键不相同、判断是否为树形结构
                  e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);//是树结构
              else {  //如果键不相同、不为树形结构
                  for (int binCount = 0; ; ++binCount) {
                      if ((e = p.next) == null) { //判断下一个元素是否为空
                          p.next = newNode(hash, key, value, null); //为空，尾插法
                          if (binCount >= TREEIFY_THRESHOLD - 1) // 判断是否需要树形化
                              treeifyBin(tab, hash); //执行树形化
                          break;
                      }
                      //不为空，判断该位置上的链表的键是否相同，相同直接跳出循环做修改操作
                      if (e.hash == hash &&
                          ((k = e.key) == key || (key != null && key.equals(k))))
                          break; 
                      p = e; //给 p 更新，继续比较
                  }
              }
              if (e != null) { // 键相同、执行修改操作,如果是新增操作，e必为null
                  V oldValue = e.value; //获取到该位置上的值
                  if (!onlyIfAbsent || oldValue == null)
                      e.value = value; //将该位置上的值替换为新的值
                  afterNodeAccess(e);  
                  return oldValue; //返回旧值
              }
          }
          ++modCount;
          if (++size > threshold) //元素个数记录+1，判断是否需要扩容
              resize();  //执行扩容
          afterNodeInsertion(evict);
          return null;
      }
  ```



### 4、HashMap的扩容机制？

- 首先，是当哈希桶没有初始化或者哈希桶的长度为 0 时，进行初始化扩容。

    - 即：设置默认的哈希桶的长度为16，扩容阈值是 12 由 数组长度×负载因子，负载因子默认是0.75。然后创建长度为16的node结点对象并赋值给哈希桶即可。
- 然后，是当哈希桶中的元素个数大于了扩容阈值时会触发扩容机制。
    - 首先，确定新数组的长度以及扩容阈值：新数组的长度为原来的**2倍**，扩容阈值也变为原来的**2倍**
    - 然后，循环遍历原数组中所有非空元素
        - 如果该元素没有形成链表，则直接用元素的hash值 & 上新数组的长度 - 1，作为在新数组中的位置。（这也就是哈希桶的长度为什么是 2 的幂次方，因为计算元素在哈希桶中的位置非常方便）
        - 如果改元素的 next 指针不为空，则进行结点的迁移。
            - 1）对于链表来说
                - 首先定义高位链表和低位链表。
                - 然后，循环遍历链表上的所有结点。对于链表上的结点按照哈希值与原数组的长度做 & 运算的结果是否为 0 ，分为地位链表和高位链表，进而重新分配地位链表和高位链表在数组中的位置来使元素分布均匀。具体的过程以高位链表为例：先判断高位链表的尾结点是否为空，如果为空则为头节点赋初始值，反之则为尾结点的next指针赋值，最后都要将当前遍历的结点赋值给尾结点。最后将低位链表插入到新哈希桶中与原下标相同的位置，而高位链表则插入到新哈希桶中原下标 + 原数组长度的位置。
            - 2）对于红黑树，则是调用树分裂的方法，先将树分为两个链表，对于链表中元素个数小于等于 6 的，直接转换为链表，如果大于6，则进行树形化。

- 源码展示：

  ```java
  /*扩容源码*/
  final Node<K,V>[] resize() {
          Node<K,V>[] oldTab = table; //将数组赋值给临时变量
          int oldCap = (oldTab == null) ? 0 : oldTab.length; //获取数组的长度
          int oldThr = threshold; //获取扩容的临界点
          int newCap, newThr = 0; //定义新的长度和扩容的临界点
          if (oldCap > 0) { //判断数组长度是否大于 0，更新临界点和数组长度
              if (oldCap >= MAXIMUM_CAPACITY) { //判断是否超出最大容量
                  threshold = Integer.MAX_VALUE;
                  return oldTab; //如果超出了，就不扩容，直接返回
              }
              else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                       oldCap >= DEFAULT_INITIAL_CAPACITY) //将新的长度为原长度的 2 倍
                  newThr = oldThr << 1; // 新的临界点为原来临界点的 2 倍
          }
          else if (oldThr > 0) // 初始容量置于阈值，用于应对自定义初始值
              newCap = oldThr;
          else {               // 零初始阈值表示使用默认值，用于处理默认情况
              newCap = DEFAULT_INITIAL_CAPACITY; //默认长度为 16
              newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
              //临界值为默认长度*加载因子
          }
          if (newThr == 0) { //用于应对自定义负载因子
              float ft = (float)newCap * loadFactor;
              newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                        (int)ft : Integer.MAX_VALUE);
          }
          threshold = newThr; //更新临界值
          @SuppressWarnings({"rawtypes","unchecked"})
          Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; //创建新的数组
          table = newTab; //将原数组指向该新的长度的数组
          if (oldTab != null) { //用于应对第一次添加的情况
              //源码赋值hashMap的逻辑
              for (int j = 0; j < oldCap; ++j) {
                  Node<K,V> e;  //创建新的节点
                  if ((e = oldTab[j]) != null) { //判断旧数组上该位置的节点是否为空
                      oldTab[j] = null;
                      if (e.next == null) //如果下一个节点为空，即：该位置上就一个元素
                          newTab[e.hash & (newCap - 1)] = e; //计算在新数组中的位置并放入
                      else if (e instanceof TreeNode) //如果下一个有元素，判断是否为树形
                          ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                      else { // 是链表
                          Node<K,V> loHead = null, loTail = null;
                          Node<K,V> hiHead = null, hiTail = null;
                          Node<K,V> next;
                          do {
                              next = e.next;
                              if ((e.hash & oldCap) == 0) { 
                                  //判断是否需要移动，因为在计算位置时，e.hash & (oldCap-1)
                                  //在增加为容量后，新的容量是2的倍数，因此，之后的位置是原来的二进制
                                  //加 1，原 1111 先 10000 因此，只需要比较最高位的 1 是否影向
                                  //而oldCap是2的倍数，因此只有一个1其余是0,16 = 0000 1000
                                  //因此，如果取与不为0，说名根据hsah值计算的位置与原来的位置不一样
                                  //需要移动，同样，只要是需要移动的，要移动的位置都一样，因为都是同一
                                  //个高位二进制为 1
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
                          if (loTail != null) {  //将不需要移动的节点直接放入该数组下
                              loTail.next = null;
                              newTab[j] = loHead;
                          }
                          if (hiTail != null) { //将需要移动的链表放入新的数组的空间下
                              hiTail.next = null;
                              //由于需要移动，因此高位二进制一定是 1，因此用hash值于新数组计算后
                              //位置是确定的，增加了oldCap的长度，故，这里不用再次计算为位置，直接用
                              newTab[j + oldCap] = hiHead;
                          }
                      }
                  }
              }
          }
          return newTab;
      }
  ```



### 5、为什么负载因子是0.75？

​     因为是在时间和空间之间，经过大量计算和实验权衡后的一个最优值。在减少哈希冲突的同时，避免频繁的扩容。

### 6、HashMap 是线程安全的吗？

​      不是，可以用并发工具包的SynchroizedMap包装hashMap，或者直接使用CurrentHashMap。

### 7、HashMapJDK8 和 JDK 7的区别

​      数据结构：从数组 + 链表 ==》数组 + 链表 + 红黑树

​      插入方式：头插法 ==》尾插法

​      迁移原理：重新计算hash ==》按照高低位拆分

### 8、HashMap的内存泄漏？

​      若key对象重写了equels()方法，但是没有重写hashCode()方法，可能无法正确定位元素，造成内存占用不能释放。例如：对于放入两个键相同的对象，此时应表现为修改，但是由于没有重写hashCode()方法，导致hash值不同，而判断是先判断hash值，再判断equals()，因此此时表现为插入元素，而当利用键来删除其中的一个元素时，另一个元素则会被留下（因为哈希值不同），造成内存泄漏。

### 9、key 的值可以为null吗？

​      可以，null 键的hash固定为 0，会被放入列表的第一个位置。