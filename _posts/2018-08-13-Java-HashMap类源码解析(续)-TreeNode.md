---
layout:     post
title:      Java HashMap类源码解析(续)-TreeNode
subtitle:   分析Map集合HashMap树结点TreeNode的实现原理
date:       2018-08-13
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 集合
---

　　由于TreeNode本身是红黑树的实现，所以在分析TreeNode的之前我还是摸了一篇算法导论里红黑树的读书笔记：[算法导论——红黑树](https://www.cnblogs.com/graywind/p/9468965.html?spm=a2c4e.11153940.blogcont625213.13.31e2453epdh3fD)，从伪代码行数也可以看出完整的红黑树的插入和删除操作代码是很长的，下面源码分析部分的行数就更多了，所以所谓手写红黑树画个图分析下逻辑还行，手写代码估计要写死(滑稽)

　　TreeNode从JDK8开始引入，作用是当HashMap解决冲突的链表长度超过了8时，生成一个红黑树来加速查找和插入，这里树结构存在并不影响本身依然存在线性链表结构，意思是Node.next这个属性依然有效，所以说树替换了线性链表依然还是链表法解决冲突，只不过链表的实现策略换了。当结点因为移除或分裂操作少于6个时，消除树结构。虽然生产树之后能加快查找插入和删除，但是建立和消除树本身是存在消耗的，所以在两个临界值之间来回插入和删除会导致开销快速增加。HashMap的源码分析见：[Java HashMap类源码解析](https://www.cnblogs.com/graywind/p/9457521.html)

　　红黑树是基于二叉搜索树扩展而来，对于TreeNode来说排序的依据是结点的hash值，若相等然后比较key值，若key不能比较或是相等则根据hash值，左儿子的hash值小于等于父亲，右儿子的hash值大于父亲。TreeNode 保有红黑树的性质：

1. 每个结点都是红色的或者是黑色的
2. 根结点是黑色的
3. 每个叶结点NIL是黑色的，但是通常我们不考虑NIL叶结点。
4. 如果一个结点是红色的，它的两个子结点都是黑色的
5. 每个结点到其他所有后代叶结点的简单路径上，均包含相同数目的黑色结点，这个属性被称为黑高，记作bh(x)

　　先来看一下TreeNode扩展的内部属性

```java
        TreeNode<K,V> parent;  //父亲结点

        TreeNode<K,V> left;    //左儿子

        TreeNode<K,V> right;   //右儿子

        TreeNode<K,V> prev;    //前方结点

        boolean red;//是否是红色
```

根据他的构造函数向上追溯TreeNode<K,V>继承了LinkedHashMap.Entry<K,V>而后者又继承了HashMap.Node<K,V>。所以TreeNode依然保有Node的属性，同时由于添加了prev这个前驱指针使得链表变为了双向的。

```java
    TreeNode(int hash, K key, V val, Node<K,V> next) {
       super(hash, key, val, next);
    }

    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

下面这个方法可以返回根结点，实现很简单就是不断从一个结点检查parent是否为null

```java
        /**
         * Returns root of tree containing this node.返回根结点
         */
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;//不断检查parent是否为null，为null的是根结点
            }
        }
