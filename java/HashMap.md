# HashMap

### 1 介绍

HashMap继承自AbstractMap<K, V>抽象类，实现了Map<K, V>、Serializable、Cloneable接口。

### 2 API

###### 2.1 构造方法

```java
HashMap() // 使用默认initial capacity(16)和默认load factor(0.75)构造一个空的HashMap
HashMap(int initialCapacity) // 使用指定的initial opacity和默认load factor(0.75)构造一个空的HashMap
HashMap(int initialCapacity, float loadFactor) // 使用指定的initial opacity和load factor构造一个空的HashMap
HashMap(Map<? extends K, ? extends V> m) // 使用指定的Map构造一个新的HashMap
```

###### 2.2 方法

```java
void clear() // 移除map所有内容
Object clone() // 返回一个HashMap实例的浅复制，不克隆键和值
V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) // 尝试为指定的键和它当前映射的值（如果没有映射就为null）计算一个映射
V computeIfAbsent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) // 如果指定的键的值存在且不为null，尝试给键和它当前的映射值计算一个新的映射。
boolean containsKey(Object key) // 如果map包含指定键的映射返回true
boolean containsValue(Object value) // 如果map有一个或多个键映射该指定的值，返回true
Set<Map.Entry<K, V>> entrySet() // 返回map包含的mapping的集合视图(Set view)
void forEach(BiConsumer<? super K, ? super V> action) // 为map里的每个条目执行指定的动作(action)直到处理完所有条目或action抛出exception为止
V get(Object key) // 返回指定的键映射的值，若map没有包含键的映射，返回null
V getOrDefault(Object key, V defaultValue) // 返回指定的键映射的值，若map没有包含键的映射，返回默认值
boolean isEmpty() // 如果map没有包含键值映射，返回true
Set<K> keySet() // 返回map包含的键的集合视图
V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction) // 若指定的键还没有关联值或关联的值为null，则用给定的非null值与它关联
V put(K key, V value) // 在map里用指定的键关联指定的值
void putAll(Map<? extends K, ? extends V> m) // 从指定的map复制全部映射到这个map
V putIfAbsent(K key, V value) // 若指定的键还没有关联值(或映射为null)，关联给定的值并返回null，否则返回当前值
V remove(Object key) // 从map里移除指定键存在的映射
boolean remove(Object key, Object value) // 只在指定的键当前映射到指定的值时移除对应的条目
V replace(K key, V value) // 只在指定的键当前映射到指定的值时替换对应的条目
boolean replace(K key, V oldValue, V newValue)
void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) // 用在每个条目上调用给定函数的结果替换每个条目的值，直到处理完所有条目或函数抛出exception
int size() // 返回map里键值映射的数量
Collection<V> values() // 返回map包含的值的Collection视图
```

### 3 使用示例

```Java
import java.util.HashMap;
import java.util.Map;

public class Application {

    public static void main(String[] args) {
        Map<String, String> map = new HashMap<>();
        map.put("a", "apple");
        String value = map.get("a");
        System.out.println(value);
    }
}

// "apple"
```

### 4 实现原理

###### 4.1 内部组成

HashMap内部有如下几个主要的实例变量：

```Java
transient Node<K,V>[] table;
transient int size; // 实际键值对个数
int threshold;
final float loadFactor;
```

table是一个Node类型的数组，称为哈希表或哈希桶，每个数组元素指向一个单向链表，链表中的每个节点表示一个键值对。Node是一个内部类，实例和构造方法代码如下：

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    
    	// ...
    }
```

其中，key和value分别表示键和值，next指向下一个Node节点，hash是key的hash值，待会我们介绍其计算方法。直接存储hash值是为了在比较的时候加快计算。

size表示实际键值对的个数。

threshold表示阈值，当键值对个数size大于等于threshold时考虑进行拓展。threshold等于table.length乘以loadFactor。比如，如果table.length为16，loadFactor为0.75，则threshold为12.

loadFactor是负载因子，表示整体上table被占用的程度，是一个浮点数，默认为0.75，可以通过构造方法进行修改。

###### 4.2 保存键值对

HashMap是如何把一个键值对保存起来的？

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

put方法调用putVal方法添加键值对。调用hash方法计算key的hash值。hash方法代码为：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

基于key自身的hashCode方法的返回值又进行了一些位运算，目的是为了随机和均匀性。

putVal方法的代码为：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n,i;
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

如果table为空，首先调用resize（）方法给table分配实际的空间。这时候调用的resize的主要代码为：

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // ...
    }
    else if (oldThr > 0) {
        // ...
    }
    else {
        newCap = DEFAULT_INITIAL_CAPACITY; // 16
        newThr = (int) DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY; // 0.75 * 16
    }
    // ...
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // ...
    return newTab;
}
```

