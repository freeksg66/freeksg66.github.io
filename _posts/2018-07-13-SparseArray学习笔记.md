---
layout:     post
title:      SparseArray学习笔记
subtitle:
date:       2018-07-13
author:     BY
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Android
    - Java
---
项目中在使用HashMap<Integer, Object>时提示可以用SparseArray<Object>代替，因此学习一下Android中专有的SparseArray和ArrayMap与Java中HashMap的区别。

## 目录

Java提供了HashMap，但是HashMap对于手机端而言，对内存的占用太大，所以Android提供了SparseArray和ArrayMap。二者都是基于二分查找，所以数据量大的时候，最坏效率会比HashMap慢很多。因此建议数据量在千以内比较合适。

## SparseArray学习

<b>SparseArray整体认识</b>

![](http://pbmurxnd0.bkt.clouddn.com/SparseArray%E5%88%86%E6%9E%90.jpg)

SparseArray内部维护了两个数组，一个是记录key的数组，另一个是记录value的数组。其中记录key的数组要保持有序，便于二分查找找到key所在的数组的索引index，该索引index在value数组中对应的元素即为key所对应的value。

SparseArray采用二分查找key(查找效率O(n))，虽然在时间效率上不及HashMap的查找效率O(1)，但在<key, value>数量较小（数量在千级别）时，二者时间效率差别不是很大，SparseArray的内存效率比HashMap的散列结构要高。对于Android程序宝贵的内存量，牺牲较少的时间换取内存效率，还是非常合适的。

### SparseArray构建
{% highlight ruby %}
public class SparseArray<E> implements Cloneable {
    private static final Object DELETED = new Object(); // 不立即删除元素，先标记为Delete，再在gc调用时删除元素对象
    private boolean mGarbage = false;

    private int[] mKeys; // 保存key的数组
    private Object[] mValues; // 保存value的数组
    private int mSize; // 当前存储<key, value>的数量

    // SparseArray的默认构造函数，默认大小是10
    public SparseArray() {
        this(10);
    }

    // SparseArray真正构造函数
    public SparseArray(int initialCapacity) {
        if (initialCapacity == 0) {
            // 稀疏阵列将用轻量级表示初始化，而不需要任何额外的Array分配。
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
            // mValues和mKeys数组的长度一致
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        mSize = 0; // 当前存储0个元素
    }
}
{% endhighlight %}

### SparseArray插入

{% highlight ruby %}
public void put(int key, E value) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key); // 二分查找该key在mKeys中的索引，如果找到了则返回一个大于等于0的下标，如果未找到则将小于该key的前一个下标位取反，此时i是小于0的数字

    if (i >= 0) { // 如果找到了该元素
        mValues[i] = value;
    } else {
        i = ~i; // 取到小于该key的前一个下标

        if (i < mSize && mValues[i] == DELETED) { // 如果是一个已经被删除的元素，则直接替换该键值对
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }

        if (mGarbage && mSize >= mKeys.length) { // 是否进行垃圾回收或者需要扩容
            gc();

            // Search again because indices may have changed.
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }

        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key); // 在i后插入key
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value); // 在i后插入value
        mSize++; // 存储的<key, value>量+1
    }
}
{% endhighlight %}

### SparseArray查找

{% highlight ruby %}
public E get(int key) {
    return get(key, null);
}

public E get(int key, E valueIfKeyNotFound) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key); // 二分查找，找到key对应的下表

    if (i < 0 || mValues[i] == DELETED) { // 如果没找到key，或者mValues[i]已经处于被删除状态，自返回“未找到”
        return valueIfKeyNotFound;
    } else { // 否则返回该key对应的value值
        return (E) mValues[i];
    }
}
{% endhighlight %}

## ArrayMap学习

首先从ArrayMap的四个数组说起。

* mHashes，用于保存key对应的hashCode；
* mArray，用于保存键值对（key，value），其结构为[key1,value1,key2,value2,key3,value3,......]；
* mBaseCache，缓存，如果ArrayMap的数据量从4，增加到8，用该数组保存之前使用的mHashes和mArray，这样如果数据量再变回4的时候，可以再次使用之前的数组，不需要再次申请空间，这样节省了一定的时间；
* mTwiceBaseCache，与mBaseCache对应，不过触发的条件是数据量从8增长到12。

上面提到的数据量有8增长到12，为什么不是16？这也算是ArrayMap的一个优化的点，它不是每次增长1倍，而是使用了如下方法（mSize+(mSize>>1)），即每次增加1/2。

### indexOf方法

这里使用了二分查找来查找对应的index

