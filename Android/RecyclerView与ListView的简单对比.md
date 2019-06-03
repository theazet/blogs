# RecyclerView与ListView的简单对比

### RecyclerView是什么？

> 官方对它的定义是：一种灵活的视图，用于为大型数据集提供有限的窗口。

在RecyclerView出现之前，我们会使用ListView来实现这个窗口，但RecyclerView并不是用来替代ListView的，因为在处理大量简单数据时，ListView会更有优势。

### 那么RecyclerView对比ListView又有什么优点呢？

1. 标准化了**ViewHolder**，RecyclerView的Adapter处理的是ViewHolder(缓存View以避免重复加载)而不再是直接操作View。
2. 提供了**LayoutManager**，可以控制item的布局样式，如横向、纵向和瀑布流
3. 提供了**ItemAnimator**，可以设置item的增删动画
4. 提供了**ItemDecoration**，可以设置item的间隔样式
5. 提供了**NestedScrollingChild2**接口的实现，使得嵌套滑动变得很容易实现
6. 支持**局部刷新**，提供了notifyItemChanged()等方法
7. 强大的**拖拽功能**：ItemDecoration有一个**ItemTouchHelper**的子类，用于处理拖动排序和滑动删除。

### ListView的优势：

1. 原生的item**单击、长按**事件，RecyclerView需要实现onTouchListener
2. 原生的**addHeaderView** 和**addFooterView**
3. ListView相对**方便快捷**，在不需要支持动、频繁更新、局部刷新的情况ListView更方便

### 缓存机制对比

ListView和RecyclerView的缓存机制相似，都是将离屏的item回收，入屏的item先从缓存中读取，但是ListView只有两级缓存，而RecyclerView有四级缓存。

**ListView**

![](http://www.theaze.cn/wp-content/uploads/2019/05/v2-0ea7851996a39115901c2ae3cd5767dd_hd.png)

**RecyclerView**

![](http://www.theaze.cn/wp-content/uploads/2019/05/v2-746b3372c1f813d990681280fe5e93b3_hd.jpg)

RecyclerView具有的优势是：mCacheViews的使用，可以做到屏幕外的列表项ItemView进入屏幕内时也无须bindView快速重用；mRecyclerPool可以供多个RecyclerView共同使用，在特定场景下，如viewpaper+多个列表页下有优势.客观来说，RecyclerView在特定场景下对ListView的缓存机制做了补强和完善。