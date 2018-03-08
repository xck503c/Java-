#类的源码注释

##HashTable和HashMap

1. 都实现了实现了Map接口
2. 它和HashMap的唯一区别就是HashMap不是同步的，而且HashMap允许出现空的key和value。
3. HashMap是无序的，且顺序是可变的。
4. 若hash函数将数据适当的分散到各个桶里面，那么Map接口提供的get和put操作将是常量时间的
5. 使用迭代器来遍历容器内所有数据所需求的时间和HashMap实例的capacity(桶的数量)加上桶的大小(键值对映射的数量)是成比例的(proportional)，所以如果迭代器的性能很重要，就不能设置initial capacity过高(或设置load factor过低)。

##HashMap实例中有两个影响其性能的参数：

1. initial capacity和load factor。
2. 其中capacity是hash table中bucket的数量。
3. load factor是指，散列表在其容量自动增加之前被允许获得的满量程度的度量(百分比)。有点拗口， 举个例子，初始容量16，默认负载因子0.75，0.75* 16=12，而这个12就是所能插入的极限。
4. 当散列表(哈希表)的entry个数超过负载因子和当前容量的乘积(entry个数>loadFactory*capacity)的时候，哈希表会进行重构(rehashed)(意思是说，重建内部数据结构)，hash table会扩容为原来的两倍左右。

load factor默认为0.75，该数值是权衡了时空开销得出的。负载因子越大会降低空间开销(space overhead)，但提高了查找开销(对HashMap类中大部分操作都成立，包括put，get)
	
1. 在设置初始容量的时候，我们需要考虑map中预计(expected)的entry个数和load factor，从而最小程度的减少rehash的次数。
2. 如果有很多的mapping要存入HashMap实例中，我们就需在创建该实例的时候要设置足够大的容量，这样mapping的储存效率会比扩容的时候自动重构的效率高得多
3. 请注意，如果有很多关键字的哈希值相同，将会降低哈希表的性能(performance)。为了降低(ameliorate)这个影响，当关键字支持java.lang.Comparable，我们可以对关键字进行排序来减少这个影响(break ties)

##需要注意的是HashMap并不是线程安全的

1. 当有多个线程同时(concurrently)访问哈希表，而且至少有一个线程修改了map的结构，那么必须对他们进行同步，通过同步对象来封装map实现，当然如果这些同步对象不存在，我们可以通过 Collections.synchronizedMap方法包裹(wrapped)map来实现同步。最好是在创建的时候完成，防止出现不同步访问map的意外情况。
	1. 结构的修改是指add或delete一个或多个mapping，但是如果仅仅是修改已经存在的键值，并不算是修改结构。
	2. Map m = Collections.synchronizedMap(new HashMap(...));

#接口实现的注意事项

#重要的变量成员

##1. 默认初始容量

	/**
	 * The default initial capacity - MUST be a power of two.
	 */
	static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

默认初始容量为16，通过移位运算符，说明它必须是2的幂

##2. 默认负载因子

	/**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

##3. 由链表转换成树的阈值(待定)

	/**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    static final int TREEIFY_THRESHOLD = 8;

##4. Node类型的table

	/**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;

用来储存桶的

##5. 缓存entrySet()

	/**
     * Holds cached entrySet(). Note that AbstractMap fields are used
     * for keySet() and values().
     */
    transient Set<Map.Entry<K,V>> entrySet;