{% highlight ruby %}
int indexOf(Object key, int hash) {
    final int N = mSize;

    // Important fast case: if nothing is in here, nothing to look for.
    //数组为空，直接返回
    if (N == 0) {
        return ~0;
    }

    //二分查找，不细说了
    int index = ContainerHelpers.binarySearch(mHashes, N, hash);

    // If the hash code wasn't found, then we have no entry for this key.
    //没找到hashCode，返回index，一个负数
    if (index < 0) {
        return index;
    }

    // If the key at the returned index matches, that's what we want.
    //对比key值，相同则返回index
    if (key.equals(mArray[index<<1])) {
        return index;
    }

    // Search for a matching key after the index.
    //如果返回的index对应的key值，与传入的key值不等，则可能对应的key在index后面
    int end;
    for (end = index + 1; end < N && mHashes[end] == hash; end++) {
        if (key.equals(mArray[end << 1])) return end;
    }

    // Search for a matching key before the index.
    //接上句，后面没有，那一定在前面。
    for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
        if (key.equals(mArray[i << 1])) return i;
    }

    // Key not found -- return negative value indicating where a
    // new entry for this key should go.  We use the end of the
    // hash chain to reduce the number of array entries that will
    // need to be copied when inserting.
    //毛都没找到，那肯定是没有了，返回个负数
    return ~end;
}
{% endhighlight %}


### put方法

{% highlight ruby %}
public V put(K key, V value) {
    final int hash;
    int index;
    //key是空，则通过indexOfNull查找对应的index；如果不为空，通过indexOf查找对应的index
    if (key == null) {
        hash = 0;
        index = indexOfNull();
    } else {
        hash = key.hashCode();
        index = indexOf(key, hash);
    }
        
    //index大于或等于0，一定是之前put过相同的key，直接替换对应的value。因为mArray中不只保存了value，还保存了key。
    //其结构为[key1,value1,key2,value2,key3,value3,......]
    //所以，需要将index乘2对应key，index乘2再加1对应value
    if (index >= 0) {
        index = (index<<1) + 1;
        final V old = (V)mArray[index];
        mArray[index] = value;
        return old;
    }

    //取正数
    index = ~index;
    //mSize的大小，即已经保存的数据量与mHashes的长度相同了，需要扩容啦
    if (mSize >= mHashes.length) {
        //扩容后的大小，有以下几个档位，BASE_SIZE（4），BASE_SIZE的2倍（8），mSize+(mSize>>1)（比之前的数据量扩容1/2）
        final int n = mSize >= (BASE_SIZE*2) ? (mSize+(mSize>>1))
                : (mSize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);

        if (DEBUG) Log.d(TAG, "put: grow from " + mHashes.length + " to " + n);

        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        //扩容方法的实现
        allocArrays(n);

        //扩容后，需要把原来的数据拷贝到新数组中
        if (mHashes.length > 0) {
            if (DEBUG) Log.d(TAG, "put: copy 0-" + mSize + " to 0");
            System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
            System.arraycopy(oarray, 0, mArray, 0, oarray.length);
        }

        //看看被废弃的数组是否还有利用价值
        //如果被废弃的数组的数据量为4或8，说明可能利用价值，以后用到的时候可以直接用。
        //如果被废弃的数据量太大，扔了算了，要不太占内存。如果浪费内存了，还费这么大劲，加了类干啥。
        freeArrays(ohashes, oarray, mSize);
    }

    //这次put的key对应的hashcode排序没有排在最后（index没有指示到数组结尾），因此需要移动index后面的数据
    if (index < mSize) {
        if (DEBUG) Log.d(TAG, "put: move " + index + "-" + (mSize-index)
                + " to " + (index+1));
        System.arraycopy(mHashes, index, mHashes, index + 1, mSize - index);
        System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
    }

    //把数据保存到数组中。看到了吧，key和value都在mArray中；hashCode放到mHashes
    mHashes[index] = hash;
    mArray[index<<1] = key;
    mArray[(index<<1)+1] = value;
    mSize++;
    return null;
}
{% endhighlight %}

### remove方法

remove方法在某种条件下，会重新分配内存，保证分配给ArrayMap的内存在合理区间，减少对内存的占用。但是如果每次remove都重新分配空间，会浪费大量的时间。因此在此处，Android使用的是用空间换时间的方式，以避免效率低下。无论从任何角度，频繁的分配回收内存一定会耗费时间的。

　　remove最终使用的是removeAt方法，此处只说明removeAt

{% highlight ruby %}
/**
 * Remove the key/value mapping at the given index.
 * @param index The desired index, must be between 0 and {@link #size()}-1.
 * @return Returns the value that was stored at this index.
 */
