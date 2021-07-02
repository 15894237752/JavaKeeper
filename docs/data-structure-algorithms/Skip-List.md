# 跳表──没听过但很犀利的数据结构

![](https://tva1.sinaimg.cn/large/008i3skNly1grm8yuyywxj30zk0qomyg.jpg)

> Redis 是怎么想的：用跳表来实现有序集合？
>
> Redis 的 zset 是一个复合结构，一方面它需要一个 hash 结构来存储 value 和 score 的对应关系，另一方面需要提供按照 score 来排序的功能，还需要能够指定 score 的范围来获取 value 列表的功能

干过服务端开发的应该都知道 Redis 的 ZSet 使用跳表实现的（当然还有压缩列表），我就不从 1990 年的那个美国大佬 William Pugh 发表的那篇论文开始了，直接开跳

![马里奥](https://i03piccdn.sogoucdn.com/bbdcce2d04b2bd83)

文章拢共两部分

- 跳表是怎么搞的
- Redis 是怎么想的



## 一、跳表

### 跳表的简历

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/datastrucutre/skiplist-resume.png)

跳表，英文名：Skip List

父亲：从英文名可以看出来，它首先是个 List，实际上，它是在有序链表的基础上发展起来的

竞争对手：跳表（skip list）对标的是平衡树（AVL Tree）

优点：是一种 插入/删除/搜索 都是 $O(logn)$ 的数据结构。它最大的优势是原理简单、容易实现、方便扩展、效率更高



### 跳表的基本思想

一如往常，采用循序渐进的手法带你窥探 William Pugh 的小心思~

前提：跳表处理的是有序的链表，所以我们先看个不能再普通了的有序列表（一般是双向链表）

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/datastrucutre/linkedlist.png)

如果我们想查找某个数，只能遍历链表逐个比对，时间复杂度 $O(n)$，插入和删除操作都一样。

为了提高查找效率，我们对链表做个”索引“

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/datastrucutre/skip-index.png)

像这样，我们每隔一个节点取一个数据作为索引节点（或者增加一个指针），比如我们要找 31 直接在索引链表就找到了（遍历 3 次），如果找 16 的话，在遍历到 31的时候，发现大于目标节点，就跳到下一层，接着遍历~ (蓝线表示搜索路径)

> 恩，如果你数了下遍历次数，没错，加不加索引都是 4 次遍历才能找到 16，这是因为数据量太少
>
> 数据量多的话，我们也可以多建几层索引，如下 4 层索引，效果就比较明显了

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/datastrucutre/skiplist.png)

每加一层索引，我们搜索的时间复杂度就降为原来的 $O(n/2)$

加了几层索引，查找一个节点需要遍历的节点个数明线减少了，效率提高不少，bingo~

有没有似曾相识的感觉，像不像二分查找或者二叉搜索树，通过索引来跳过大量的节点，从而提高搜索效率。

这样的多层链表结构，就是『**跳表**』了~~



**那到底提高了多少呢？**

推理一番：

1. 如果一个链表有 n 个结点，如果每两个结点抽取出一个结点建立索引的话，那么第一级索引的结点数大约就是 n/2，第二级索引的结点数大约为 n/4，以此类推第 m 级索引的节点数大约为 $n/(2^m)$。

2. 假如一共有 m 级索引，第 m 级的结点数为两个，通过上边我们找到的规律，那么得出 $n/(2^m)=2$，从而求得 m=$log(n)$-1。如果加上原始链表，那么整个跳表的高度就是 $log(n)$。

3. 我们在查询跳表的时候，如果每一层都需要遍历 k 个结点，那么最终的时间复杂度就为 $O(k*log(n))$。

4. 那这个 k 值为多少呢，按照我们每两个结点提取一个基点建立索引的情况，我们每一级最多需要遍历两个节点，所以 k=2。

   > 为什么每一层最多遍历两个结点呢？
   >
   > 因为我们是每两个节点提取一个节点建立索引，最高一级索引只有两个节点，然后下一层索引比上一层索引两个结点之间增加了一个结点，也就是上一层索引两结点的中值，看到这里是不是想起了二分查找，每次我们只需要判断要找的值在不在当前节点和下一个节点之间就可以了。
   >
   > 不信，你照着下图比划比划，看看同一层能画出 3 条线不~~
   >
   > ![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/datastrucutre/skiplist-index-count.png)

5. 既然知道了每一层最多遍历两个节点，那跳表查询数据的时间复杂度就是 $O(2*log(n))$，常数 2 忽略不计，就是 $O(logn)$ 了。



**空间换时间**

