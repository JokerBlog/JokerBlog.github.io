---
layout:     post
title:      "图解LinkedHashMap原理"
subtitle:   ""
date:       2019-07-09 09:47:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

## 1 前言

LinkedHashMap继承于HashMap，如果对HashMap原理还不清楚的同学，请先看上一篇：[图解HashMap原理](https://www.jianshu.com/p/dde9b12343c1)



## 2 LinkedHashMap使用与实现

先来一张LinkedHashMap的结构图，不要虚，看完文章再来看这个图，就秒懂了，先混个面熟：

![](https://upload-images.jianshu.io/upload_images/4843132-7abca1abd714341d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/739/format/webp)

### 2.1 应用场景

HashMap是无序的，当我们希望有顺序地去存储key-value时，就需要使用LinkedHashMap了。

```java
   Map<String, String> hashMap = new HashMap<String, String>();
        hashMap.put("name1", "josan1");
        hashMap.put("name2", "josan2");
        hashMap.put("name3", "josan3");
        Set<Entry<String, String>> set = hashMap.entrySet();
        Iterator<Entry<String, String>> iterator = set.iterator();
        while(iterator.hasNext()) {
            Entry entry = iterator.next();
            String key = (String) entry.getKey();
            String value = (String) entry.getValue();
            System.out.println("key:" + key + ",value:" + value);
        }
```

![](https://upload-images.jianshu.io/upload_images/4843132-1c5470069f9626e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/335/format/webp)



我们是按照xxx1、xxx2、xxx3的顺序插入的，但是输出结果并不是按照顺序的。

同样的数据，我们再试试LinkedHashMap



```java
     Map<String, String> linkedHashMap = new LinkedHashMap<>();
        linkedHashMap.put("name1", "josan1");
        linkedHashMap.put("name2", "josan2");
        linkedHashMap.put("name3", "josan3");
        Set<Entry<String, String>> set = linkedHashMap.entrySet();
        Iterator<Entry<String, String>> iterator = set.iterator();
        while(iterator.hasNext()) {
            Entry entry = iterator.next();
            String key = (String) entry.getKey();
            String value = (String) entry.getValue();
            System.out.println("key:" + key + ",value:" + value);
        })
```

![](https://upload-images.jianshu.io/upload_images/4843132-2688396af2d206d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/335/format/webp)

结果可知，LinkedHashMap是有序的，且默认为插入顺序。

### 2.2 简单使用

跟HashMap一样，它也是提供了key-value的存储方式，并提供了put和get方法来进行数据存取。

```java
        LinkedHashMap<String, String> linkedHashMap = new LinkedHashMap<>();
        linkedHashMap.put("name", "josan");
        String name = linkedHashMap.get("name");
```

### 2.3 定义

LinkedHashMap继承了HashMap，所以它们有很多相似的地方。

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
{
```

### 2.4 构造方法

![](https://upload-images.jianshu.io/upload_images/4843132-3ec0494c6b87c52f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/330/format/webp)



LinkedHashMap提供了多个构造方法，我们先看空参的构造方法。

```java
    public LinkedHashMap() {
        // 调用HashMap的构造方法，其实就是初始化Entry[] table
        super();
        // 这里是指是否基于访问排序，默认为false
        accessOrder = false;
    }
```

首先使用super调用了父类HashMap的构造方法，其实就是根据初始容量、负载因子去初始化Entry[] table，详细的看上一篇[HashMap解析](https://www.jianshu.com/p/dde9b12343c1)。

然后把accessOrder设置为false，这就跟存储的顺序有关了，LinkedHashMap存储数据是有序的，而且分为两种：插入顺序和访问顺序。

这里accessOrder设置为false，表示不是访问顺序而是插入顺序存储的，这也是默认值，表示LinkedHashMap中存储的顺序是按照调用put方法插入的顺序进行排序的。LinkedHashMap也提供了可以设置accessOrder的构造方法，我们来看看这种模式下，它的顺序有什么特点？

```java
                // 第三个参数用于指定accessOrder值
        Map<String, String> linkedHashMap = new LinkedHashMap<>(16, 0.75f, true);
        linkedHashMap.put("name1", "josan1");
        linkedHashMap.put("name2", "josan2");
        linkedHashMap.put("name3", "josan3");
        System.out.println("开始时顺序：");
        Set<Entry<String, String>> set = linkedHashMap.entrySet();
        Iterator<Entry<String, String>> iterator = set.iterator();
        while(iterator.hasNext()) {
            Entry entry = iterator.next();
            String key = (String) entry.getKey();
            String value = (String) entry.getValue();
            System.out.println("key:" + key + ",value:" + value);
        }
        System.out.println("通过get方法，导致key为name1对应的Entry到表尾");
        linkedHashMap.get("name1");
        Set<Entry<String, String>> set2 = linkedHashMap.entrySet();
        Iterator<Entry<String, String>> iterator2 = set2.iterator();
        while(iterator2.hasNext()) {
            Entry entry = iterator2.next();
            String key = (String) entry.getKey();
            String value = (String) entry.getValue();
            System.out.println("key:" + key + ",value:" + value);
        }
```

![](https://upload-images.jianshu.io/upload_images/4843132-442ce33c7124092c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/342/format/webp)

因为调用了get("name1")导致了name1对应的Entry移动到了最后，这里只要知道LinkedHashMap有插入顺序和访问顺序两种就可以，后面会详细讲原理。

还记得，上一篇HashMap解析中提到，在HashMap的构造函数中，调用了init方法，而在HashMap中init方法是空实现，但LinkedHashMap重写了该方法，所以在LinkedHashMap的构造方法里，调用了自身的init方法，init的重写实现如下：

```java
   /**
     * Called by superclass constructors and pseudoconstructors (clone,
     * readObject) before any entries are inserted into the map.  Initializes
     * the chain.
     */
    @Override
    void init() {
        // 创建了一个hash=-1，key、value、next都为null的Entry
        header = new Entry<>(-1, null, null, null);
        // 让创建的Entry的before和after都指向自身，注意after不是之前提到的next
        // 其实就是创建了一个只有头部节点的双向链表
        header.before = header.after = header;
    }
```

这好像跟我们上一篇HashMap提到的Entry有些不一样，HashMap中静态内部类Entry是这样定义的：

```java
 static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
```

没有before和after属性啊！原来，LinkedHashMap有自己的静态内部类Entry，它继承了HashMap.Entry，定义如下:

```java
    /**
     * LinkedHashMap entry.
     */
    private static class Entry<K,V> extends HashMap.Entry<K,V> {
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;

        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }
```

所以LinkedHashMap构造函数，主要就是调用HashMap构造函数初始化了一个Entry[] table，然后调用自身的init初始化了一个只有头结点的双向链表。完成了如下操作：

![](https://upload-images.jianshu.io/upload_images/4843132-cac0b65e4d23b6bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/524/format/webp)



## 2.5 put方法

LinkedHashMap没有重写put方法，所以还是调用HashMap得到put方法，如下：

```java
   public V put(K key, V value) {
        // 对key为null的处理
        if (key == null)
            return putForNullKey(value);
        // 计算hash
        int hash = hash(key);
        // 得到在table中的index
        int i = indexFor(hash, table.length);
        // 遍历table[index]，是否key已经存在，存在则替换，并返回旧值
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        
        modCount++;
        // 如果key之前在table中不存在，则调用addEntry，LinkedHashMap重写了该方法
        addEntry(hash, key, value, i);
        return null;
    }
```

我们看看LinkedHashMap的addEntry方法：

```java
  void addEntry(int hash, K key, V value, int bucketIndex) {
        // 调用父类的addEntry，增加一个Entry到HashMap中
        super.addEntry(hash, key, value, bucketIndex);

        // removeEldestEntry方法默认返回false，不用考虑
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        }
    }
```

这里调用了父类HashMap的addEntry方法，如下：

```java
   void addEntry(int hash, K key, V value, int bucketIndex) {
        // 扩容相关
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
        // LinkedHashMap进行了重写
        createEntry(hash, key, value, bucketIndex);
    }
```

前面是扩容相关的代码，在上一篇HashMap解析中已经讲过了。这里主要看createEntry方法，LinkedHashMap进行了重写。

```java
   void createEntry(int hash, K key, V value, int bucketIndex) {
       HashMap.Entry<K,V> old = table[bucketIndex];
       // e就是新创建了Entry，会加入到table[bucketIndex]的表头
       Entry<K,V> e = new Entry<>(hash, key, value, old);
       table[bucketIndex] = e;
       // 把新创建的Entry，加入到双向链表中
       e.addBefore(header);
       size++;
   }
```

我们来看看LinkedHashMap.Entry的addBefore方法：

```java
     private void addBefore(Entry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }
```

从这里就可以看出，当put元素时，不但要把它加入到HashMap中去，还要加入到双向链表中，所以可以看出LinkedHashMap就是HashMap+双向链表，下面用图来表示逐步往LinkedHashMap中添加数据的过程，红色部分是双向链表，黑色部分是HashMap结构，header是一个Entry类型的双向链表表头，本身不存储数据。

首先是只加入一个元素Entry1，假设index为0：

![](https://upload-images.jianshu.io/upload_images/4843132-d48bb58775418c95.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/475/format/webp)



当再加入一个元素Entry2，假设index为15：

![](https://upload-images.jianshu.io/upload_images/4843132-10c917166c1f4745.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/525/format/webp)



当再加入一个元素Entry3, 假设index也是0：

![](https://upload-images.jianshu.io/upload_images/4843132-32fb46d33b0ed3c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/555/format/webp)



以上，就是LinkedHashMap的put的所有过程了，总体来看，跟HashMap的put类似，只不过多了把新增的Entry加入到双向列表中。



### 2.6 扩容

在HashMap的put方法中，如果发现前元素个数超过了扩容阀值时，会调用resize方法，如下：

```java
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        boolean oldAltHashing = useAltHashing;
        useAltHashing |= sun.misc.VM.isBooted() &&
                (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        boolean rehash = oldAltHashing ^ useAltHashing;
       // 把旧table的数据迁移到新table
        transfer(newTable, rehash);
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
```

LinkedHashMap重写了transfer方法，数据的迁移，它的实现如下：

```java
    void transfer(HashMap.Entry[] newTable, boolean rehash) {
        // 扩容后的容量是之前的2倍
        int newCapacity = newTable.length;
        // 遍历双向链表，把所有双向链表中的Entry，重新就算hash，并加入到新的table中
        for (Entry<K,V> e = header.after; e != header; e = e.after) {
            if (rehash)
                e.hash = (e.key == null) ? 0 : hash(e.key);
            int index = indexFor(e.hash, newCapacity);
            e.next = newTable[index];
            newTable[index] = e;
        }
    }
```

可以看出，LinkedHashMap扩容时，数据的再散列和HashMap是不一样的。

HashMap是先遍历旧table，再遍历旧table中每个元素的单向链表，取得Entry以后，重新计算hash值，然后存放到新table的对应位置。

LinkedHashMap是遍历的双向链表，取得每一个Entry，然后重新计算hash值，然后存放到新table的对应位置。

从遍历的效率来说，遍历双向链表的效率要高于遍历table，因为遍历双向链表是N次（N为元素个数）；而遍历table是N+table的空余个数（N为元素个数）。



## 2.7 双向链表的重排序

前面分析的，主要是当前LinkedHashMap中不存在当前key时，新增Entry的情况。当key如果已经存在时，则进行更新Entry的value。就是HashMap的put方法中的如下代码：

```java
       for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                // 重排序
                e.recordAccess(this);
                return oldValue;
            }
        }
```

主要看e.recordAccess(this)，这个方法跟访问顺序有关，而HashMap是无序的，所以在HashMap.Entry的recordAccess方法是空实现，但是LinkedHashMap是有序的,LinkedHashMap.Entry对recordAccess方法进行了重写。

```java
    void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            // 如果LinkedHashMap的accessOrder为true，则进行重排序
            // 比如前面提到LruCache中使用到的LinkedHashMap的accessOrder属性就为true
            if (lm.accessOrder) {
                lm.modCount++;
                // 把更新的Entry从双向链表中移除
                remove();
                // 再把更新的Entry加入到双向链表的表尾
                addBefore(lm.header);
            }
        }
```

在LinkedHashMap中，只有accessOrder为true，即是访问顺序模式，才会put时对更新的Entry进行重新排序，而如果是插入顺序模式时，不会重新排序，这里的排序跟在HashMap中存储没有关系，只是指在双向链表中的顺序。

举个栗子：开始时，HashMap中有Entry1、Entry2、Entry3，并设置LinkedHashMap为访问顺序，则更新Entry1时，会先把Entry1从双向链表中删除，然后再把Entry1加入到双向链表的表尾，而Entry1在HashMap结构中的存储位置没有变化，对比图如下所示：

![](https://upload-images.jianshu.io/upload_images/4843132-ba6fc470c045ffa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/757/format/webp)





## 2.8 get方法

LinkedHashMap有对get方法进行了重写，如下：

```java
 public V get(Object key) {
        // 调用genEntry得到Entry
        Entry<K,V> e = (Entry<K,V>)getEntry(key);
        if (e == null)
            return null;
        // 如果LinkedHashMap是访问顺序的，则get时，也需要重新排序
        e.recordAccess(this);
        return e.value;
    }
```

先是调用了getEntry方法，通过key得到Entry，而LinkedHashMap并没有重写getEntry方法，所以调用的是HashMap的getEntry方法，在上一篇文章中我们分析过HashMap的getEntry方法：首先通过key算出hash值，然后根据hash值算出在table中存储的index，然后遍历table[index]的单向链表去对比key，如果找到了就返回Entry。

后面调用了LinkedHashMap.Entry的recordAccess方法，上面分析过put过程中这个方法，其实就是在访问顺序的LinkedHashMap进行了get操作以后，重新排序，把get的Entry移动到双向链表的表尾。



### 2.9 遍历方式取数据

我们先来看看HashMap使用遍历方式取数据的过程：

![](https://upload-images.jianshu.io/upload_images/4843132-51c76a3e45566c32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/552/format/webp)



很明显，这样取出来的Entry顺序肯定跟插入顺序不同了，既然LinkedHashMap是有序的，那么它是怎么实现的呢？  
先看看LinkedHashMap取遍历方式获取数据的代码：

```java
        Map<String, String> linkedHashMap = new LinkedHashMap<>();
        linkedHashMap.put("name1", "josan1");
        linkedHashMap.put("name2", "josan2");
        linkedHashMap.put("name3", "josan3");
                // LinkedHashMap没有重写该方法，调用的HashMap中的entrySet方法
        Set<Entry<String, String>> set = linkedHashMap.entrySet();
        Iterator<Entry<String, String>> iterator = set.iterator();
        while(iterator.hasNext()) {
            Entry entry = iterator.next();
            String key = (String) entry.getKey();
            String value = (String) entry.getValue();
            System.out.println("key:" + key + ",value:" + value);
        }
```

LinkedHashMap没有重写entrySet方法，我们先来看HashMap中的entrySet，如下：

```java
public Set<Map.Entry<K,V>> entrySet() {
        return entrySet0();
    }

    private Set<Map.Entry<K,V>> entrySet0() {
        Set<Map.Entry<K,V>> es = entrySet;
        return es != null ? es : (entrySet = new EntrySet());
    }

    private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public Iterator<Map.Entry<K,V>> iterator() {
            return newEntryIterator();
        }
        // 无关代码
        ......
    }
```

可以看到，HashMap的entrySet方法，其实就是返回了一个EntrySet对象。

我们得到EntrySet会调用它的iterator方法去得到迭代器Iterator，从上面的代码也可以看到，iterator方法中直接调用了newEntryIterator方法并返回，而LinkedHashMap重写了该方法

```java
    Iterator<Map.Entry<K,V>> newEntryIterator() { 
        return new EntryIterator();
    }
```

这里直接返回了EntryIterator对象，这个和上一篇HashMap中的newEntryIterator方法中一模一样，都是返回了EntryIterator对象，其实他们返回的是各自的内部类。我们来看看LinkedHashMap中EntryIterator的定义：

```java
    private class EntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() { 
          return nextEntry();
        }
    }
