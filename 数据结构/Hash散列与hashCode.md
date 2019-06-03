# 散列与Java中散列的应用

### 散列技术

**散列**的理想情况是不用经过任何比较，通过在存储位置(H)和关键码(key)之间确立一个关系，使得关键码和唯一的存储位置相对应。

**散列的具体过程为：**

1. 存储记录时，通过散列函数计算散列地址，并按此散列地址储存数据。
2. 查找记录时，通过同样的散列函数计算出散列地址，并按此散列地址访问该数据。

**散列冲突：**

散列冲突是指两个不同的关键码经过同一个散列函数计算后得出相同的散列地址，这种情况也叫“散列碰撞”。

**散列函数需要关注的主要问题：**

1. 散列函数的设计：需要简单、均匀、储存利用率高。
2. 冲突处理：使用合理的方法解决散列冲突。

#### 常用的散列函数

**1.直接定址法**

散列函数是关键码的线性函数：$H=a×key+b(a,b为常数)$

优点：单调、均匀。

适用于预先知道关键码的分布以及关键码集合不是很大的情况。

**2.除数留余法**

散列函数：$H=key\mod \ p $

优点：简单、不需要预先知道关键码的分布

**3.数字分析法**

根据关键码在各个位上的分布情况，选取分布比较均匀的若干位组成散列地址。

**4.平方取中法**

对关键码平方后去截取中间几位作为散列地址，适用于关键码位数不是很大的情况。

**5.折叠法**

将关键码分割成位数相等的几部分，然后叠加，适用于关键码位数很多的情况。

#### 处理冲突的方法

**1.开放定址法**

当散列表产生冲突时，就去寻找下一个空的散列地址，只要散列表足够大，就总能找到需要的地址，这种冲突处理方式得到的散列地址成为**闭散列表**，下面介绍三种解决方式。

- 线性探测法：发生冲突时，从发生冲突的下一个位置起依次寻找空的散列地址。但是这种方法容易产生**堆积**，即本来不是同义词的两个关键码会争夺同一个散列地址，这样会大大降低了查询的效率。
- 二次探测法：$H_i=(H+d_i)\,mod\,m（d_i=1^2,-1^2,2^2,-2^2.....,-q^2且q\leq\sqrt m）$
- 随机探测法：$H_i=(H+d_i)\,mod\,m（d_i是一个随机数列，i=1,2,3,4,5...m-1）$

**2.拉链法(链地址法)**

基本思想是将所有散列地址相同的记录储存在一个单链表里，在需要访问时依次判断单链表记录。

#### 影响散列性能的因素

1. **散列函数是否均匀：**是否均匀直接影响产生冲突的概率，散列函数应当是均匀的。
2. **处理冲突的方法：**根据集合的不同特点选择合适的处理冲突的方法。
3. **散列表的装填因子：**装填因子=填入表中的记录个数/散列表的长度,当装填因子越大，产生冲突的概率就会越大，装填因子越小，占用的内存空间就会越大，所以应该选用一个合适的装填因子。

### Java中的散列

#### hashCode方法

hashCode是Object类的方法，也就是说每个对象都有其hashCode，那么hashCode是什么？

Java对hashCode的常规规定为：

1. 对于一个对象，在equals的值没有改变时，多次调用hashCode()方法必须返回相同的整数，但对于同一段代码，程序的某一次执行与另一次执行返回的值可以不同。
2. 如果根据equals()判断两个对象是相等的，那么他们返回的hashCode也是相等的。
3. 对于两个不同的对象，当equals()方法返回的值不同，也就是两个对象不相等时，允许返回相同的hashCode，但为了提高散列表的性能，应让不相等的对象返回不同的hashCode

这里同时介绍一下equals方法：

equals()用来指示两个对象是否“相等”。

equal方法在非空对象引用中实现相等关系：