跳表的效率比链表高了，但是跳表需要额外存储多级索引，所以需要更多的内存空间。

跳表的空间复杂度分析并不难，如果一个链表有 n 个节点，每两个节点抽取出一个节点建立索引的话，那么第一级索引的节点数大约就是 n/2，第二级索引的节点数大约为 n/4，以此类推第 m 级索引的节点数大约为 $n/(2^m)$，我们可以看出来这是一个等比数列。

这几级索引的结点总和就是 n/2+n/4+n/8…+8+4+2=n-2，所以跳表的空间复杂度为  $O(n)$。

> 实际上，在软件开发中，我们不必太在意索引占用的额外空间。在讲数据结构和算法时，我们习惯性地把要处理的数据看成整数，但是在实际的软件开发中，原始链表中存储的有可能是很大的对象，而索引结点只需要存储关键值和几个指针，并不需要存储对象，所以当对象比索引结点大很多时，那索引占用的额外空间就可以忽略了。



#### 插入数据

其实插入数据和查找一样，先找到元素要插入的位置，时间复杂度也是 $O(logn)$，但有个问题就是如果一直往原始列表里加数据，不更新我们的索引层，极端情况下就会出现两个索引节点中间数据非常多，相当于退化成了单链表，查找效率直接变成 $O(n)$

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/datastrucutre/skiplist-insert.png)



#### 跳表索引动态更新

我们上边建立索引层都是下层节点个数的 1/2，最高层索引的节点数就是 2 个，但是我们随意插入或者删除一个原有链表的节点，这个比例就肯定会被破坏。

作为一种动态数据结构，我们需要某种手段来维护索引与原始链表大小之间的平衡，也就是说，如果链表中结点多了，索引结点就相应地增加一些，避免复杂度退化。

如果重建索引的话，效率就不能保证了。

> 如果你了解红黑树、AVL 树这样平衡二叉树，你就知道它们是通过左右旋的方式保持左右子树的大小平衡，而跳表是通过随机函数来维护前面提到的“平衡性”。

所以跳表（skip list）索性就不强制要求 `1:2` 了，一个节点要不要被索引，建几层的索引，就随意点吧，都在节点插入时由抛硬币决定。

比如我们要插入新节点 X，那要不要为 X 向上建索引呢，就是抛硬币决定的，正面的话建索引，否则就不建了，就是这么随意（哪几层建索引同样这么随机）。

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/datastrucutre/20210626125654.gif)

其实是因为我们不能预测跳表的添加和删除操作，很难用一种有效的算法保证索引部分始终均匀。学过概率论的我们都知道抛硬币虽然不能让索引位置绝对均匀，当数量足够多的时候最起码可以保证大体上相对均匀。

删除节点相对来说就容易很多了，在索引层找到节点的话，就顺藤摸瓜逐个往下删除该索引节点和原链表上的节点，如果哪一层索引节点被删的就剩 1 个节点的话，直接把这一层搞掉就可以了。



其实跳表的思想很容易理解，可是架不住实战

### 跳表的实现

> https://leetcode-cn.com/problems/design-skiplist/

差不多了解了跳表，其实就是加了几层索引的链表，一共有 N 层，以 0 ~ N-1 层表示，设第 0 层是原链表，抽取其中部分元素，在第 1 层形成新的链表，上下层的相同元素之间连起来；再抽取第 1 层部分元素，构成第 2 层，以此类推。



