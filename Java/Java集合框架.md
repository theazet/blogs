# Java集合框架

![](http://www.theaze.cn/wp-content/uploads/2019/04/collection.png)

![](http://www.theaze.cn/wp-content/uploads/2019/04/map.png)

以上图示是Java Collection Framework的部分成员类图。

### Iterable接口

Iterable接口是一个特殊的接口，它不在Java集合框架的范围内，它不像集合框架在java.util包内，而是在java.lang包内，但它和Java集合框架却有很大关联。

它有一个iterator()方法用于返回一个迭代器，实现它的类可以使用foreach语句,因为Map接口没有继承Collection接口，所以Map不能使用foreach语句，但是Set和List却可以。

### Collection接口

Collection代表的一组对象的集合，这些对象也成为collection的元素。JDK不提供任何Collection的直接实现，而是提供更具体的子接口的具体实现。它是Set、List、Queue的父接口。

### Set接口

Set模仿了数学上对集合的定义：无序性、互异性、确定性

**无序性**：一个集合中，每个元素的地位都是相同的，元素之间是无序的。

**互异性**：一个集合中，任何两个元素都认为是不相同的，即每个元素只能出现一次。

**确定性**：给定一个集合，任给一个元素，该元素或者属于或者不属于该集合，二者必居其一，不允许有模棱两可的情况出现。

在Java Collection Framework中，Set的有些实现是有序的，具有有序性(如LinkedHashSet)，但是所有实现有具有互异性和确定性。

### SortSet与NavigableSet接口

SortSet是提供关于元素总体排序的Set，它们基于自然顺序进行排序，或者根据创建有序Set时提供的Comparator进行排序，插入的元素必须实现Comparable接口或者被比较器接受。SortSet提供了许多关于元素查找有关的方法。

NavigableSet则是对SortSet的扩展，提供了有关分割Set与返回逆序迭代器的方法。

### AbstractSet与AbstractCollection抽象类

AbstractCollectiont提供了对于集合通用的基本方法的具体实现，如clear()、toArray()、contains()等；AbstractSet提供了对于equals和hashCode具体实现。提供这些抽象类的原因都是为了以最大限度地减少了实现相关接口所需的工作。 

### List相关内容

相关内容请看我的文章[线性表](https://www.theaze.cn/2019/03/29/%E7%BA%BF%E6%80%A7%E8%A1%A8/)。

### Map相关内容

相关内容请看我的文章[散列与JAVA中散列的应用]([https://www.theaze.cn/2019/04/16/%E6%95%A3%E5%88%97%E4%B8%8Ejava%E4%B8%AD%E6%95%A3%E5%88%97%E7%9A%84%E5%BA%94%E7%94%A8/](https://www.theaze.cn/2019/04/16/散列与java中散列的应用/))

### TreeMap

TreeMap是基于红黑树的NavigableMap实现，它需要所有元素的**键**都必须实现Comparator接口，或者可以根据自然顺序排序。

**什么是红黑树？**

红黑树是一种平衡二叉树，红黑树的结构复杂，但它的操作有着良好的最坏情况运行时间，并且在实践中高效：它可以在$O(log_n)$时间内完成查找，插入和删除。

### TreeSet

TreeSet与TreeMap的关系就像LinkedHashMap与LinkedHashSet的关系。TreeSet底层是基于TreeMap的，用一个final的Object作为Map的值，将需要存储的值保存为Map的键。

```java
private static final Object PRESENT = new Object();
```

### Comparator接口

用来控制某些数据结构的顺序，或者为那些没有自然顺序的对象Collection提供排序。

### TreeMap、LinkedHashMap的区别

TreeMap是按自然顺序或者实现Comparator接口的对象的排序，而LinkedHashMap是按accessOrder来判断的，如果为true是按访问顺序，如果是false是按插入顺序排序。

### Iterator接口

Iterator接口中只有三个方法：next(),hasNext(),remove()，接口实现类可以根据自己的数据结构特点，实现这些方法。

在迭代器创建之后，如果从结构上对集合进行修改，除非通过迭代器自身的 `remove` 或 `add` 
方法，其他任何时间任何方式的修改都会使迭代器抛出`ConcurrentModificationException`异常

### WeakHashMap

以*弱键* 实现的基于哈希表的 `Map`。在 `WeakHashMap` 中，当某个键不再正常使用时，将自动移除其条目。更精确地说，对于一个给定的键，其映射的存在并不阻止垃圾回收器对该键的丢弃，这就使该键成为可终止的，被终止，然后被回收。丢弃某个键时，其条目从映射中有效地移除，因此，该类的行为与其他的 `Map` 实现有所不同。 