public V removeAt(int index) {
    final Object old = mArray[(index << 1) + 1];
    //如果数据量小于等于1，说明删除该元素后，没有数组为空，清空两个数组。
    if (mSize <= 1) {
        // Now empty.
        if (DEBUG) Log.d(TAG, "remove: shrink from " + mHashes.length + " to 0");
        //put中已有说明
        freeArrays(mHashes, mArray, mSize);
        mHashes = EmptyArray.INT;
        mArray = EmptyArray.OBJECT;
        mSize = 0;
    } else {
        //如果当初申请的数组最大容纳数据个数大于BASE_SIZE的2倍（8），并且现在存储的数据量只用了申请数量的1/3，
        //则需要重新分配空间，已减少对内存的占用
        if (mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3) {
            // Shrunk enough to reduce size of arrays.  We don't allow it to
            // shrink smaller than (BASE_SIZE*2) to avoid flapping between
            // that and BASE_SIZE.
            //新数组的大小
            final int n = mSize > (BASE_SIZE*2) ? (mSize + (mSize>>1)) : (BASE_SIZE*2);

            if (DEBUG) Log.d(TAG, "remove: shrink from " + mHashes.length + " to " + n);

            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;
            allocArrays(n);

            mSize--;
            //index之前的数据拷贝到新数组中
            if (index > 0) {
                if (DEBUG) Log.d(TAG, "remove: copy from 0-" + index + " to 0");
                System.arraycopy(ohashes, 0, mHashes, 0, index);
                System.arraycopy(oarray, 0, mArray, 0, index << 1);
            }
            //将index之后的数据拷贝到新数组中，和（index>0）的分支结合，就将index位置的数据删除了
            if (index < mSize) {
                if (DEBUG) Log.d(TAG, "remove: copy from " + (index+1) + "-" + mSize
                        + " to " + index);
                System.arraycopy(ohashes, index + 1, mHashes, index, mSize - index);
                System.arraycopy(oarray, (index + 1) << 1, mArray, index << 1,
                        (mSize - index) << 1);
            }
        } else {
            mSize--;
            //将index后的数据向前移位
            if (index < mSize) {
                if (DEBUG) Log.d(TAG, "remove: move " + (index+1) + "-" + mSize
                        + " to " + index);
                System.arraycopy(mHashes, index + 1, mHashes, index, mSize - index);
                System.arraycopy(mArray, (index + 1) << 1, mArray, index << 1,
                        (mSize - index) << 1);
            }
            //移位后最后一个数据清空
            mArray[mSize << 1] = null;
            mArray[(mSize << 1) + 1] = null;
        }
    }
    return (V)old;
}
{% endhighlight %}

### freeArrays方法

put中有说明，这里就不进行概述了，直接上代码，印证上面的说法。

{% highlight ruby %}
private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
    //已经废弃的数组个数为BASE_SIZE的2倍（8），则用mTwiceBaseCache保存废弃的数组；
    //如果个数为BASE_SIZE（4），则用mBaseCache保存废弃的数组
    if (hashes.length == (BASE_SIZE*2)) {
        synchronized (ArrayMap.class) {
            if (mTwiceBaseCacheSize < CACHE_SIZE) {
                //array为刚刚废弃的数组，mTwiceBaseCache如果有内容，则放入array[0]位置，
                //在allocArrays中会从array[0]取出，放回mTwiceBaseCache
                array[0] = mTwiceBaseCache;
                //array[1]存放hash数组。因为array中每个元素都是Object对象，所以每个元素都可以存放数组
                array[1] = hashes;
                //清除index为2和之后的数据
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                mTwiceBaseCache = array;
                mTwiceBaseCacheSize++;
                if (DEBUG) Log.d(TAG, "Storing 2x cache " + array
                        + " now have " + mTwiceBaseCacheSize + " entries");
            }
        }
    } else if (hashes.length == BASE_SIZE) {
        synchronized (ArrayMap.class) {
            if (mBaseCacheSize < CACHE_SIZE) {
                //代码的注释可以参考上面，不重复说明了
                array[0] = mBaseCache;
                array[1] = hashes;
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                mBaseCache = array;
                mBaseCacheSize++;
                if (DEBUG) Log.d(TAG, "Storing 1x cache " + array
                        + " now have " + mBaseCacheSize + " entries");
            }
        }
    }
}
{% endhighlight %}

### allocArrays方法

总体来说，通过新数组的个数产生3个分支，个数为BASE_SIZE（4），从mBaseCache取之前废弃的数组；BASE_SIZE的2倍（8），从mTwiceBaseCache取之前废弃的数组；其他，之前废弃的数组没有存储，因为太耗费内存，这种情况下，重新分配内存。

### clear和erase方法

clear清空数组，如果再向数组中添加元素，需要重新申请空间；erase清除数组中的数组，空间还在。

### get方法

主要的逻辑都在indexOf中了。。。