```java
package skiplist;

import java.util.Random;

/**
 * 跳表的一种实现方法。
 * 跳表中存储的是正整数，并且存储的是不重复的。
 */
public class SkipList {

  private static final int MAX_LEVEL = 16;

  private static final float SKIPLIST_P = 0.5f;

  private int levelCount = 1;

  private Node head = new Node();  // 带头链表

  private Random r = new Random();

  public Node find(int value) {
    Node p = head;
    for (int i = levelCount - 1; i >= 0; --i) {
      while (p.forwards[i] != null && p.forwards[i].data < value) {
        p = p.forwards[i];
      }
    }

    if (p.forwards[0] != null && p.forwards[0].data == value) {
      return p.forwards[0];
    } else {
      return null;
    }
  }

  public void insert(int value) {
    int level = randomLevel();
    Node newNode = new Node();
    newNode.data = value;
    newNode.maxLevel = level;
    Node update[] = new Node[level];
    for (int i = 0; i < level; ++i) {
      update[i] = head;
    }

    // record every level largest value which smaller than insert value in update[]
    Node p = head;
    for (int i = level - 1; i >= 0; --i) {
      while (p.forwards[i] != null && p.forwards[i].data < value) {
        p = p.forwards[i];
      }
      update[i] = p;// use update save node in search path
    }

    // in search path node next node become new node forwords(next)
    for (int i = 0; i < level; ++i) {
      newNode.forwards[i] = update[i].forwards[i];
      update[i].forwards[i] = newNode;
    }

    // update node hight
    if (levelCount < level) levelCount = level;
  }

  public void delete(int value) {
    Node[] update = new Node[levelCount];
    Node p = head;
    for (int i = levelCount - 1; i >= 0; --i) {
      while (p.forwards[i] != null && p.forwards[i].data < value) {
        p = p.forwards[i];
      }
      update[i] = p;
    }

    if (p.forwards[0] != null && p.forwards[0].data == value) {
      for (int i = levelCount - 1; i >= 0; --i) {
        if (update[i].forwards[i] != null && update[i].forwards[i].data == value) {
          update[i].forwards[i] = update[i].forwards[i].forwards[i];
        }
      }
    }
  }

 // 理论来讲，一级索引中元素个数应该占原始数据的 50%，二级索引中元素个数占 25%，三级索引12.5% ，一直到最顶层。
  // 因为这里每一层的晋升概率是 50%。对于每一个新插入的节点，都需要调用 randomLevel 生成一个合理的层数。
  // 该 randomLevel 方法会随机生成 1~MAX_LEVEL 之间的数，且 ：
  //        50%的概率返回 1
  //        25%的概率返回 2
  //      12.5%的概率返回 3 ...
  private int randomLevel() {
    int level = 1;

    while (Math.random() < SKIPLIST_P && level < MAX_LEVEL)
      level += 1;
    return level;
  }

  public void printAll() {
    Node p = head;
    while (p.forwards[0] != null) {
      System.out.print(p.forwards[0] + " ");
      p = p.forwards[0];
    }
    System.out.println();
  }

  public class Node {
    private int data = -1;
    private Node forwards[] = new Node[MAX_LEVEL];
    private int maxLevel = 0;

    @Override
    public String toString() {
      StringBuilder builder = new StringBuilder();
      builder.append("{ data: ");
      builder.append(data);
      builder.append("; levels: ");
      builder.append(maxLevel);
      builder.append(" }");

      return builder.toString();
    }
  }

}
```



## 二、Redis 为什么选择跳表？

为什么 Redis 要用跳表来实现有序集合，而不是红黑树？

Redis 中的有序集合是通过跳表来实现的，严格点讲，其实还用到了散列表。

如果你去查看 Redis 的开发手册，就会发现，Redis 中的有序集合支持的核心操作主要有下面这几个：

- 插入一个数据；
- 删除一个数据；
- 查找一个数据；
- 按照区间查找数据（比如查找值在[100, 356]之间的数据）；
- 迭代输出有序序列。

其中，插入、删除、查找以及迭代输出有序序列这几个操作，红黑树也可以完成，时间复杂度跟跳表是一样的。但是，按照区间来查找数据这个操作，红黑树的效率没有跳表高。

对于按照区间查找数据这个操作，跳表可以做到 $O(logn)$ 的时间复杂度定位区间的起点，然后在原始链表中顺序往后遍历就可以了。这样做非常高效。

当然，Redis 之所以用跳表来实现有序集合，还有其他原因，比如，跳表更容易代码实现。虽然跳表的实现也不简单，但比起红黑树来说还是好懂、好写多了，而简单就意味着可读性好，不容易出错。还有，跳表更加灵活，它可以通过改变索引构建策略，有效平衡执行效率和内存消耗。



> 有序集合的英文全称明明是**sorted sets**，为啥叫zset呢？
>
> edis官网上没有解释，但是在Github上有人向作者提问了。作者是这么回答的哈哈哈
>
> Hello. Z is as in XYZ, so the idea is, sets with another dimension: the
> order. It’s a far association… I know 😃
>
> 原来前面的Z代表的是XYZ中的Z，最后一个英文字母，zset是在说这是比set有更多一个维度的set 😦
>
> 是不没道理？
>
> 更没道理的有个，Redis 默认端口 6379 ，因为作者喜欢的一个叫Merz的女明星，其名字在手机上输入正好对应号码6379，索性就把Redis的默认端口叫6379了…





Redis 的跳跃表共有 64 层，容纳 2^64 个元素应该不成问题。每一个 kv 块对应的结构如下面的代码中的`zslnode`结构，kv header 也是这个结构，只不过 value 字段是 null 值——无效的，score 是 Double.MIN_VALUE，用来垫底的。kv 之间使用指针串起来形成了双向链表结构，它们是 **有序** 排列的，从小到大。不同的 kv 层高可能不一样，层数越高的 kv 越少。同一层的 kv 会使用指针串起来。每一个层元素的遍历都是从 kv header 出发。