```

该类是继承LinkedHashIterator，并重写了next方法；而HashMap中是继承HashIterator。  
我们再来看看LinkedHashIterator的定义：

```java
   private abstract class LinkedHashIterator<T> implements Iterator<T> {
        // 默认下一个返回的Entry为双向链表表头的下一个元素
        Entry<K,V> nextEntry    = header.after;
        Entry<K,V> lastReturned = null;

        public boolean hasNext() {
            return nextEntry != header;
        }

        Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (nextEntry == header)
                throw new NoSuchElementException();

            Entry<K,V> e = lastReturned = nextEntry;
            nextEntry = e.after;
            return e;
        }
        // 不相关代码
        ......
    }
```

我们先不看整个类的实现，只要知道在LinkedHashMap中，Iterator<Entry<String, String>> iterator = set.iterator()，这段代码会返回一个继承LinkedHashIterator的Iterator，它有着跟HashIterator不一样的遍历规则。

接着，我们会用while(iterator.hasNext())去循环判断是否有下一个元素，LinkedHashMap中的EntryIterator没有重写该方法，所以还是调用LinkedHashIterator中的hasNext方法，如下：

```java
        public boolean hasNext() {
            // 下一个应该返回的Entry是否就是双向链表的头结点
            // 有两种情况：1.LinkedHashMap中没有元素；2.遍历完双向链表回到头部
            return nextEntry != header;
        }