```

moveRootToFront这个方法的作用是确保根结点被保存在了table数组上面，如果不是的话，就将root从链表中取出，将他放到数组对应的位置上，原本在数组上的结点链接到root的后面。这里最后调用了断言方法checkInvariants，作用是递归检查整棵树是否符合红黑树的性质，若检查不符会返回false导致moveRootToFront抛出错误。

```java
        /**
         * Ensures that the given root is the first node of its bin.
         * 确保给出的根结点是箱中的第一个结点也就是直接位于table上，原本的第一个结点若不是root则将root从链表中剪下放到第一个结点的前方
         */
        static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
            int n;
            if (root != null && tab != null && (n = tab.length) > 0) {
                int index = (n - 1) & root.hash;//根据root的hash值快速定位下标
                TreeNode<K,V> first = (TreeNode<K,V>)tab[index];//取出table[index]中的第一个结点
                if (root != first) {//root不是第一个结点
                    Node<K,V> rn;
                    tab[index] = root;//root放到table[index]位置
                    TreeNode<K,V> rp = root.prev;//rp=root的前一个结点
                    if ((rn = root.next) != null)//rn=root的后一个结点
                        ((TreeNode<K,V>)rn).prev = rp;//rn的前指针指向root的前一个结点
                    if (rp != null)
                        rp.next = rn;//rp的后指针指向root的后一个结点
                    if (first != null)
                        first.prev = root;//将原本的first放到root的后面
                    root.next = first;
                    root.prev = null;
                }
                assert checkInvariants(root);//assert后面的表达式为false时会抛出错误
            }
        }

        /**
         * Recursive invariant check
         * 从root开始递归检查红黑树的性质，仅在检查root是否落在table上时调用
         */
        static <K,V> boolean checkInvariants(TreeNode<K,V> t) {
            TreeNode<K,V> tp = t.parent, tl = t.left, tr = t.right,
                tb = t.prev, tn = (TreeNode<K,V>)t.next;
            if (tb != null && tb.next != t)
                return false;//t的前一个结点的后续应为t
            if (tn != null && tn.prev != t)
                return false;//t的后一个结点的前驱应为t
            if (tp != null && t != tp.left && t != tp.right)
                return false;//t因为t父亲的左儿子或右儿子
            if (tl != null && (tl.parent != t || tl.hash > t.hash))
                return false;//t的左儿子的hash值应小于t，父亲应为t
            if (tr != null && (tr.parent != t || tr.hash < t.hash))
                return false;//t的右儿子的hash值应大于t，父亲应为t
            if (t.red && tl != null && tl.red && tr != null && tr.red)
                return false;//t和t的儿子不能同时是红色
            if (tl != null && !checkInvariants(tl))
                return false;//递归检查t的左儿子
            if (tr != null && !checkInvariants(tr))
                return false;//递归检查t的右儿子
            return true;
        }
```

getTreeNode这个方法在HashMap中被多次使用，左右是寻找某个结点所在的树中是否有hash和key值符合的结点。我们可以看到这个方法一定会确保最后调用的是root.find()，也就是说find方法调用时this一定是根结点。所以无论最初调用getTreeNode的结点在树中处于什么位置，最后都会从根结点开始寻找，由于红黑树是相对平衡的二叉搜索树，所以可以认为搜索时间相比于链表从O(n)下降到了O(lgn)

```java

```

下面这个treeify就是根据链表生成树了，遍历链表获取结点，一个个插入到红黑树中，每次插入从根开始根据hash值寻找到叶结点位置进行插入，插入一个结点后调用一次balanceInsertion(root, x)检查x位置的红黑树性质是否需要修复。tieBreakOrder(k, pk)是在插入结点的key值k和父结点的key值pk无法比较出大小时，用于比较k和pk的hash值大小。关于红黑树性质的修复和保持稍后一起讨论。

```java

