---
title: TreeMap源码实现分析
date: 2019-08-19 12:50:59
categories:
- Core Java
tags:
- Java Collections
- TreeMap
- Red-Black Tree
---

有序的哈希表，允许被插入的元素按照自然顺序或者特殊指定的比较器的顺序进行排序，底层基于红黑树的数据结构实现，支持`O(log n)`时间复杂度进行`get`,`put`,`remove`等些操作

<!-- more -->

### 基本介绍
- 基于{% post_link learn-about-red-black-tree %}`NavigableMap接口`的实现，map按照自然顺序排序或者按照在构建map时提供的比较器(`Comparator`)来进行排序
- 实现为`containsKey`,`get`,`put`,`remove`操作提供`log(n)`算法复杂度的支持
- 注意，如果要正确实现`map`接口，维护`tree map`的顺序，必须保证要跟`equals`一致。这是因为`map接口`是根据`equals`操作定义的，但是`sorted tree`使用的是`compareTo 或者 compare`用来比较所有的`key`.
- 非同步的，可以使用`SortedMap m = Collections.synchronizedSortedMap(new TreeMap(...));`将其装饰为一个支持同步操作的有序集合
- 迭代器返回的所有方法(`collection view methods`)都是`fail-fast`的，如果`map`内部结构在返回迭代器后发生变化，在使用该迭代器时将抛出`ConcurrentModicationExcetion`异常

### 常用方法

#### put
```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    if (t == null) {
        compare(key, key); // type (and possibly null) check 这个是做什么?

        root = new Entry<>(key, value, null);//如果红黑树为空，新建结点，并设置该结点为红黑树的根节点
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {//如果在构建TreeMap时有传Comparator参数，则使用Comparator的compare方法来比较各个key的大小
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);//如果之前已存在该key,则更新对应结点的值并返回旧值
        } while (t != null);
    }
    else {//否则使用key自带的compareTo比较方法，需要key实现Comparable接口
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);//如果之前已存在该key,则更新对应结点的值并返回旧值
        } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent);
    // 将新的Mapping添加入红黑树
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    // 为了维护红黑树相关的性质，在插入新结点后需要进行调整
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}

```

#### get
```java
public V get(Object key) {
    Entry<K,V> p = getEntry(key);
    return (p==null ? null : p.value);
}

final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)//如果比较器不为空，则通过比较器的方式获取值
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while (p != null) {//二叉搜索树查询
        int cmp = k.compareTo(p.key);//使用Key自带的compateTo方法
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}

final Entry<K,V> getEntryUsingComparator(Object key) {
        @SuppressWarnings("unchecked")
            K k = (K) key;
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            Entry<K,V> p = root;
            while (p != null) {//二叉搜索树的查询
                int cmp = cpr.compare(k, p.key);
                if (cmp < 0)
                    p = p.left;
                else if (cmp > 0)
                    p = p.right;
                else
                    return p;
            }
        }
        return null;
    }
```

#### remove
```java
public V remove(Object key) {
    Entry<K,V> p = getEntry(key);//查询对应的key是否在红黑树上
    if (p == null)
        return null;

    V oldValue = p.value;
    deleteEntry(p);//删除找到的结点p
    return oldValue;
}

// 跟算法导论中的红黑树删除貌似不太一样.
 private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    if (p.left != null && p.right != null) {
        Entry<K,V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);

    if (replacement != null) {
        // Link replacement to parent
        replacement.parent = p.parent;
        if (p.parent == null)
            root = replacement;
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        else
            p.parent.right = replacement;

        // Null out links so they are OK to use by fixAfterDeletion.
        p.left = p.right = p.parent = null;

        // Fix replacement
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        root = null;
    } else { //  No children. Use self as phantom replacement and unlink.
        if (p.color == BLACK)
            fixAfterDeletion(p);

        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```

### 参考
- `jdk-11.0.2`
- 算法导论第13章