```

nextEntry表示下一个应该返回的Entry，默认值是header.after，即双向链表表头的下一个元素。而上面介绍到，LinkedHashMap在初始化时，会调用init方法去初始化一个before和after都指向自身的Entry，但是put过程会把新增加的Entry加入到双向链表的表尾，所以只要LinkedHashMap中有元素，第一次调用hasNext肯定不会为false。

然后我们会调用next方法去取出Entry，LinkedHashMap中的EntryIterator重写了该方法，如下：

```java
 public Map.Entry<K,V> next() { 
    return nextEntry(); 
}
```

而它自身又没有重写nextEntry方法，所以还是调用的LinkedHashIterator中的nextEntry方法：

```java
        Entry<K,V> nextEntry() {
            // 保存应该返回的Entry
            Entry<K,V> e = lastReturned = nextEntry;
            //把当前应该返回的Entry的after作为下一个应该返回的Entry
            nextEntry = e.after;
            // 返回当前应该返回的Entry
            return e;
        }
```

这里其实遍历的是双向链表，所以不会存在HashMap中需要寻找下一条单向链表的情况，从头结点Entry header的下一个节点开始，只要把当前返回的Entry的after作为下一个应该返回的节点即可。直到到达双向链表的尾部时，after为双向链表的表头节点Entry header，这时候hasNext就会返回false，表示没有下一个元素了。LinkedHashMap的遍历取值如下图所示：

![](https://upload-images.jianshu.io/upload_images/4843132-87fc7def1002acd4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/548/format/webp)



易知，遍历出来的结果为Entry1、Entry2...Entry6。  
可得，LinkedHashMap是有序的，且是通过双向链表来保证顺序的。



### 2.10 remove方法

LinkedHashMap没有提供remove方法，所以调用的是HashMap的remove方法，实现如下：

```java
    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }

    final Entry<K,V> removeEntryForKey(Object key) {
        int hash = (key == null) ? 0 : hash(key);
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e)
                    table[i] = next;
                else
                    prev.next = next;
                // LinkedHashMap.Entry重写了该方法
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }
```

在上一篇HashMap中就分析了remove过程，其实就是断开其他对象对自己的引用。比如被删除Entry是在单向链表的表头，则让它的next放到表头，这样它就没有被引用了；如果不是在表头，它是被别的Entry的next引用着，这时候就让上一个Entry的next指向它自己的next，这样，它也就没被引用了。

在HashMap.Entry中recordRemoval方法是空实现，但是LinkedHashMap.Entry对其进行了重写，如下：

```java
        void recordRemoval(HashMap<K,V> m) {
            remove();
        }

        private void remove() {
            before.after = after;
            after.before = before;
        }