```

untreeify的作用就是把树转为链表，由于replacementNode这个方法会生成新的Node，所以产生的新链表不再具有树的信息了，原本的TreeNode被gc了。

```java
        /**
         * Calls find for root node.从根结点寻找h和k符合的结点
         */
        final TreeNode<K,V> getTreeNode(int h, Object k) {
            return ((parent != null) ? root() : this).find(h, k, null);
        }

        /**
         * Finds the node starting at root p with the given hash and key.
         * The kc argument caches comparableClassFor(key) upon first use
         * comparing keys.
         * 从根结点p开始根据hash和key值寻找指定的结点。kc是key的class
         */
        final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
            TreeNode<K,V> p = this;//该方法调用时this是根结点
            do {
                int ph, dir; K pk;
                TreeNode<K,V> pl = p.left, pr = p.right, q;
                if ((ph = p.hash) > h)
                    p = pl;//p.hash>参数hash时，移向左子树
                else if (ph < h)
                    p = pr;//p.hash<参数hash时，移向右子树
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;//p.hash=参数hash，且p.key与参数key相等找到指定结点并返回
                else if (pl == null)//若hash相等但key不等，向左右子树非空的一侧移动
                    p = pr;
                else if (pr == null)
                    p = pl;
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) &&//kc是否是一个可比较的类
                         (dir = compareComparables(kc, k, pk)) != 0)//比较k和p.key
                    p = (dir < 0) ? pl : pr;//k<p.key向左子树移动否则向右子树移动
                else if ((q = pr.find(h, k, kc)) != null)//这里开始的条件仅当输入k=null的时候才会进入，先检查右子树再检查左子树
                    return q;
                else
                    p = pl;
            } while (p != null);
            return null;
        }
```

拆分这个方法只有在resize的时候调用，可以对照线性链表扩展的情况，作用是把树拆成两棵，一棵放到新扩展出来的数组高位去，一棵留在原来的位置，划分的依据是扩展后新增的hash有效位是0还是1，拆分的时候会破坏树结构，所以先拆成两个链表再调用treeify来组装树。

```java
        /**
         * Splits nodes in a tree bin into lower and upper tree bins,
         * or untreeifies if now too small. Called only from resize;
         * see above discussion about split bits and indices.
         * 将树从给定的结点分裂成低位和高位的两棵树，若新树结点太少则转为线性链表。只有resize时会调用
         *
         * @param map the map
         * @param tab the table for recording bin heads存储链表头的hash表
         * @param index the index of the table being split需要分裂的表下标位置
         * @param bit the bit of hash to split on分裂时分到高位和低位的依据参数，实际使用时输入的是扩展之前旧数组的大小
         */
        final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            TreeNode<K,V> loHead = null, loTail = null;//低位头尾指针
            TreeNode<K,V> hiHead = null, hiTail = null;//高位头尾指针
            int lc = 0, hc = 0;//低位和高位的结点个数统计
            for (TreeNode<K,V> e = b, next; e != null; e = next) {//e从this开始遍历直到next为null
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                //这段决定了该结点被分到低位还是高位，依据算式是e.hash mod bit，由于bit是扩展前数组的大小，所以一定是2的指数次幂，所以bit一定只有一个高位是1其余全是0
                //这个算式实际是判断e.hash新多出来的有效位是0还是1，若是0则分去低位树，是1则分去高位树
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead != null) {
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);//分裂后的低位树结点太少转为线性链表
                else {
                    tab[index] = loHead;
                    if (hiHead != null) //若高位树为null则代表整棵树全保留在了低位，树没有变化所以不用进行后面的treeify
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {//这段与上面对于低位部分的分析相对应
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;//高位所处的位置为原本位置+旧数组的大小即bit
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }
```

下面开始要进入插入和删除操作部分分析了，为了便于说明把之前那篇几张关键的图贴过来方便与代码进行对照

### 旋转

如图所示是基本的左旋和右旋操作，这部分对着图看很容易理解

![rotate](/img/blog/2018-08-13/rotate.png)

```java
        //左旋操作，见图中右向左，p是x，r是y
        static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
            TreeNode<K,V> r, pp, rl;
            if (p != null && (r = p.right) != null) {//r是p的右儿子，也就是图中的y
                if ((rl = p.right = r.left) != null)//r的左儿子β成为p的右儿子
                    rl.parent = p;
                if ((pp = r.parent = p.parent) == null)//p的父亲成为r的父亲
                    (root = r).red = false;//若p是根结点,r的颜色改黑色
                else if (pp.left == p)//r取代p原本的位置
                    pp.left = r;
                else
                    pp.right = r;
                r.left = p;//p成为r的右儿子
                p.parent = r;
            }
            return root;
        }

        //右旋操作，见图中左向右，p是y，l是x
        static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr;
            if (p != null && (l = p.left) != null) {//l是p的左儿子，即图中x
                if ((lr = p.left = l.right) != null)//l的右儿子β成为p的左儿子
                    lr.parent = p;
                if ((pp = l.parent = p.parent) == null)//p的父亲成为l的父亲
                    (root = l).red = false;//若p是根结点,l的颜色改黑色
                else if (pp.right == p)//l取代p原本的位置
                    pp.right = l;
                else
                    pp.left = l;
                l.right = p;//p成为l的右儿子
                p.parent = l;
            }
            return root;
        }
