---
layout: page
title: HashMaps, ArrayMaps and SparseArrays in Android
categories: Android,java
---

[原文链接](http://deepakmishra.me/blog/index.php/2015/10/19/hashmaps-arraymaps-and-sparsearrays-in-android/)

Android开发者都知道Lint在我们使用HashMap的时候会给出警告——使用SparseArray会优化内存。这可是一件好事情。那现在我们有几个类要学习去使用。比如:ArrayMap和SimpleArrayMap，当然还有各种类型的SparseArray。这篇文章将讲解这些类及它们的原理。
先从如何使用它们开始吧。

```java
java.util.HashMap<String, String> hashMap = new java.util.HashMap<String, String>(16);
hashMap.put("key", "value");
hashMap.get("key");
hashMap.entrySet().iterator();

android.util.ArrayMap<String, String> arrayMap = new android.util.ArrayMap<String, String>(16);
arrayMap.put("key", "value");
arrayMap.get("key");
arrayMap.entrySet().iterator();

android.support.v4.util.ArrayMap<String, String> supportArrayMap =
        new android.support.v4.util.ArrayMap<String, String>(16);
supportArrayMap.put("key", "value");
supportArrayMap.get("key");
supportArrayMap.entrySet().iterator();

android.support.v4.util.SimpleArrayMap<String, String> simpleArrayMap =
        new android.support.v4.util.SimpleArrayMap<String, String>(16);
simpleArrayMap.put("key", "value");
simpleArrayMap.get("key");
//simpleArrayMap.entrySet().iterator();      <- will not compile

android.util.SparseArray<String> sparseArray = new android.util.SparseArray<String>(16);
sparseArray.put(10, "value");
sparseArray.get(10);

android.util.LongSparseArray<String> longSparseArray = new android.util.LongSparseArray<String>(16);
longSparseArray.put(10L, "value");
longSparseArray.get(10L);

android.util.SparseLongArray sparseLongArray = new android.util.SparseLongArray(16);
sparseLongArray.put(10, 100L);
sparseLongArray.get(10);

```
接下我们一个一个的讨论。java中的集合基本都是基于数组。在我们了解这些替代类之前我们需要理解HashMap是怎么样工作的。

java.util.HashMap
HashMap本质上是一个HashMapEntry构成的数组。每个Entry的实体都包括:

一个非基本类型的key
一个非基本类型的value
一个key的Hashcode
指向下一个Entry的指针
如下代码（因为是泛型所以不能是基本类型）

```java
  final K key;
  V value;
  HashMapEntry<K,V> next;
  int hash;
```
需要注意的是key和value都不是基本类型的。这是Java工程师做出的设计决策。所以我们不得不容忍它。当插入一个基本类型的时候会产生自动装箱的消耗。
当HashMap插入一个object时：

key的Hashcode会被计算出来并赋值到Entry类的变量中。
java.util.HashMap.indexFor()这个方法依赖于hashcode。这个方法你也可以看成是利用Entry[]的size的取模函数。
```java
 static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }
```
并用这个方法来决定当前的Entry放在Entry[]的哪个index,这个index叫做"bucket（桶）"

如果这个桶已经存在了元素，那么新的元素会被插入到上一个元素中指针指定的位置——这个结构和LinkedList基本一样。
它查询的复杂度是O(1)：
已经计算好了插入的Key的hashcode。
java.util.HashMap.indexFor() 这个方法是基于hashcode的，所以我们获取Entry的bucket（位置）就像查询一个数组。
O(1)的时间复杂度是所有的开发都乐意看到的，但是内存消耗也是应该考虑的因素。特别是在移动设备上。
HashMap的缺点：

自动装箱意味着需要产生额外的对象，这对于内存的使用和垃圾回收产生影响。
HashMapEntity自己本身也会产生额外的对象，这同样会影响内存的使用和垃圾回收产生。
每次HashMap的存储对象减少或都增加的时候，这个开销会随着Hashmap的size增加而增加。
哈希是很好的实现方案，但是如果实现的不好将会让我们的开销回到O(N)
很多人都会忽略关于哈希的另外一个缺点：我们需要存储它的key和对应的hash值。这种冗余有助于解决冲突。 非散列解决方案也可以在这方面有所帮助。
android.util.ArrayMap
ArrayMap 用了两个数组。在它内部用了Object[] mArray来存储Object,还用了int[] mHashes 来存储hashcode,当存储一对键值对的时候。

Key/Value会被自动装箱。
key会存储在mArray[]的下一个可用的位置。
而value会存储在mArray[]中key的下一个位置。（key和value在mArray中交叉存储）
key的哈希值会被计算出来并存储在mHashed[]中。
当查找一个key的时候：
计算key的hashcode。
在mHashes[]中对这个hashcode进行二分法查找。也就意味着时间复杂度增加到了O(logN)
一旦我们得到了这个哈希值的位置index。我们就知道这个key是在mArray的2index的位置，而value则在2index+1的位置。
这个ArrayMap还是没能解决自动装箱的问题。当put一对键值对进入的时候，它们只接受Object，但是我们相对于HashMap来说每一次put会少创建一个对象(HashMapEntry)。这是不是值得我们用O(1)的查找复杂度来换呢？对于大多数app应用来说是值得的。
android.support.v4.util.ArrayMap
android.util.ArrayMap只能在api不小于19（Kitkat）的平台才能使用。而Support library则支持在旧平台上提供相同的功能。

android.support.v4.util.SimpleArrayMap
在之前发布的代码片段中你可以看到，这个类没有entrySet()这个支持迭代的方法。如果你查看它的文档，你会发现很java标准集合的方法它都没有。那我们为什么要用它呢。让它失去与其它java容器的相互操作的特性来减小apk的大小。这样的话，
Proguard(代码优化和混沌工具：可能是你代码构建生成的一部分)可以帮你减少大多数没有使用的Collections API代码从而减小你的apk大小。它的内部工作和android.util.ArrayMap是一样的。

android.util.SparseArray
和ArrayMap一样，它里面也用了两个数组。一个int[] mKeys和Object[] mValues。从名字都可以看得出来一个用来存储key一个用来保存value的。
当保存一对键值对的时候：

key（不是它的hashcode）保存在mKeys[]的下一个可用的位置上。所以不会再对key自动装箱了。
value保存在mValues[]的下一个位置上，value还是要自动装箱的，如果它是基本类型。
查找的时候：
查找key还是用的二分法查找。也就是说它的时间复杂度还是O(logN)
知道了key的index，也就可以用key的index来从mValues中检索出value。
相较于HashMap,我们舍弃了Entry和Object类型的key,放弃了HashCode并依赖于二分法查找。在添加和删除操作的时候有更好的性能开销。
KitKat以前的版本用android.support.v4.util.SparseArrayCompat
android.util.LongSparseArray
SparseArray只接受int类型作为key,而LongSparseArray我们就可以用long作为key。实现原理和SparseArray一致。

android.util.SparseIntArray, android.util.SparseLongArray and android.util.SparseBooleanArray
对于key是int类型而value是int 或者long再或者是boolean,我们可以对应使用SparseIntArray,SparseLongArray ,SparseBooleanArray 。它们使用方式是和SparseArray一样的。它的好处是mValues[]是基本类型的数组。也就意味着无论是key还是value都不用装箱。并且相对于HashMap来说我们节约了3个对象的初始化（Entry,Key和Value），但是我们将查看复杂度从O(1)上升到了O(logN)

结语
使用SparseArray和ArrayMap肯定会减少对象创建的数目。当集合的的数目多达几百个的时候，性能差异也不会很明显（少于50%）。将ArrayMap和SparseArray迁移到新代码中是很有好处的。并且由于方法签名匹配，所以迁移也很容易。

注意：即使它们听起来像数组(Array)，ArrayMap和SparseArray不能保证保留它们的插入顺序，在迭代的时候应该注意。
其中二分法查找方法是 android.util.ContainerHelpers中的方法。

```java
class ContainerHelpers {

    // This is Arrays.binarySearch(), but doesn't do any argument validation.
    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;
            final int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        return ~lo;  // value not present
    }

    static int binarySearch(long[] array, int size, long value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;
            final long midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        return ~lo;  // value not present
    }
}
```