- 自反性：a非空，a.equals(a)返回true。
- 对称性：a，b非空，当且仅当a.equals(b)返回true，b.equals(a)返回true。
- 传递性：a，b，c非空，a.equals(b)为true，b.equals(c)为true，那么a.equals(c)为true。
- 一致性：当对象没有变化时，equals()返回值不变。
- a为空，a.equals(null)返回false。

#### Java集合中的散列

**HashMap**

HashMap在冲突处理用的是拉链法，所以在内存中，一般情况下是数组+链表的存储方式，可以把每个链表看做是一个“桶”，当这个桶存储的数据到达TREEIFY_THRESHOLD个，也就是8个的时候，为了查找的效率，Java会把链表转换为平衡二叉树——红黑树。

**两个重要的参数**

initialCapacity(**初始容量**，默认为16)与loadFactor(**加载因子**，默认为0.75)，初始容量的意义很简单，指的就是哈希表中桶的数量，加载因子就是前边说的装填因子，它是一个在0到1之间的float类型数据。加载因子默认为0.75,这是一个折中的参数，当参数设的过大时，会经常出现散列冲突，增加查询成本(如get和put操作)，而如果过小，所需要的资源就越大，而且可能会经常出现rehash(即重建内部数据结构,重建后的桶的数量为原来的两倍)。

**put方法原理**

1. 对key做hash操作，计算出桶的位置(length-1)&hash，如果没有冲突，直接放入桶中
2. 如果发生冲突，则以链表形式存储
3. 如果发生冲突且桶中数据大于8个，则将链表转换为红黑树
4. 如果已经存在key就替换原有数据
5. 如果超过能够存储的容量initialCapacity*loadFactor就进行rehash

**HashMap与HashTable的区别**

1. HashMap实现原理和HashTable实现原理都差不多，HashMap的键、值都是允许为null值的，但是HashTable不可以
2. HashTable是线程同步的，但是HashMap不是，所以我们在没有单线程下要使用HashMap，因为HashTable中的方法都被synchronized所修饰，线程的同步需要耗费资源，但是在单线程下这是没有必要的；而在多线程的情况下，应该使用HashTable。

#### HashSet

已经了解了HashMap了，那么了解HashSet就很容易了，因为在HashSet内部，其实就是利用了HashMap来存储数据。

```java
public HashSet(int initialCapacity, float loadFactor) {
	map = new HashMap<>(initialCapacity, loadFactor);
}
```

那么为什么HashSet能够做到元素的唯一性呢？HashSet内部其实是利用一个final的Object来当做HashMap的value，在HashSet的add方法中当加入重复的key时就会返回false。

```java
private static final Object PRESENT = new Object();
public boolean add(E e) {
	return map.put(e, PRESENT)==null;
}
```

HashMap的key当做HashSet的元素值，这也是为什么HashMap可以通过keySet方法返回一个Set。

#### LinkedHashMap

LinkedHashMap直接继承自HashMap，它是一个**有序**的HashMap，内部结构是在HashMap原来的数组+链表的基础上，加上了双向链表。

结构示意图：

![](http://www.theaze.cn/wp-content/uploads/2019/04/20170512160734275.png)

![](http://www.theaze.cn/wp-content/uploads/2019/04/20170512155609530.png)

*图片来自[<https://blog.csdn.net/justloveyou_/article/details/71713781>](<https://blog.csdn.net/justloveyou_/article/details/71713781>)*

**利用LinkedHashMap实现LRU(Least recently used)**

boolean类型的accessOrder定义了输出的顺序，true为按照访问顺序，false为按照插入顺序。accessOrder设置为true正好满足LRU算法的核心思想。

```java
public LinkedHashMap(int initialCapacity,
                        float loadFactor,
                        boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;  //想要实现LRU时要设置为true
}
```
LinkedHashMap中有个**removeEldestEntry(Map.Entry<K,V> eldest)**方法，它可以删除最长时间没有被访问的变量，可以方便我们实现LRU，Android的LRUCache内部就是基于LinkedHashMap。