```

### 插入

插入分为如图所示的3中情况，具体描述和处理方法见文章最前面的链接。

![insert1](/img/blog/2018-08-13/insert1.png)情况1

![insert2](/img/blog/2018-08-13/insert2.png)

插入操作涉及到两段很长的代码，首先是putTreeVal只要有h和k值符合的结点就不做插入，这里k必须是==或者equals才算是相等，返回找到的结点由调用的方法修改已有结点的value值，否则插入一个新结点并返回null。前面提到过，结点在树中的排序按照hash值大小，再按照key的大小，最后比较key计算Hash的大小进行排列，对应方法中的查找逻辑。

```java
        final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            TreeNode<K,V> root = (parent != null) ? root() : this;//this.parent为null代表已是根结点，否则通过root()获取根结点
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if ((ph = p.hash) > h)//p.hash>h时dir=-1，p.hash<h时dir=1
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;//p.hash=h且p.key与k相等时，已存在k值对应的结点则返回
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {///k是不可比较的类或者k和p.key通过compareTo比较相等
                    if (!searched) {
                        //这部分只会在在k和p.key通过compareTo比较相等时执行一次，若未能在在左右子树中寻找到k==p.key或者k.equals(p.key)的情况则下次不会再进入
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;//从p的左子树或者右子树中找到符合条件的结点则返回
                    }
                    dir = tieBreakOrder(k, pk);//比较k和p.key的hash值大小，-1表示k<p.key
                }
                //p.hash=hash但是key值不相等且p的左右子树中也没有找到符合的结点
                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {//找到了新增结点该插入的位置
                    Node<K,V> xpn = xp.next;//链表关系上的下一个结点
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);//新建一个结点，插入到p与它在链表上的下一个结点之间
                    if (dir <= 0)//根据dir大小把p的左儿子或者右儿子设为新增的结点
                        xp.left = x;
                    else
                        xp.right = x;
                    xp.next = x;
                    x.parent = x.prev = xp;
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                    moveRootToFront(tab, balanceInsertion(root, x));//插入后进行红黑树的性质修复，并检查root是否是直接在table数组上
                    return null;
                }
            }
        }