```



易知，这是要把双向链表中的Entry删除，也就是要断开当前要被删除的Entry被其他对象通过after和before的方式引用。

所以，LinkedHashMap的remove操作。首先把它从table中删除，即断开table或者其他对象通过next对其引用，然后也要把它从双向链表中删除，断开其他对应通过after和before对其引用。

# 3 HashMap与LinkedHashMap的结构对比

再来看看HashMap和LinkedHashMap的结构图，是不是秒懂了。LinkedHashMap其实就是可以看成HashMap的基础上，多了一个双向链表来维持顺序。

![](https://upload-images.jianshu.io/upload_images/4843132-2e04e0f72a751a47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600/format/webp)



![](https://upload-images.jianshu.io/upload_images/4843132-23488d46581b87ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/739/format/webp)



## 4 LinkedHashMap在Android中的应用

在Android中使用图片时，一般会用LruCacha做图片的内存缓存，它里面就是使用LinkedHashMap来实现存储的。`

```java
public class LruCache<K, V> {
    private final LinkedHashMap<K, V> map;
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        // 注意第三个参数，是accessOrder，这里为true，后面会讲到
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }


```

前面提到了，accessOrder为true，表示LinkedHashMap为访问顺序，当对已存在LinkedHashMap中的Entry进行get和put操作时，会把Entry移动到双向链表的表尾（其实是先删除，再插入）。  
我们拿LruCache的put方法举例：