##6. map结构修改计数器

	/**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
    transient int modCount;

#hash算法

通过计算key的哈希值和异或操作将高位散布到低位中。所以我们应用一个变化来向下(downward)分散高位的影响。该算法是在速度，效率(utility)，比特传播质量之间权衡(tradeoff)得出的。因为许多常见的哈希集合已经分布合理(所以我们并不能从spreading中受益)，并且我们使用树来处理箱子中的大量碰撞，所以我们只需要用开销最低的方式对其进行位移(shifted bits)和异或运算来减少系统的损失(systematic lossage)，同时也是为了吸收(incorporate)最高位的影响。否则由于表的界限，我绝对不会在索引计算中使用

	/**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

首先计算出key的哈希值，然后将该值右移16位，高位变低位。这样高16位经过异或操作后全部保留，就会变成高位和低位进行异或。

低位的信息中加入了高位的信息，这样高位的信息被变相的保留了下来。掺杂的元素多了，那么生成的hash值的随机性会增大。

#(n - 1) & hash

顺便说一下，这也正好解释了为什么HashMap的数组长度要取2的整次幂。因为这样（数组长度-1）正好相当于一个“低位掩码”。“与”操作的结果就是散列值的高位全部归零，只保留低位值，用来做数组下标访问。以初始长度16为例，16-1=15。2进制表示是00000000 00000000 00001111。和某散列值“与”操作如下，结果就是截取了最低的四位值。

#自定义数据结构

##结点Node<k,v>，链式储存结构

	/**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
   	static class Node<K,V> implements Map.Entry<K,V> {
		//键和哈希值无法更改  
		final int hash;
		final K key;
		V value;
		//用来实现链式储存的重要成员变量
		Node<K,V> next;

		Node(int hash, K key, V value, Node<K,V> next) {...省略}

		//键值的get方法是final
		...省略
		public final String toString() { return key + "=" + value; }

		public final int hashCode() {
			return Objects.hashCode(key) ^ Objects.hashCode(value);
		}

		//替换旧value，返回旧的value
		public final V setValue(V newValue) {...省略}


		//注意他的equals写法，首先比较两个内存引用是否相同==，若不是，则再来看是否同意实现了Entry接口，然后再比较它的key，value
		public final boolean equals(Object o) {
			if (o == this)
				return true;
			if (o instanceof Map.Entry) {
				Map.Entry<?,?> e = (Map.Entry<?,?>)o;
				//使用Objects的equals是为了防止两个参数之一或两个为null
				if (Objects.equals(key, e.getKey()) &&
					Objects.equals(value, e.getValue()))
					return true;
            }
            return false;
        }
    }

##TreeNode红黑树

###成员变量

	TreeNode<K,V> parent;  // red-black tree links
	TreeNode<K,V> left;
	TreeNode<K,V> right;
	TreeNode<K,V> prev;    // needed to unlink next upon deletion
	boolean red;

###左旋操作

	static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
											TreeNode<K,V> p) {
		TreeNode<K,V> r, pp, rl;
		if (p != null && (r = p.right) != null) {
			if ((rl = p.right = r.left) != null)
				rl.parent = p;
			if ((pp = r.parent = p.parent) == null)
				(root = r).red = false;
			else if (pp.left == p)
				pp.left = r;
			else
				pp.right = r;
			r.left = p;
			p.parent = r;
		}
		return root;
	}

#重要的函数

##储存键值对put(K key, V value)

	/**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

	/**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
	final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
	                   boolean evict) {
		Node<K,V>[] tab; Node<K,V> p; int n, i;
		//若table为空或长度为0，通过resize来创建table
		if ((tab = table) == null || (n = tab.length) == 0)
			n = (tab = resize()).length;
		//若tab[i]上没有结点，则创建结点存放
		if ((p = tab[i = (n - 1) & hash]) == null)
			tab[i] = newNode(hash, key, value, null);
		else {
			Node<K,V> e; K k;
			//若p=tab[i]上的结点哈希值和key值和给定的hash值和key值相同，则替换旧值
			if (p.hash == hash &&
	             	((k = p.key) == key || (key != null && key.equals(k))))
	            		e = p;
			//若p的类型是TreeNdoe，则将给定的结点插入树中
			else if (p instanceof TreeNode)
	            		e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
	            else {
				//遍历链表，查找给定的关键字和hash值
				for (int binCount = 0; ; ++binCount) {
					//已经遍历到最后一个了
					if ((e = p.next) == null) {
						p.next = newNode(hash, key, value, null);
						//若链上的结点超过阈值，树化
						if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
							treeifyBin(tab, hash);
						break;
					}
					//找到
					if (e.hash == hash &&
						((k = e.key) == key || (key != null && key.equals(k))))
	                    		break;
	                    	p = e;
				}
			}
			//若e不为空，则map中存在要添加的key
			if (e != null) { // existing mapping for key
				V oldValue = e.value;
				if (!onlyIfAbsent || oldValue == null)
					e.value = value;
				afterNodeAccess(e);
				return oldValue;
			}
		}
		//结构性修改要+
		++modCount;
		//已经超过重构阈值，重新调整大小
		if (++size > threshold)
			resize();
		afterNodeInsertion(evict);
		return null;
	}

##重构，resize

	/**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
	final Node<K,V>[] resize() {
		//保存当前table
		Node<K,V>[] oldTab = table;
		//当前table大小  
		int oldCap = (oldTab == null) ? 0 : oldTab.length;
		//当前table阈值
		int oldThr = threshold;
		int newCap, newThr = 0;
		//之前table大小大于0
		if (oldCap > 0) {
		//之前table大小大于最大容量1<<30
			if (oldCap >= MAXIMUM_CAPACITY) {
				//阈值为最大整型
				threshold = Integer.MAX_VALUE;
				return oldTab;
			}
		//容量加倍，左移效率高
			else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
					oldCap >= DEFAULT_INITIAL_CAPACITY)
				newThr = oldThr << 1; // 阈值加倍
        	}
		//table大小=0，阈值大于0
		else if (oldThr > 0) // 用threhold来初始化capacity
			newCap = oldThr;
		//oldCap=0，oldThr=0
		else {               // 0初始化阈值表示使用默认值
			newCap = DEFAULT_INITIAL_CAPACITY;
			newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
		}
		//若新的阈值还没有更新
		if (newThr == 0) {
			float ft = (float)newCap * loadFactor;
			newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
		}
		threshold = newThr;
		@SuppressWarnings({"rawtypes","unchecked"})
		//创建新的table
		Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
		table = newTab;
		if (oldTab != null) {
			//将原来map中非null的元素rehash之后再放到newTab里面去
			 for (int j = 0; j < oldCap; ++j) {
				Node<K,V> e;
				if ((e = oldTab[j]) != null) {
					//方便垃圾回收
					oldTab[j] = null;
					//若链表上只有一个结点，重新散列
					if (e.next == null)
						newTab[e.hash & (newCap - 1)] = e; 
					else if (e instanceof TreeNode)
						((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
					else { // preserve order
						//这里的目的是为了将同一桶中的元素根据(e.hash & oldCap)是否为0进行分割，分成两个不同的链表，将oldTab[j]上的结点进行重新散列到newTab中
						//其中若e.hash&oldCap==0，则该结点待在原地是第一种情况，并链如loHead中，反之放入hiHead中，而这里只有两种情况发生
						//1. newTab[j] = loHead;
						//2. newTab[j+oldCap] = hiHead;
						//DK1.8改进了rehash算法，扩容时，容量翻倍，新扩容部分，标识为hi，原来old的部分标识为lo
						// 声明了队尾和队头指针
						Node<K,V> loHead = null, loTail = null;
						Node<K,V> hiHead = null, hiTail = null;
						Node<K,V> next;
						do {
							next = e.next;
							if ((e.hash & oldCap) == 0) {
								if (loTail == null)
									oHead = e;
								else
									loTail.next = e;
								loTail = e;
							}
							else {
								if (hiTail == null)
									hiHead = e;
								else
									hiTail.next = e;
								hiTail = e;
							}
						} while ((e = next) != null);
						if (loTail != null) {
							loTail.next = null;
							newTab[j] = loHead;
						}
						if (hiTail != null) {
							hiTail.next = null;
							newTab[j + oldCap] = hiHead;
						}
					}
				}
			}
		}
		return newTab;
	}

一开始在类的注释中说到，应该尽量避免扩容处理，因为扩容处理会遍历所有的元素，时间复杂度高。虽然说经过扩容，元素会更加均匀的分布到各个桶中，提高访问效率，但是损耗的性能要大于提供访问效率带来的好处

	1. 在设置初始容量的时候，我们需要考虑map中预计(expected)的entry个数和load factor，从而最小程度的减少rehash的次数。