```

 balanceInsertion是插入后用于维持红黑树性质的修复操作，这里涉及到了上面图中展示的3中情况不同的操作

```java
        static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
            x.red = true;//插入的结点设为红色
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                if ((xp = x.parent) == null) {
                    x.red = false;//x的父亲为null代表x是根结点，x改黑色直接结束
                    return x;
                }
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;//若x的父结点为黑色或者x的父亲为根结点(实际上根应该是黑色)插入红色结点不影响红黑树性质
                if (xp == (xppl = xpp.left)) {//若x的父亲为左儿子
                    if ((xppr = xpp.right) != null && xppr.red) {
                        //xppr为x的叔叔，且叔叔为红色，图中的情况1，x的叔叔和父亲改为红色，x的爷爷改为黑色，x指针上移到爷爷的位置
                        xppr.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.right) {
                            //情况2，x的叔叔是黑色且x是右儿子。对x上升至父亲后执行一次左旋
                            root = rotateLeft(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            //情况3，x的叔叔是黑色且x是左儿子。x的父亲改黑色，x的爷爷改红色后对x的爷爷进行右旋
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }
                else {//以下为对称的操作
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }
```

### 删除

最后是删除操作部分，删除操作需要寻找一个后驱结点来顶替原结点的位置，在结点无儿子时删除后不需做其他调整，结点只有一个儿子时那个儿子是后驱，否则右子树中的最小结点作为后驱。

```java
        /**
         * Removes the given node, that must be present before this call.
         * This is messier than typical red-black deletion code because we
         * cannot swap the contents of an interior node with a leaf
         * successor that is pinned by "next" pointers that are accessible
         * independently during traversal. So instead we swap the tree
         * linkages. If the current tree appears to have too few nodes,
         * the bin is converted back to a plain bin. (The test triggers
         * somewhere between 2 and 6 nodes, depending on tree structure).
         * 移除给定的结点，这个方法相比一般的红黑树删除更加杂乱，因为我们无法交换内部结点的内容他们被next指针给限制了，这个指针是在遍历的时候独立的。
         * 因此我们交换树的连接。如果当前的树结点太少，需要转换为线性链表，通常这个值设定为2-6个结点
         */
        final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                                  boolean movable) {
            int n;
            if (tab == null || (n = tab.length) == 0)
                return;
            int index = (n - 1) & hash;//index = hash mod n
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
            TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;//succ指向要删除结点的后一个点，pred指向要删除结点的前一个
            if (pred == null)
                tab[index] = first = succ;//若要删除的结点的前一个为空，则first和tab[index]都指向要删除结点的后一个结点
            else
                pred.next = succ;//若要删除结点的前驱非空，则前一个结点的next指针指向该结点的后驱
            if (succ != null)
                succ.prev = pred;//后驱结点不为空时，后驱结点的前置指针设为删除结点的前置结点
            if (first == null)
                return;//若删除的结点是树中的唯一结点则直接结束
            if (root.parent != null)
                root = root.root();//确保root指向根结点
            if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                tab[index] = first.untreeify(map);  // 根自身或者左右儿子其中一个为空说明结点数过少（不超过2）转为线性表并结束
                return;
            }
            TreeNode<K,V> p = this, pl = left, pr = right, replacement;//p指向要删除的结点
            if (pl != null && pr != null) {
                TreeNode<K,V> s = pr, sl;
                while ((sl = s.left) != null) //删除结点的左右儿子都不为空时，寻找右子树中最左的叶结点作为后继，s指向这个后继结点
                    s = sl;
                boolean c = s.red; s.red = p.red; p.red = c; //交换后继结点和要删除结点的颜色
                TreeNode<K,V> sr = s.right;
                TreeNode<K,V> pp = p.parent;
                if (s == pr) { 
                    p.parent = s;//p是s的直接右儿子，交换p和s的位置
                    s.right = p;
                }
                else {
                    TreeNode<K,V> sp = s.parent;
                    if ((p.parent = sp) != null) {
                        if (s == sp.left)
                            sp.left = p;//p放到s原本的位置
                        else
                            sp.right = p;
                    }
                    if ((s.right = pr) != null)
                        pr.parent = s;//s放到p原本的位置
                }
                p.left = null;
                if ((p.right = sr) != null)
                    sr.parent = p;//s原本的右子树成为p的右子树
                if ((s.left = pl) != null)
                    pl.parent = s;//s原本的左子树成为p的左子树
                if ((s.parent = pp) == null)
                    root = s;//若p原本是根则新的根是s
                else if (p == pp.left)
                    pp.left = s;//若p是某个结点的左儿子，则s成为该结点的左儿子
                else
                    pp.right = s;//若p是某个结点的右儿子，则s成为该结点的右儿子
                if (sr != null)//若s结点有右儿子（s一定没有左儿子），则replacement为这个右儿子否则为p
                    replacement = sr;
                else
                    replacement = p;
            }
            else if (pl != null)//若p的左右儿子有一方为null，则replacement为非空的一方，否则为p自己
                replacement = pl;
            else if (pr != null)
                replacement = pr;
            else
                replacement = p;
            if (replacement != p) {//p有儿子或者s有儿子
                TreeNode<K,V> pp = replacement.parent = p.parent;
                if (pp == null)//用replacement来替换p
                    root = replacement;
                else if (p == pp.left)
                    pp.left = replacement;
                else
                    pp.right = replacement;
                p.left = p.right = p.parent = null;//移除p结点
            }
            //以replacement为中心，进行红黑树性质的修复，replacement可能为s的右儿子或者p的儿子或者p自己
            TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

            if (replacement == p) {  //p没有儿子或者s没有儿子，直接移除p
                TreeNode<K,V> pp = p.parent;
                p.parent = null;
                if (pp != null) {
                    if (p == pp.left)
                        pp.left = null;
                    else if (p == pp.right)
                        pp.right = null;
                }
            }
            if (movable)
                moveRootToFront(tab, r);//整理根结点
        }