默认情况下，capacity的值为16，threshold会变为12，table会分配一个长度为16的Node数组。

HashMap中，n = tab.length为2的幂次方，i = (n - 1) & h等同于求模运算h%n，结果i即为键值对保存位置，tab[i]指向一个单向链表。如果数组元素为空，调用newNode方法创建一个Node实例。

如果p = tab[i]有值，首先比较链表第一个元素。

比较的时候，是先比较hash值，hash值相同的时候，再使用equals方法进行比较。代码为：

```java
if (p.hash == hash &&
    ((k = p.key) == key || (key != null && key.equals(k))))
    e = p;
```

hash是整数，比较的性能一般要比equals高很多，hash不同，就没有必要调用equals方法了，这样整体上可以提高性能。

如果Node是TreeNode实例。（待补充）

如若不是，则在链表中逐个查找是否已经有这个键了，遍历代码为：

```Java
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
```

如果找不到，则调用newNode方法添加一个节点。如果能找到，直接修改Node中value的值就行了。++modCount的含义是记录修改次数，方便在迭代中检测结构性变化。如果size超过阈值了，则调用resize方法对table进行扩展，扩展的策略是乘2，这时候调用 resize方法的主要代码为：

```Java
final Node<K,V>[] resize() {
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) { // 1 << 30
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // ...
    threshold = newThr;
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
	table = newTab;
    if (oldTab != null) {
        // 移植至新table
    }
    return newTab;
}
```

分配一个容量为原来两倍的Node数组，更新内部的table变量，以及threshold的值。然后把原来的键值对移植过来。

以上就是保存键值对的主要代码，简单总结一下，基本步骤为：

1）计算键的hash值；

2）根据hash值得到保存位置（取模）；

3）插到对应位置的链表尾部或更新已有值；

4）根据需要扩展table大小。

###### 4.3 查找方法

根据键获取值的get方法的代码为：

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

首先调用hash方法获取key的hash值，然后调用getNode方法获取键值对节点node，然后调用节点的value获取值。getNode方法代码为：

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V> tab; Node<K,V> first, e; int n; K k;
    if ((tab == table) != null && (n = tab.length) > 0 && (first = tab[(n-1) & hash]) != null) {
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

若hash对应的数组元素为空，则返回null。若有值，则首先检查链表（或树）的第一个节点。首先判断hash是否相等，若hash一致，再调用equals方法比较key，若key也一致，则直接返回首节点。

若首节点不匹配，且链表（或树）不止一个节点，则继续查找。

若首节点first为TreeNode实例，则返回getTreeNode方法查找结果。否则遍历链表余下所有节点，若找到匹配的节点，则直接返回该节点，否则返回null。

HashMap的get方法总结：

1）首先，调用hash方法计算key的hash值；

2）然后，调用getNode方法，根据hash和key查找匹配节点；

3）最后，返回查找到的节点的value，若未找到对应节点，则返回null。

###### 4.4 根据键删除键值对

remove方法的代码为：

```java
public remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ? null : e.value;
}
```

首先，调用hash方法计算key的hash值，hash方法代码为：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

key为null，则hash为0，否则调用key的hashCode方法获取hashCode，按位异或hashCode无符号右移16位的结果，最后得到key的hash值。

然后调用removeNode方法查找并移除对应节点，removeNode方法代码为：

```java
final Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 && (p = tab[index = (n -1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if (( e = p.next) != null) {
            if (p instanceof treeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value || (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modeCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

对hash进行取模，得到链表对应数组中位置。若该数组位置为空，则返回null。若数组对应位置有值，则先检查首节点。若首节点不匹配，则继续检查后续节点。若首节点为TreeNode实例，则调用TreeNode的实例方法getTreeNode获取对应node。否则遍历链表余下节点，找出匹配的节点。

若为找到匹配的节点则返回null。否则，若node为TreeNode实例，则调用ThreeNode的实例方法removeTreeNode移除该node。若node为链表首节点，则将对应数组元素指向链表next节点。否则将链表匹配节点node的前一个节点的next变量指向node的后一个节点。

然后modeCount记录结构化修改，更新键值对数量，并返回移除的节点。