```c
struct zslnode {
  string value;
  double score;
  zslnode*[] forwards;  // 多层连接指针
  zslnode* backward;  // 回溯指针
}

struct zsl {
  zslnode* header; // 跳跃列表头指针
  int maxLevel; // 跳跃列表当前的最高层
  map<string, zslnode*> ht; // hash 结构的所有键值对
}
```



## 查找过程

设想如果跳跃列表只有一层会怎样？插入删除操作需要定位到相应的位置节点 (定位到最后一个比「我」小的元素，也就是第一个比「我」大的元素的前一个)，定位的效率肯定比较差，复杂度将会是 O(n)，因为需要挨个遍历。也许你会想到二分查找，但是二分查找的结构只能是有序数组。跳跃列表有了多层结构之后，这个定位的算法复杂度将会降到 O(lg(n))。



![img](https://user-gold-cdn.xitu.io/2018/7/27/164dc52ae7e6444c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



如图所示，我们要定位到那个紫色的 kv，需要从 header 的最高层开始遍历找到第一个节点 (最后一个比「我」小的元素)，然后从这个节点开始降一层再遍历找到第二个节点 (最后一个比「我」小的元素)，然后一直降到最底层进行遍历就找到了期望的节点 (最底层的最后一个比我「小」的元素)。

我们将中间经过的一系列节点称之为「搜索路径」，它是从最高层一直到最底层的每一层最后一个比「我」小的元素节点列表。

有了这个搜索路径，我们就可以插入这个新节点了。不过这个插入过程也不是特别简单。因为新插入的节点到底有多少层，得有个算法来分配一下，跳跃列表使用的是随机算法。

## 随机层数

对于每一个新插入的节点，都需要调用一个随机算法给它分配一个合理的层数。直观上期望的目标是 50% 的 Level1，25% 的 Level2，12.5% 的 Level3，一直到最顶层`2^-63`，因为这里每一层的晋升概率是 50%。

```
/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

不过 Redis 标准源码中的晋升概率只有 25%，也就是代码中的 ZSKIPLIST_P 的值。所以官方的跳跃列表更加的扁平化，层高相对较低，在单个层上需要遍历的节点数量会稍多一点。

也正是因为层数一般不高，所以遍历的时候从顶层开始往下遍历会非常浪费。跳跃列表会记录一下当前的最高层数`maxLevel`，遍历时从这个 maxLevel 开始遍历性能就会提高很多。

## 插入过程

下面是插入过程的源码，它稍微有点长，不过整体的过程还是比较清晰的。

```
/* Insert a new node in the skiplist. Assumes the element does not already
 * exist (up to the caller to enforce that). The skiplist takes ownership
 * of the passed SDS string 'ele'. */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    // 存储搜索路径
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    // 存储经过的节点跨度
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    // 逐步降级寻找目标节点，得到「搜索路径」
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        // 如果score相等，还需要比较value
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    // 正式进入插入过程
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    // 随机一个层数
    level = zslRandomLevel();
    // 填充跨度
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        // 更新跳跃列表的层高
        zsl->level = level;
    }
    // 创建新节点
    x = zslCreateNode(level,score,ele);
    // 重排一下前向指针
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }
    // 重排一下后向指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```

首先我们在搜索合适插入点的过程中将「搜索路径」摸出来了，然后就可以开始创建新节点了，创建的时候需要给这个节点随机分配一个层数，再将搜索路径上的节点和这个新节点通过前向后向指针串起来。如果分配的新节点的高度高于当前跳跃列表的最大高度，就需要更新一下跳跃列表的最大高度。

## 删除过程

删除过程和插入过程类似，都需先把这个「搜索路径」找出来。然后对于每个层的相关节点都重排一下前向后向指针就可以了。同时还要注意更新一下最高层数`maxLevel`。

## 更新过程

当我们调用 zadd 方法时，如果对应的 value 不存在，那就是插入过程。如果这个 value 已经存在了，只是调整一下 score 的值，那就需要走一个更新的流程。假设这个新的 score 值不会带来排序位置上的改变，那么就不需要调整位置，直接修改元素的 score 值就可以了。但是如果排序位置改变了，那就要调整位置。那该如何调整位置呢？

