## 0x00. 概述

 LinkedHashMap 是 HashMap 的子类，与 HashMap 有着同样的存储结构，但它加入了一个双向链表的头结点，将所有 put 到 LinkedHashmap 的结点串成了一个双向循环链表，因此它保留了节点插入的顺序，可以使节点的输出顺序与输入顺序相同。先来跑个 Demo 直观地感受一下它到底能做什么：

```java
public static void main(String args[]) {
    System.out.print("LinkedHashMap：");
    Map<Integer,String> map = new LinkedHashMap<>();
    map.put(6, "apple");
    map.put(3, "banana");
    map.put(2,"pear");

    for (Integer key : map.keySet()) {
        System.out.print("(" + key + ", " + map.get(key) + ") ");
    }

    System.out.print("\nHashMap：");
    Map<Integer,String> map1 = new HashMap<>();
    map1.put(6, "apple");
    map1.put(3, "banana");
    map1.put(2,"pear");

    for (Integer key : map1.keySet()) {
        System.out.print("(" + key + ", " + map.get(key) + ") ");
    }
}
```

输出结果如下，可以看到相比于 HashMap，LinkedHashMap 的遍历结果与键值对的插入顺序是一致的，下面翻下 LinkedHashMap  的源码看看它到底是怎么实现这一机制的。

```
LinkedHashMap：(6, apple) (3, banana) (2, pear) 
HashMap：(2, pear) (3, banana) (6, apple)
```

## 0x01. 源码

### 1. 类签名

LinkedHashMap 实现了 Map<K,V> 接口，另外它继承自 HashMap，也是非线程安全的。

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>{}
```

### 2. Entry 结点

LinkedHashMap 结点类型是基于 HashMap.Node 类型的，在继承的基础上采用了循环双向链表的结构连接两个相邻的键值对，同时类里有两个成员变量head 和 tail 分别指向内部双向链表的表头和表尾结点。

![img](/images/java/collections/linkedhashmap/entry.jpg)

乍一看代码可能会以为在 HashMap.Node 定义的 next  属性与此处的 after 属性冗余了，但是从上面的图中我们不难看出，next 指针是用于维护桶中结点的插入顺序，而 before 和 after 指针则是用于维护结点在整个哈希表中的插入顺序。

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

transient LinkedHashMap.Entry<K,V> head;
transient LinkedHashMap.Entry<K,V> tail;
```

### 3. 构造函数

与 HashMap 相比，LinkedHashMap 增加了一个`accessOrder`字段。用于控制迭代时的节点顺序，它的默认值是 false，表示迭代时输出的顺序与插入结点的顺序一致；若为 true，则表示输出的顺序是按照访问结点的顺序，可以利用这一性质设计 LRU 缓存。

```java
final boolean accessOrder;

public LinkedHashMap() {
    super();
    accessOrder = false;
}

//指定初始化时的容量，
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

//指定初始化时的容量，和扩容的加载因子
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

//指定初始化时的容量，和扩容的加载因子，以及迭代输出节点的顺序
public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}

//利用另一个 Map 来构建，
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    //批量插入一个 map 中的所有数据到 本集合中。
    putMapEntries(m, false);
}
```

### 4. 添加键值对

LinkedHashMap 没有重写父类中任何 put 方法。但是其重写了构建新节点的 newNode() 方法. 
，newNode() 会在 HashMap 的 putVal() 方法里被调用。先回头看看 HashMap 的 putVal()  方法，它会在批量插入数据 putMapEntries 方法或者插入单个数据 put 方法时被调用。先看看 HashMap  的 putVal() 方法

```java
// HashMap.putVal
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

首先，在上面代码块的很多处都能看到 对 newNode() 方法的调用，在HashMap 中，该方法只是创建并返回一个新的 Node 对象，而 LinkedHashMap 重写了 newNode()，在每次构建新节点时，通过 linkNodeLast(p); 将新节点链接在内部双向链表的尾部。

```java
// HashMap.newNode
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
    return new Node<>(hash, key, value, next);
}

// LinkedHashMap.newNode
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

// LinkedHashMap.linkNodeLast
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```

另外可以看到，当要在哈希表中插入结点时发现已存在键相同的键值对时，将更新键值对的值，如果是普通的哈希表在 afterNodeAccess 方法中不会做任何事情，当时如果是 LinkedHashMap  的话 afterNodeAccess 将根据 accessOrder 等决定是否更新表示插入/访问顺序的双向链表。同样地，在 putVal 方法的最后当新的键值对插入哈希表时，将 afterNodeInsertion 方法来维护双向链表中的结点顺序。

#### afterNodeAccess 函数

在 HashMap 中定义了几个对于 LinkedHashMap 而言很关键的函数，它们会在添加或删除键值对的时候其作用，在 HashMap 中默认是空实现，把具体实现留给 LinkedHashMap。

```java
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

来看看 afterNodeAccess 函数，如果将 accessOrder 设置为 true，就可以保证最近访问节点放到最后，在进行 put 之后就算是对节点的访问了，那么这个时候就会更新链表，先把该结点从双向链表中删除，然后再把它放到最后。

```java
// LinkedHashMap.afterNodeAccess
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        // 断开链接
        if (a != null)
            a.before = b;
        else
            last = b;
        // 插入结点
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

#### afterNodeInsertion 函数

afterNodeInsertion 方法的 evict 参数如果为 false，表示哈希表处于创建模式，而只有在使用 Map 集合作为构造器创建 LinkedHashMap 或 HashMap 时才为 false，使用其他构造器创建的 LinkedHashMap 和之后再调用 put方法，该参数均为 true。LinkedHashMap 的 afterNodeInsertion() 实现如下

```java
// LinkedHashMap.afterNodeInsertion
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```

从上面可以看到，如果要进入到if语句块中需要同时满足三个条件： 

1. evict 为 true。只有当使用插入 Map 集合的构造方法时才为 false 
2. first! != null。表明表不为空，一般地，当调用该方法时，哈希表不会为空 
3. removeEldestEntry() 方法返回 true。该方法删除最老的节点

removeElestEntry 用于定义删除最老元素的规则。默认返回  false，当覆写该方法并在某种条件下返回 true 时，将调用 removeNode 删除最老节点。

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

### 5. 删除键值对

与添加键值对类似，删除键值对的主体方法在 HashMap 中定义，而对于 LinkedHashMap，删除的过程并不复杂，主要有三步

1. 根据 hash 定位到桶位置
2. 遍历链表或调用红黑树相关的删除方法
3. 从 LinkedHashMap 维护的双链表中移除要删除的节点

```java
// HashMap 中实现
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

// HashMap 中实现
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        ......
        ......
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode) {...}
            // 将要删除的节点从单链表中移除
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);    // 调用删除回调方法进行后续操作
            return node;
        }
    }
    return null;
}
```

#### afterNodeRemoval 函数

重点还是在于 afterNodeRemoval  方法，如果用户定义了`removeEldestEntry`的规则，那么便可以执行相应的移除操作。

```java
// LinkedHashMap.afterNodeRemoval
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```