```java
  public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        // 对map进行操作之前，先进行同步操作
        synchronized (this) {
            putCount++;
            size += safeSizeOf(key, value);
            previous = map.put(key, value);
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }
        // 整理内存，看是否需要移除LinkedHashMap中的元素
        trimToSize(maxSize);
        return previous;
    }
```

之前提到了，HashMap是线程不安全的，LinkedHashMap同样是线程不安全的。所以在对调用LinkedHashMap的put方法时，先使用synchronized 进行了同步操作。

我们最关心的是倒数第一行代码，其中maxSize为我们给LruCache设置的最大缓存大小。我们看看该方法：

```java
    /**
     * Remove the eldest entries until the total of remaining entries is at or
     * below the requested size.
     *
     * @param maxSize the maximum size of the cache before returning. May be -1
     *            to evict even 0-sized elements.
     */
    public void trimToSize(int maxSize) {
        // while死循环，直到满足当前缓存大小小于或等于最大可缓存大小
        while (true) {
            K key;
            V value;
            // 线程不安全，需要同步
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }
                // 如果当前缓存的大小，已经小于等于最大可缓存大小，则直接返回
                // 不需要再移除LinkedHashMap中的数据
                if (size <= maxSize || map.isEmpty()) {
                    break;
                }
                // 得到的就是双向链表表头header的下一个Entry
                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                // 移除当前取出的Entry
                map.remove(key);
                // 从新计算当前的缓存大小
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }

```

