---
layout: post
title: 基于LinkedHashMap实现LRU
tags: [JAVA核心, ]

---

**LRU（Least Recently Used）**是内存管理中的一种页面置换算法，在工程中，也往往被用作缓存的淘汰算法，如在Redis中用来选择缓存中需要淘汰的值。  

它的算法核心思想是：
+ 写入、读取缓存时，该缓存都算作被访问一次  
+ 缓存被访问时，更新其访问时间  
+ 当缓存大小超过容量时，淘汰更新时间最远的缓存  

具体实现的时候，我们不需要记录其访问时间，可以通过队列来标识访问顺序，比如将最近访问的节点插入队头，那么每次淘汰的时候都是选择淘汰队尾节点。  

**LinkedHashMap**是Java核心内库里实现的一种有序哈希表，继承自HashMap，并实现了Map接口。它与HashMap在结构上主要有两点区别：
+ LinkedHashMap多了两个Entry节点：before、after，以实现链表的双向遍历  
+ 引入了对节点顺序的控制，新增了一个**accessOrder**布尔属性：当它为false时，和HashMap一样，新插入的节点在发生哈希冲突时通过拉链法存在链尾；当它为true时，每次访问（get\put）某个节点时，都会将这个节点重新放置于链尾。  

在LeetCode上刷LRU的时候，看到一种绝妙的算法，提交者直接利用LinkedHashMap，短短十来行便实现了LRU。  

他的算法如下：
```java
class LRUCache extends LinkedHashMap<Integer, Integer> {
    private int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    public int get(int key) {
        return super.getOrDefault(key, -1);
    }

    public void put(int key, int value) {
        super.put(key, value);
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity;
    }
}
```

基本上，这位大佬啥都没干，把核心的处理逻辑都交给了LinkedHashMap自己来处理。  

比对下自己的代码，为了实现LRU，我定义的数据结构如下：
```java
    ListNode cacheListHead;
    ListNode cacheListTail;

    Map<Integer, Integer> cache = new HashMap<>();
    int count = 0;
    int capacity;
```
ListNode是我构造的链表节点，我采取的是维护一个访问队列的方式，来“标识”访问顺序，最近访问节点插入队头，每次淘汰队尾节点。cache这个HashMap用来存储键值对缓存，count标识当前缓存数据数目，capacity代表缓存总容量。  

仔细思考下，每次我get数据的时候，都要从cacheListHead开始遍历到队尾，如果以空间换时间，将cache对象进行下改造：其Key仍旧代表缓存数据的Key值，但Value不直接存储缓存的数据，而是改成存储ListNode，那么我就可以直接通过Key来获取到相应的节点。
```java
    ListNode cacheListHead;
    ListNode cacheListTail;

    Map<Integer, ListNode> cache = new HashMap<>();
    int count = 0;
    int capacity;
```

这样，通过Key值查找对应节点的算法复杂度直接降成了O(1)。但此时还有一个问题，由于一开始我设计的ListNode只是单向节点，这样即便我找到了对应的节点，仍旧无法快速的调整这个节点的位置。所以我将ListNode改成了双向节点：
```java
    public static class ListNode {
        public int val;
        public ListNode pre;
        public ListNode next;
        public ListNode(int val, ListNode pre, ListNode next) {
            this.val = val;
            this.pre = pre;
            this.next = next;
        }
    }
```
这样一来，我实现方法中的数据结构不也变成了一个LinkedHashMap么！！！  

接下来，我研究了下LinkedHashMap的实现。  

LinkedHashMap有个三参数的构造函数，
```java
    public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```
initialCapacity初始容量，和HashMap一致，初始容量是16；loadFactor加载因子，默认0.75，用于求取扩容阈值，扩容阈值 = 容量 * 加载因子；accessOrder指定排序方式：true按访问顺序排序，false按插入顺序排序。  

在LinkedHashMap中没有重写HashMap的put方法，只是在HashMap的对象发起put的时候，会调用afterNodeAccess()方法，LinkedHashMap重写了afterNodeAccess()
```java
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
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
可以看到，当accessOrder为true的时候，会将访问的节点移到链尾。  

而在get的时候，LinkedHashMap重写了HashMap的get方法，
```java
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```
其原理与get一致，也是在最后通过afterNodeAccess()方法来重置节点的位置。  

在搞清了LinkedHashMap如何调整节点访问顺序之后，就到了最关键的地方了：如何按照LRU算法的要求淘汰最近最少使用的数据？  

在HashMap put处理逻辑的最后，HashMap调用了afterNodeInsertion()方法，而LinkedHashMap重写了这个方法。
```java
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
```
在可以看到大佬的代码里面重写了removeEldestEntry()方法。当当前数据节点数目大于cache容量的时候，便会移除跳跃表中的首节点。根据LinkedHashMap最新访问节点置于链表尾部的规则，链首节点刚好是LRU算法规则中需要被淘汰的节点。

**总结：**  
LinkedHashMap本身的特质，让它可以作为LRU算法绝佳的实践。

> **TIPS：HashMap的扩容规则**  
> 容量：存储值的节点数组大小  
> 扩容阈值：put之后，若当前数据节点数大于这个值，便会开始扩容  
>   
> HashMap的扩容规则主要定义在resize()方法里，LinkedHashMap直接继承了这个方法，大致规则如下：  
> 1、新的扩容阈值初始化为0  
> 2、旧容量大于等于`\( 2^{ 30 } \)`时，扩容阈值提升为`\( 2^{ 31 } \)`  
> 3、旧容量小于`\( 2^{ 30 } \)`时，容量扩容2倍，此时若旧容量的2倍大于等于16但小于`\( 2^{ 30 } \)`，扩容阈值也提升至旧阈值的2倍  
> 4、旧容量小于等于0，且旧扩容阈值大于0，容量扩容为旧扩容阈值大小  
> 5、旧容量小于等于0，且旧扩容阈值小于等于0，容量设置为16，新扩容阈值 = 初始容量 * 加载因子（暂时发现只有调用无参数构造函数时会存在这种情况）  
> 6、若经过上述判断处理，得到的新扩容阈值仍为0，则若新容量小于`\( 2^{ 30 } \)`且新容量 * 加载因子也小于`\( 2^{ 30 } \)`，则新扩容阈值 = 新容量 * 加载因子；否则，新扩容阈值 = `\( 2^{ 31 } \)`  
> 7、新建一个大小为新容量值的节点数组，在此基础上，对旧节点数组的数据进行重新HASH计算,然后写入新数组中  