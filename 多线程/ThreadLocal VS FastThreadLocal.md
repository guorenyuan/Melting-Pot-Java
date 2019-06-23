#### JDK中的ThreadLocal是如何存储绑定在当前线程中的值 ？
Thread类中threadLocals存储了当前线程对象下的所有ThreadLocal对象，threadLocals的类型是ThreadLocal.ThreadLocalMap，即ThreadLocalMap是ThreadLocal的内部类。ThreadLocalMap内部维护了一个Entry对象数组，该数组的初始长度是16，当前线程下的所有ThreadLocal对象最终是存储在此Entry对象数组中。下面是Entry的定义：
```
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```
每个Entry对象是一个key-value的结构，key是ThreadLocal对象，value是与该ThreadLocal对象对应的值。

#### JDK中的ThreadLocal存在哪些问题 ？

* 存取效率低下
ThreadLocal对值进行存储时，需要先根据key.threadLocalHashCode & (len-1)来确定此ThreadLocal对象和其对应的值在Entry对象数组中的位置，如果发生冲突，则采用线性探测法（检查数组的下一个位置，直至没有冲突）来确定其最终的位置。此外，若Entry数组中的对象数量达到阈值后，需要进行rehash操作。
```
private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            // 计算ThreadLocal及其值在数组中的索引
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 // 线性探测
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                // 扩容
                rehash();
        }
```
* 存在内存泄漏问题
Entry对象中的ThreadLocal对象是通过虚引用对象进行保存，如果在下一次GC时，该ThreadLocal对象无外部强引用指向，则会被回收，而该ThreadLocal对象对应的value由于当前Thread还未结束(比如线程池中的线程一般生命周期比较长)，此时产生内存泄漏。

#### Netty中的FastThreadLocal时如何存储绑定在当前线程中的值 ？
FastThreadLocalThread类（在Netty中FastThreadLocal一般与FastThreadLocalThread配套使用)中threadLocalMap存储了当前线程对象下的所有FastThreadLocal对象对应的值，threadLocalMap的类型是InternalThreadLocalMap，InternalThreadLocalMap内部维护了一个Object[]，该数组的初始长度是32，每个FastThreadLocal对象会对应一个当前线程下唯一的索引，该索引也是Object[] 数组的索引，而Object数组中的值即是每个FastThreadLocal对象对应的值。当前线程下的所有ThreadLocal对象最终是存储在此Entry对象数组中。下面是FastThreadLocal类中属性的定义：，

    public class FastThreadLocal<V> {
		private static final int variablesToRemoveIndex = InternalThreadLocalMap.nextVariableIndex();
	    private final int index = InternalThreadLocalMap.nextVariableIndex();
	}
	
#### Netty中的FastThreadLocal如何保证快且安全 ？
* 快
FastThreadLocal对象每次调用set或get方法时，直接根据FastThreadLocal中的index值去InternalThreadLocalMap的Object[]中进行存储或查询，无冲突，且扩容高效（位于算加上数组拷贝）。下面是扩容的代码：

```
private void expandIndexedVariableTableAndSet(int index, Object value){
        Object[] oldArray = this.indexedVariables;
        int oldCapacity = oldArray.length;
        int newCapacity = index | index >>> 1;
        newCapacity |= newCapacity >>> 2;
        newCapacity |= newCapacity >>> 4;
        newCapacity |= newCapacity >>> 8;
        newCapacity |= newCapacity >>> 16;
        ++newCapacity;
        Object[] newArray = Arrays.copyOf(oldArray, newCapacity);
        Arrays.fill(newArray, oldCapacity, newArray.length, UNSET);
        newArray[index] = value;
        this.indexedVariables = newArray;
 }
```

* 安全
FastThreadLocalThread对象会将当前线程创建的所有FastThreadLocal对象存储在InternalThreadLocalMap的Object[]数组的variablesToRemoveIndex索引处以便后续对FastThreadLocal对象进行清理。下面是对象清理的过程：

```
final class FastThreadLocalRunnable implements Runnable {
    private final Runnable runnable;

    private FastThreadLocalRunnable(Runnable runnable) {
        this.runnable = (Runnable)ObjectUtil.checkNotNull(runnable, "runnable");
    }

    public void run() {
        try {
            this.runnable.run();
        } finally {
            FastThreadLocal.removeAll();
        }

    }

    static Runnable wrap(Runnable runnable) {
        return (Runnable)(runnable instanceof FastThreadLocalRunnable ? runnable : new FastThreadLocalRunnable(runnable));
    }
}
```

FastThreadLocalThread执行的任务会被封装成FastThreadLocalRunnable，每次任务执行结束之后会对当前线程下所有的FastThreadLocal对象进行清理同时每个FastThreadLocal对象对应的值也会被清理。

    public static void removeAll() {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.getIfSet();
        if (threadLocalMap != null) {
            try {
                Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
                if (v != null && v != InternalThreadLocalMap.UNSET) {
                    Set<FastThreadLocal<?>> variablesToRemove = (Set)v;
                    FastThreadLocal<?>[] variablesToRemoveArray = (FastThreadLocal[])variablesToRemove.toArray(new FastThreadLocal[0]);
                    FastThreadLocal[] var4 = variablesToRemoveArray;
                    int var5 = variablesToRemoveArray.length;

                    for(int var6 = 0; var6 < var5; ++var6) {
                        FastThreadLocal<?> tlv = var4[var6];
                        tlv.remove(threadLocalMap);
                    }
                }
            } finally {
                InternalThreadLocalMap.remove();
            }

        }
    }

#### FastThreadLocal 的缺点 ？
当前线程每创建一个FastThreadLocal就会生成一个index，且这个index是一直递增的，导致Object[] 数组长度一直在递增，是典型的空间换时间的方式。

##### JDK 中的Thread使用FastThreadLocal
在netty 4.1.26 之前的版本，netty通过io.netty.util.internal.ObjectCleaner这个类解决JDK中的Thread使用FastThreadLocal产生内存泄漏的问题，从4.1.26开始， ObjectCleaner被移除。
具体原因可参考：
[netty-4.1.26](https://github.com/netty/netty/commit/5b1fe611a637c362a60b391079fff73b1a4ef912#diff-e0eb4e9a6ea15564e4ddd076c55978de)