从注释上就可以看出，该方法就是不断移除LinkedHashMap中双向链表表头的元素，直到当前缓存大小小于或等于最大可缓存的大小。

由前面的重排序我们知道，对LinkedHashMap的put和get操作，都会让被操作的Entry移动到双向链表的表尾，而移除是从map.entrySet().iterator().next()开始的，也就是双向链表的表头的header的after开始的，这也就符合了LRU算法的需求。

下图表示了LinkedHashMap中删除、添加、get/put已存在的Entry操作。  
红色表示初始状态  
紫色表示缓存图片大小超过了最大可缓存大小时，才能够表头移除Entry1  
蓝色表示对已存在的Entry3进行了get/put操作，把它移动到双向链表表尾  
绿色表示新增一个Entry7，插入到双向链表的表尾（暂时不考虑在HashMap中的位置）

![](https://upload-images.jianshu.io/upload_images/4843132-8a943e9216bccf27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/541/format/webp)





# 5 总结

1. LinkedHashMap是继承于HashMap，是基于HashMap和双向链表来实现的。
2. HashMap无序；LinkedHashMap有序，可分为插入顺序和访问顺序两种。如果是访问顺序，那put和get操作已存在的Entry时，都会把Entry移动到双向链表的表尾(其实是先删除再插入)。
3. LinkedHashMap存取数据，还是跟HashMap一样使用的Entry[]的方式，双向链表只是为了保证顺序。
4. LinkedHashMap是线程不安全的。