```

![delete](/img/blog/2018-08-13/delete.png)

 

同样有删除和删除之后维持红黑树性质的修复操作，这里涉及到图中展示的4种不同情况的操作 

```java
        static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root,
                                                   TreeNode<K,V> x) {
            for (TreeNode<K,V> xp, xpl, xpr;;)  {
                if (x == null || x == root)
                    return root;//删除结点为空或者删除的是根结点，直接返回
                else if ((xp = x.parent) == null) {
                    x.red = false;//删除后x成为根结点，x的颜色改为黑色
                    return x;
                }
                else if (x.red) {
                    x.red = false;//将一个红色的结点提升到删除结点的位置不会改变黑高
                    return root;
                }
                else if ((xpl = xp.left) == x) {//x的父亲是左儿子
                    if ((xpr = xp.right) != null && xpr.red) {
                        //情况1，x的兄弟是红色的
                        xpr.red = false;
                        xp.red = true;
                        root = rotateLeft(root, xp);
                        xpr = (xp = x.parent) == null ? null : xp.right;
                    }
                    if (xpr == null)
                        x = xp;//若x没有兄弟，x上升到父亲的位置
                    else {
                        TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                        if ((sr == null || !sr.red) &&
                            (sl == null || !sl.red)) {
                            //情况2，x兄弟是黑色，他的两个儿子是黑色的
                            xpr.red = true;
                            x = xp;
                        }
                        else {
                            if (sr == null || !sr.red) {
                                //情况3，x兄弟是黑色，他的右儿子是黑色，左儿子红色
                                if (sl != null)
                                    sl.red = false;
                                xpr.red = true;
                                root = rotateRight(root, xpr);
                                xpr = (xp = x.parent) == null ?
                                    null : xp.right;
                            }
                            //情况4
                            if (xpr != null) {
                                xpr.red = (xp == null) ? false : xp.red;
                                if ((sr = xpr.right) != null)
                                    sr.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateLeft(root, xp);
                            }
                            x = root;
                        }
                    }
                }
                else { //以下为对称操作
                    if (xpl != null && xpl.red) {
                        xpl.red = false;
                        xp.red = true;
                        root = rotateRight(root, xp);
                        xpl = (xp = x.parent) == null ? null : xp.left;
                    }
                    if (xpl == null)
                        x = xp;
                    else {
                        TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                        if ((sl == null || !sl.red) &&
                            (sr == null || !sr.red)) {
                            xpl.red = true;
                            x = xp;
                        }
                        else {
                            if (sl == null || !sl.red) {
                                if (sr != null)
                                    sr.red = false;
                                xpl.red = true;
                                root = rotateLeft(root, xpl);
                                xpl = (xp = x.parent) == null ?
                                    null : xp.left;
                            }
                            if (xpl != null) {
                                xpl.red = (xp == null) ? false : xp.red;
                                if ((sl = xpl.left) != null)
                                    sl.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateRight(root, xp);
                            }
                            x = root;
                        }
                    }
                }
            }
        }
```

 