```
/* Remove and re-insert when score changes. */
    if (score != curscore) {
        zskiplistNode *node;
        serverAssert(zslDelete(zs->zsl,curscore,ele,&node));
        znode = zslInsert(zs->zsl,score,node->ele);
        /* We reused the node->ele SDS string, free the node now
        * since zslInsert created a new one. */
        node->ele = NULL;
        zslFreeNode(node);
        /* Note that we did not removed the original element from
        * the hash table representing the sorted set, so we just
        * update the score. */
        dictGetVal(de) = &znode->score; /* Update score ptr. */
        *flags |= ZADD_UPDATED;
        }
    return 1;
```

一个简单的策略就是先删除这个元素，再插入这个元素，需要经过两次路径搜索。Redis 就是这么干的。 不过 Redis 遇到 score 值改变了就直接删除再插入，不会去判断位置是否需要调整，从这点看，Redis 的 zadd 的代码似乎还有优化空间。关于这一点，读者们可以继续讨论。

## 如果 score 值都一样呢？

在一个极端的情况下，zset 中所有的 score 值都是一样的，zset 的查找性能会退化为 O(n) 么？Redis 作者自然考虑到了这一点，所以 zset 的排序元素不只看 score 值，如果 score 值相同还需要再比较 value 值 (字符串比较)。



## 元素排名是怎么算出来的？

前面我们啰嗦了一堆，但是有一个重要的属性没有提到，那就是 zset 可以获取元素的排名 rank。那这个 rank 是如何算出来的？如果仅仅使用上面的结构，rank 是不能算出来的。Redis 在 skiplist 的 forward 指针上进行了优化，给每一个 forward 指针都增加了 span 属性，span 是「跨度」的意思，表示从当前层的当前节点沿着 forward 指针跳到下一个节点中间跳过多少个节点。Redis 在插入删除操作时会小心翼翼地更新 span 值的大小。

```
struct zslforward {
  zslnode* item;
  long span;  // 跨度
}

struct zslnode {
  String value;
  double score;
  zslforward*[] forwards;  // 多层连接指针
  zslnode* backward;  // 回溯指针
}
```

这样当我们要计算一个元素的排名时，只需要将「搜索路径」上的经过的所有节点的跨度 span 值进行叠加就可以算出元素的最终 rank 值。





还见过面试问：MySQL 的 Innodb ，为什么不用 skiplist，而用 B+ Tree ？

如果是磁盘文件，b+Tree 会比 skiplist 好很多。磁盘查询性能比内存差很多，所以尽量减少查询的次数。
b+ tree 每个节点有好多数据，每次查询可以查询一批数据到内存中。b+ 树的层数低，可以减少访问磁盘的次数



## 小结

1. 各种搜索结构提高效率的方式都是通过空间换时间得到的。
2. 跳表最终形成的结构和搜索树很相似。
3. 跳表通过随机的方式来决定新插入节点来决定索引的层数。
4. 跳表搜索的时间复杂度是 $O(logn)$，插入/删除也是。

想到快排(quick sort)与其它排序算法（如归并排序/堆排序）虽然时间复杂度是一样的，但复杂度的常数项较小；跳表的原论文也说跳表能提供一个常数项的速度提升，因此想着常数项小是不是随机算法的一个特点？这也它们大放异彩的重要因素吧。



跳表使用空间换时间的设计思路，通过构建多级索引来提高查询的效率，实现了基于链表的“二分查找”。跳表是一种动态数据结构，支持快速地插入、删除、查找操作，时间复杂度都是 O(logn)。



> 来源：https://lotabout.me/2018/skip-list/



![](https://i02piccdn.sogoucdn.com/06eb28fd58fa8840)

## 参考

- [ftp://ftp.cs.umd.edu/pub/skipLists/skiplists.pdf](ftp://ftp.cs.umd.edu/pub/skipLists/skiplists.pdf) 原论文
- https://ticki.github.io/blog/skip-lists-done-right/ skip list 的一些变种、优化
- https://eugene-eeo.github.io/blog/skip-lists.html skip list 的一些相关复杂度分析
- http://cglab.ca/~morin/teaching/5408/refs/p90b.pdf skip list cookbook，算是 skip list 各方面的汇总
- [一个可以在有序元素中实现快速查询的数据结构](https://juejin.im/entry/59b0eed46fb9a0249471f357) 包含 skip list 的 C++ 实现
- [Redis内部数据结构详解(6)——skiplist](http://zhangtielei.com/posts/blog-redis-skiplist.html) 图文并茂讲解 skip list，可与本文交叉对照
- https://www.youtube.com/watch?v=2g9OSRKJuzM MIT 关于 skip list 的课程
- https://courses.csail.mit.edu/6.046/spring04/handouts/skiplists.pdf MIT 课程讲义