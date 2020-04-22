我赌5块钱，你能看懂红黑树！（TreeMap源码分析）

&#160; &#160; &#160; &#160; TreeMap是通过红黑树实现的。要分析TreeMap的源码，就要对红黑树有一定的了解。而红黑树是一种特殊的二叉查找树，其红黑特性保证了**每个节点的左右子树高度差不超过两倍**，所以红黑树是一种接近平衡的二叉查找树。

##1.二叉查找树的特性
1.对于树中的每个节点T（T可能是父节点），它的左子树中所有的值都小于T中的值。
2.对于树中每个节点T，它的右子树中所有的值都大于T中的值。
![image.png](https://upload-images.jianshu.io/upload_images/1870458-32b7d148ff018c13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&#160; &#160; &#160; &#160;二叉查找树可以使用二分法很便利的进行查找，查找效率最优能达到O(log n)。在极端情况下查找效率为O(n)。例如下图：
![image.png](https://upload-images.jianshu.io/upload_images/1870458-12238d9aca4522f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&#160; &#160; &#160; &#160;该树一共有5个节点，当我要查找5这个节点时，就要一个一个节点进行对比，一共对比5次，所以查找效率为O(n)。
&#160; &#160; &#160; &#160;为了让查找效率变高，我们就要避免上图的情况出现，出现上图情况的主要原因是左右子树的高度相差太大。如果我们让每个节点的左右子树高度差不超过1，那么就实现的平衡二叉查找树，AVL树就是这样一棵平衡二叉查找树。
![image.png](https://upload-images.jianshu.io/upload_images/1870458-422c11b3a6223819.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&#160; &#160; &#160; &#160;AVL树的查找，插入，删除的时间复杂度都为O（log n），既然AVL树这么优秀，为什么TreeMap要使用红黑树？

##2.AVL树与红黑树的区别
&#160; &#160; &#160; &#160; 首先AVL树比红黑树更平衡，在插入和删除操作时就更容易引起树的旋转操作，来维持树的平衡性。在大量数据需要插入和删除的场景中，AVL树的rebalance的频率会大幅度上升。而红黑树是不符合AVL树的平衡条件的，红黑树用非严格的平衡来换取旋转次数的降低，任何的rebalance都能在三次旋转内完成，并且红黑树的时间复杂度也为O（log n），所以在实际应用中，红黑树的使用会更广。


##3.红黑树的特性
    1.每个节点或者是黑色，或者红色。
    2.根节点必须是黑色。
    3.如果一个节点是红色的，则它的子节点必须是黑色的。
    4.对于任一节点而言，其到叶节点树尾端NIL指针的每一条路径都包含相同数目的黑节点
    5.新添加的节点总是红色的。

&#160; &#160; &#160; &#160; 以上5点红黑树的特性一定要记住，因为在下面的代码分析过程中，会经常发现插入，删除节点后出现跟以上特性冲突的情况，这时候就需要进行修复。

##4.TreeMap中的数据结构：
&#160; &#160; &#160; &#160; 既然是树，那么就是由很多节点组成，我们先看一下TreeMap中的静态内部类Entry<k,v>，整棵红黑树就是由它组成。
```
    //用两个常量定义了红黑
    private static final boolean RED   = false;
    private static final boolean BLACK = true;

   
    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left; //左子树
        Entry<K,V> right;//右子树
        Entry<K,V> parent;//父亲
        boolean color = BLACK;//当前节点的颜色

        /**
         * Make a new cell with given key, value, and parent, and with
         * {@code null} child links, and BLACK color.
         */
        Entry(K key, V value, Entry<K,V> parent) {
            this.key = key;
            this.value = value;
            this.parent = parent;
        }

        /**
         * Returns the key.
         *
         * @return the key
         */
        public K getKey() {
            return key;
        }

        /**
         * Returns the value associated with the key.
         *
         * @return the value associated with the key
         */
        public V getValue() {
            return value;
        }

        /**
         * Replaces the value currently associated with the key with the given
         * value.
         *
         * @return the value associated with the key before this method was
         *         called
         */
        public V setValue(V value) {
            V oldValue = this.value;
            this.value = value;
            return oldValue;
        }

        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;

            return valEquals(key,e.getKey()) && valEquals(value,e.getValue());
        }

        public int hashCode() {
            int keyHash = (key==null ? 0 : key.hashCode());
            int valueHash = (value==null ? 0 : value.hashCode());
            return keyHash ^ valueHash;
        }

        public String toString() {
            return key + "=" + value;
        }
    }
```
&#160; &#160; &#160; &#160; 从代码中我们可以发现，每个节点存储了Key,Value，左子树，右子树，父节点，当前节点的颜色。

##5.TreeMap中左旋方法：
&#160; &#160; &#160; &#160;在红黑树的特性遭到破坏以后，我们会对红黑树进行修复，修复的工作包括对节点颜色的改变，对红黑树结构的调整，其中结构的调整又包括：左旋，右旋。
![image.png](https://upload-images.jianshu.io/upload_images/1870458-7b15dd40942c2cb0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&#160; &#160; &#160; &#160;从上图中可以看见，节点3发生左旋以后，成为了10的左孩子，10成为了局部的根节点，10以前的左孩子成为了3的右孩子。其实很容易理解，发生左旋后，3成了10的左孩子，10以前的左孩子怎么办？根据二叉查找树的性质，他只能成为3的右孩子。在这里总结出一个口诀，**左旋找右子树，右旋找左子树。旋转过后左旋成为左孩子，右旋成为右孩子。**
```
        /**
	* 左旋方法
	*/
	private void rotateLeft(Entry<K,V> p) {
	
		//p --- 将要左旋的节点
		//r --- 将要成为新的局部根节点
		
		//判断当前节点是否为空
    	if (p != null) {
    		//得到要旋转节点的右子树，该右子树将成为新的局部根节点
        	Entry<K,V> r = p.right;
        	//将新的根节点的左子树转变为要旋转节点的右子树，无论该左子树是否为NULL
        	p.right = r.left;
        	//如果新的根节点左子树不为Null,那么将该左子树的父亲设置为要旋转节点
        	if (r.left != null)
            	r.left.parent = p;
            	//以上两步其实是新的根节点的左子树与要旋转的节点 父亲认孩子， 孩子认父亲的过程。
        	
        	//将要旋转节点的父亲继承给新根节点
        	r.parent = p.parent;
        	//如果将要旋转的节点没有父节点，说明将要旋转的节点为根节点 那么旋转过后，新的根节点成为真正的全局根节点
        	if (p.parent == null)
            	root = r;
            	//如果p节点是父亲的左孩子，则新的根节点也应该继承为父亲的左孩子，否则为右孩子
        	else if (p.parent.left == p)
            	p.parent.left = r;
        	else
            	p.parent.right = r;
            	//再将p节点成为r节点的左孩子
        	r.left = p;
        	//r节点成为p节点的父亲
        	p.parent = r;
    	}
	}
```

##6.TreeMap中的右旋方法：
![image.png](https://upload-images.jianshu.io/upload_images/1870458-1994aebe14d9d901.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&#160; &#160; &#160; &#160; 从上图可以看见，它其实是左旋的一个逆过程，根据口诀，右旋找左子树，旋转过后右旋成为右子树，所以10就变成了3的右子树，以前3的右子树6就只能变成10的左子树。
 ```
private void rotateRight(Entry<K,V> p) {
  			//p --- 将要右旋的节点
  			//l --- 将要成为新的根节点  	
  
        	if (p != null) {
        		//要右旋得到p的孩子
            	Entry<K,V> l = p.left;
            	//将l节点的右孩子成为p节点的左孩子
            	p.left = l.right;
            	//如果l节点的右孩子不为null，那么将右孩子的父亲设置成p节点
            	if (l.right != null) l.right.parent = p;
            	//将p节点的父亲继承给l节点
            	l.parent = p.parent;
            	//如果p节点的父亲为null，说明p节点是全局根节点，那么l节点将成为新的全局根节点
            	if (p.parent == null)
                	root = l;
                	//如果p节点是父亲的右孩子，那么l节点也将成为父亲的右孩子，否则为左孩子
            	else if (p.parent.right == p)
                	p.parent.right = l;
            	else p.parent.left = l;
            	//p节点成为了l节点的右孩子
            	l.right = p;
            	//p节点的父亲为l节点
            	p.parent = l;
        	}
    	}
 ```
##7.通过key进行查找
&#160; &#160; &#160; &#160; 在TreeMap中进行查找相对比较容易，使用二分法思想进行查找就行了，如果对二分法查找有不了解的，自己百度一下吧。
![image.png](https://upload-images.jianshu.io/upload_images/1870458-745e6c185f6b7f7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
     public V get(Object key) {
 		//通过getEntry方法找到Entry 并且判断找到的Entry是否为空，如果不为空则返回value
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
    }
    
    //查找Entry
     final Entry<K,V> getEntry(Object key) {
        // Offload comparator-based version for sake of performance
        if (comparator != null)
            return getEntryUsingComparator(key);
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        Entry<K,V> p = root;
        //采用二分查找的方式查找key值
        while (p != null) {
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
               return p;
        }
        return null;
    }
```

##8.TreeMap中的添加新元素
&#160; &#160; &#160; &#160;添加新元素的思路很简单，也是根据二分法找到新添加元素要存放的位置，然后进行插入。

```
public V put(K key, V value) {
        	Entry<K,V> t = root;
        	//如果没有根节点，那么就将要添加的节点当做根节点
        	if (t == null) {
            	compare(key, key); // type (and possibly null) check
            	root = new Entry<>(key, value, null);
            	size = 1;
            	modCount++;
            	return null;
        	}
        	int cmp;
        	Entry<K,V> parent;
        	
        	Comparator<? super K> cpr = comparator;
        	//如果添加了自定义的比较器
        	if (cpr != null) {
        		//通过比较确定key值的位置，如果存在相同的key则覆盖value值
            	do {
                	parent = t;
                	cmp = cpr.compare(key, t.key);
                	if (cmp < 0)
                    	t = t.left;
                	else if (cmp > 0)
                    	t = t.right;
                	else
                    	return t.setValue(value);
            	} while (t != null);
        	}
        	else {
        	//没有添加自定义比较器，采用自然排序
            	if (key == null)
                	throw new NullPointerException();
            	@SuppressWarnings("unchecked")
                	Comparable<? super K> k = (Comparable<? super K>) key;
                	//通过比较确定key值的位置，如果存在相同的key则覆盖value值
            	do {
                	parent = t;
                	cmp = k.compareTo(t.key);
                	if (cmp < 0)
                    	t = t.left;
                	else if (cmp > 0)
                    	t = t.right;
                	else
                    	return t.setValue(value);
            	} while (t != null);
        	}
        	//确定key值的位置以后，新建节点
        	Entry<K,V> e = new Entry<>(key, value, parent);
        	//确定该节点是父节点的左孩子还是右孩子
        	if (cmp < 0)
            	parent.left = e;
        	else
            	parent.right = e;
            //进行插入修复
        	fixAfterInsertion(e);
        	size++;
        	modCount++;
        	return null;
    	}	
    	
```

##9.TreeMap中的插入后修复
&#160; &#160; &#160; &#160;红黑树的插入和删除都不难，难的在于插入后的修复和删除后的修复。我们知道红黑树中插入一个新节点，那么该节点一定是红色，如果该节点的父节点也是红色，那么就跟上面列出的红黑树特性3发生了冲突，因为一个节点是红色，那么它的左右子树必须是黑色。这个时候我们就要对新添加的节点进行修复工作。

![treeMap插入修复.png](https://upload-images.jianshu.io/upload_images/1870458-f6dc115cb6209d3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


&#160; &#160; &#160; &#160;为了理解插入后修复的流程，我自己做了一张流程图。别看流程图比较复杂，其实流程图的左右是对称的，实现的操作都差不多，仔细看就能发现，有些只是把左旋改成了右旋，右旋改成了左旋。用文字来表述无非以下情况：
1.叔节点为红色，直接把叔叔变黑，父亲变黑，祖父变红，然后当前节点指向祖父，继续循环判断，看祖父节点的父亲是否为红色，为红色即为有冲突，继续走流程判断。
2.叔节点不为红色，那么就判断父亲是祖父的左节点还是右节点，如果是左节点，那么就要判断当前节点是父节点的左节点还是右节点，如果是右节点，那么就得多操作一步，就是将父节点左旋，这时候3个节点就在一条直线上了。如果当前节点是左子树，那么3个节点已经在同一直线上，就不用旋转。 3个节点在同一直线以后，就把父亲变黑，祖父变红，然后祖父右旋。右旋过后可能会产生新的冲突，那么还需要继续循环检查。

&#160; &#160; &#160; &#160;所以总结起来，也就以上两种情况。而且再进一步总结可以发现，在插入修复的时候颜色的变化都有共同之处，就是总是**父节点都变黑，祖父都变红。**旋转方面则要看祖孙3代是否在同一直线，如果不在，则把3个节点先旋转成一条直线，如果直线是左斜那么最后是右旋转，如果直线是右斜那么最后是左旋转。就这么简单。

```
 	//插入后修复，修复包括颜色的调整和结构的调整
    	private void fixAfterInsertion(Entry<K,V> x) {
    		//新插入的节点设置为红色
        	x.color = RED;
			//如果新插入节点不为Null,不是根节点，新插入节点的父节点为红色 则开始循环
        	while (x != null && x != root && x.parent.color == RED) {
        		//parentOf 返回给定节点的父节点
        		//leftOf 返回给定节点的左子树
        		//如果x节点的父节点是祖父节点的左子树
            	if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            		//得到x的叔叔节点
                	Entry<K,V> y = rightOf(parentOf(parentOf(x)));
                	//如果叔叔节点是红色，则父亲节点跟叔叔节点都变黑，祖父节点变红
                	if (colorOf(y) == RED) {
                    	setColor(parentOf(x), BLACK);
                    	setColor(y, BLACK);
                    	setColor(parentOf(parentOf(x)), RED);
                    	//当前节点指向祖父节点，因为祖父节点变红以后，有可能出现冲突
                    	x = parentOf(parentOf(x));
                	} else {
                	//如果叔叔节点是黑色，父亲节点是红色，父亲节点是祖父节点的左子树
                	//这里包含两种情况，第一种情况是当前节点是父亲节点的右子树，
                    	if (x == rightOf(parentOf(x))) {
                    	//把当前节点指向当前节点的父节点
                        	x = parentOf(x);
                        	//然后把父节点左旋
                        	rotateLeft(x);
                    	}
                    //这里隐含着另外一种情况，就是当前节点是父节点的左子树，如果是左子树就不会执行上面if里面的代码
                    //右子树会多执行一步操作
                    //也就是说如果是当前节点是父节点的右子树的情况下，要把右子树旋转为左子树，然后进行下一步。
                    	//父节点设置为黑色
                    	setColor(parentOf(x), BLACK);
                    	//祖父节点设置为红色
                    	setColor(parentOf(parentOf(x)), RED);
                    	//然后把祖父节点右旋
                    	rotateRight(parentOf(parentOf(x)));
                	}
            	} else {
            	//如果当前节点的父节点是祖父节点的右孩子
            	//得到x节点的叔叔节点
                	Entry<K,V> y = leftOf(parentOf(parentOf(x)));
                	//判断叔叔节点是否为红色
                	if (colorOf(y) == RED) {
                		//如果叔叔节点为红色，父亲节点为红色，不论当前节点是左孩子还是右孩子
                		//都将父节点，叔叔节点设置为黑色，祖父节点设置为红色
                    	setColor(parentOf(x), BLACK);
                    	setColor(y, BLACK);
                    	setColor(parentOf(parentOf(x)), RED);
                    	//祖父节点设置为红色以后，有可能跟上一层有冲突，所以要把当前节点指向祖父节点来循环
                    	x = parentOf(parentOf(x));
                	} else {
                		//如果叔叔节点为黑色，父亲节点为红色，这里也分两种情况
                		//1.当前节点是父亲节点的左孩子，那么将父节点右旋转
                    	if (x == leftOf(parentOf(x))) {
                    		//x节点指向父节点
                        	x = parentOf(x);
                        	rotateRight(x);
                    	}
                    	//2.当前节点是父亲节点的右孩子，那么就不用执行以上右旋代码
                    	//将父亲节点设置为黑色，祖父节点设置为红色，祖父节点左旋转
                    	setColor(parentOf(x), BLACK);
                    	setColor(parentOf(parentOf(x)), RED);
                    	rotateLeft(parentOf(parentOf(x)));
                	}
            	}
        	}
        	//将根节点设置为黑色，根据红黑树的特性，根节点永远为黑色
       		 root.color = BLACK;
    	}
    	
```
&#160; &#160; &#160; &#160;最后上一张添加+修复的实例图，大家对照流程图，代码分析理解一下。
![image.png](https://upload-images.jianshu.io/upload_images/1870458-2c80787cf377221a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&#160; &#160; &#160; &#160;在上图中我们添加了6节点，添加过后发现父亲是红色的，产生了冲突，二话不说，看叔叔节点是否为红色，发现叔叔节点是红色，那好办了，父亲跟叔叔变黑，祖父变红，当前节点指向祖父继续遍历，发现10节点跟5节点也冲突，那么继续看叔叔节点3，发现3不是红色，就看10节点，5节点，20节点是否在一条直线上，发现不在一条直线，那么就要进行父亲旋转，左旋过后20,10,5就在同一条直线上了，就可以进行最后的操作了，父节点变黑，祖父变红，直线左斜那么祖父右旋转。最后发现当前节点5的父亲不是红色节点，退出循环，调整完毕。

##10.TreeMap中的删除：
&#160; &#160; &#160; &#160;TreeMap中的删除有三种情况，简单来说就是看删除节点是否还有孩子。
````
1.被删除节点p有左，右孩子，那么通过中序遍历找到p节点的后继节点进行顶替，
  若顶替节点有孩子，则孩子继承顶替节点的关系，并且删除顶替节点，如果顶替节点为黑色，则对其孩子进行修复
2.被删除节点p只有一个孩子，则让该孩子顶替，删除p节点，若p是黑色则对p的孩子进行修复
3.被删除节点p没有孩子，如果p是黑色则进行修复，p是红色那么直接进行删除
````
&#160; &#160; &#160; &#160;以上其实就是列出了被删除节点有2个孩子，有1个孩子，和没有孩子的情况，只是加上了颜色判断就显得有点复杂。
```
/**
 		* 删除key值对应的节点
 		*/
 		public V remove(Object key) {
 		//通过getEntry方法得到key值对应的Entry
        	Entry<K,V> p = getEntry(key);
        	if (p == null)
            	return null;
		
        	V oldValue = p.value;
        	//删除该Entry
        	deleteEntry(p);
        	//返回该Entry保存的value
        	return oldValue;
    	}
```
```
/**
    	* 通过key值找到Entry
    	*/
    	final Entry<K,V> getEntry(Object key) {
        // Offload comparator-based version for sake of performance
        if (comparator != null)
            return getEntryUsingComparator(key);
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        Entry<K,V> p = root;
        //采用二分查找的方式查找key值
        while (p != null) {
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
               return p;
        }
        return null;
    }
```
```
/**
    * 找到节点的后继节点，有两种情况:
    * 1.节点有右孩子，那么后继节点为右子树中最小的节点
    * 2.节点没有右孩子，那么只能不断向上找，找到第一个孩子节点是父亲的左孩子的情况，父亲节点就是后继节点。
    */
      static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
        if (t == null)
            return null;
        else if (t.right != null) {
        	//如果节点有右子树
            Entry<K,V> p = t.right;
            while (p.left != null)
            //不断循环找到右子树中最左边的节点，就是后继节点，因为这个是右子树中最小的节点。
                p = p.left;
            return p;
        } else {
        //如果节点没有右子树，那么只能往父亲那一辈不断往上找
            Entry<K,V> p = t.parent;
            Entry<K,V> ch = t;
            //将ch节点一直与父亲节点p进行比较，如果ch节点不是父亲的右孩子，那么此时的父亲节点就是后继节点
            //这里还需要注意，p节点为null时也会停止循环，也就是说最后如果找不到后继节点，将会返回null值。
            while (p != null && ch == p.right) {
                ch = p;
                p = p.parent;
            }
            return p;
        }
    }
```
&#160; &#160; &#160; &#160;在这里要注意successor这个方法，这个方法的功能其实是通过中序遍历找到被删除节点的后继节点来顶替它。但是从代码中我们发现跟我们以前见过的二叉树的中序遍历递归算法不一样是吧？所以在以上代码中分为两种情况：
&#160; &#160; &#160; &#160;第1种情况：被删除节点有右孩子的，这时候比较好办，就找到右子树中最左的那颗树，一定是它的后继节点，因为这个节点是右子树中最小的，而又比删除节点大的数，那可不就是后继节点吗？
&#160; &#160; &#160; &#160;第2种情况：被删除的节点没有右子树，那么就要向上找，一直找到孩子节点是父亲节点的左子树的情况，那么父亲节点就是后继节点。这么说可能有点抽象，我们看图。
![image.png](https://upload-images.jianshu.io/upload_images/1870458-b8cd7bc0064de74d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&#160; &#160; &#160; &#160;右边这张图就对应着我们说的第二种情况，要删除的节点30没有右子树，只能往上走，直到遇见第一个孩子是父亲的左子树，也就是18是35的左子树，那么35就是30的后继节点。我们再来看，如果删除的是9节点，那么它自己就是父亲的左子树，它的父亲18就是它的后继节点。

```
 /**
    * 删除节点有3种情况：
    * 1.被删除节点p有左，右孩子，那么通过中序遍历找到p节点的后继节点进行顶替，若顶替节点有孩子，则孩子继承顶替节点的关系，并且删除顶替节点，如果顶替节点为黑色，则对其孩子进行修复
    * 2.被删除节点p只有一个孩子，则让该孩子顶替，删除p节点，若p是黑色则对p的孩子进行修复
    * 3.被删除节点p没有孩子，如果p是黑色则进行修复，p是红色那么直接进行删除
    */
     private void deleteEntry(Entry<K,V> p) {
     	//修改次数加1
        modCount++;
        //树中的Entry减1
        size--;

        //如果删除节点P的左右子树都不为null，那么找到顶替p的节点以后，将p节点指向顶替节点。p节点永远指向的是将要被删除的节点
        if (p.left != null && p.right != null) {
        	//找到p节点的中序遍历后继节点，也就是比p大的节点中最小的节点。
            Entry<K,V> s = successor(p);
            //用该节点代替被删除的p节点
            p.key = s.key;
            p.value = s.value;
            //p节点完成赋值以后，将p节点指向替换节点，因为把s节点替换到p节点的位置以后，s节点应该被删掉
            p = s;
        } 
		
        
        Entry<K,V> replacement = (p.left != null ? p.left : p.right);
		
		//replacement不为Null 说明至少有一个孩子
        if (replacement != null) {
            
            replacement.parent = p.parent;
            //以下是让replacement节点继承p节点与父节点之间的关系
            //如果p节点的父亲为null，说明p节点是根节点，那么就让replacement节点成为新的根节点
            if (p.parent == null)
                root = replacement;
                //如果p节点是父亲的左孩子，那么replacement也成为父亲节点的左孩子
            else if (p == p.parent.left)
                p.parent.left  = replacement;
            else
            	//如果p节点是父亲的右孩子，那么replacement也成为父亲节点的右孩子
                p.parent.right = replacement;

            //p节点的关系被继承后，清除p节点所有的指向关系
            p.left = p.right = p.parent = null;

            //如果删除的p节点是黑色，那么就要进行修复，因为删除了一个黑节点，会导致每条路径上的黑节点数量不一致的情况，删除红节点则不用修复
            if (p.color == BLACK)
                fixAfterDeletion(replacement);
        } else if (p.parent == null) {
        	//如果replacement==null 说明p节点没有左右子树。如果p节点父亲也为null，则说明该树就只有p节点。那么root指向null。 
            root = null;
        } else { 
        	//如果replacement==null,p.parent!=null，说明p节点是叶子节点。如果p节点是黑色节点那么要进行修复。
            if (p.color == BLACK)
                fixAfterDeletion(p);
			//如果p节点有父亲，是父亲左孩子，那么将父亲左孩子设置为null，如果p是父亲的右孩子，则将父亲的右孩子设置为null,最后将p节点的父亲节点也设置为null。
			//断绝p节点与父亲的关系
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

![treeMap删除.png](https://upload-images.jianshu.io/upload_images/1870458-dd35034baa068307.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&#160; &#160; &#160; &#160;上面的删除代码写了很详细的注释，配合着流程图很容易看懂，流程图无非就是描述了删除节点时的三种情况，这里值得注意的有三点：
1.删除节点同时有左右子树的时候，我们通过successor方法找到后继节点替换删除节点，替换过后删除节点将指向替换的节点，因为这时候替换节点没用了，要删除掉。
2.如果删除节点有左子树或者右子树的情况，是先删除节点，再判断是否需要修复。而在删除叶子节点的情况下是先判断是否修复，再进行删除。
3.在有孩子的情况下如果进行删除修复，修复的是删除节点的孩子。而在叶子节点的情况下，修复的是删除节点。

```
  /**
    * 删除后修复
    **/
     private void fixAfterDeletion(Entry<K,V> x) {
     	//colorOf返回当前节点的颜色
     	//如果当前节点不是根节点，并且颜色是黑色
        while (x != root && colorOf(x) == BLACK) {
        	//leftOf 返回当前节点的左子树
        	//如果x节点是父节点的左子树
            if (x == leftOf(parentOf(x))) {
            	//得到兄弟节点
                Entry<K,V> sib = rightOf(parentOf(x));
				//兄弟节点如果是红色
                if (colorOf(sib) == RED) {
               		//兄弟节点变黑
                    setColor(sib, BLACK);
                    //父节点变红
                    setColor(parentOf(x), RED);
                    //父节点左旋
                    rotateLeft(parentOf(x));
                    //旋转完成以后，兄弟节点发生了变化，重新获得兄弟节点
                    sib = rightOf(parentOf(x));
                }
				
				//兄弟节点的左，右孩子都为黑色
                if (colorOf(leftOf(sib))  == BLACK &&
                    colorOf(rightOf(sib)) == BLACK) {
                    //将兄弟节点设置为红色
                    setColor(sib, RED);
                    //当前节点指向父节点
                    x = parentOf(x);
                } else {
                	//如果兄弟节点只有一个孩子为黑色，或者都不为黑色的情况
                	//如果兄弟节点的右孩子为黑色,左孩子为红色
                    if (colorOf(rightOf(sib)) == BLACK) {
                    	//设置兄弟节点的左孩子为黑色
                        setColor(leftOf(sib), BLACK);
                        //设置兄弟节点为红色
                        setColor(sib, RED);
                        //把兄弟节点往右旋转
                        rotateRight(sib);
                        //旋转完成以后，兄弟节点发生了变化，重新获得兄弟节点
                        sib = rightOf(parentOf(x));
                    }
                    //设置兄弟节点的颜色为父节点的颜色
                    setColor(sib, colorOf(parentOf(x)));
                    //设置父节点颜色为黑色
                    setColor(parentOf(x), BLACK);
                    //设置兄弟节点右子树颜色为黑色
                    setColor(rightOf(sib), BLACK);
                    //将父亲节点左旋
                    rotateLeft(parentOf(x));
                    //当前节点指向根节点
                    x = root;
                }
            } else { 
            	//以下是x节点是父节点右子树的情况
            	//得到兄弟节点
                Entry<K,V> sib = leftOf(parentOf(x));
				//兄弟节点颜色是红色
                if (colorOf(sib) == RED) {
                	//兄弟节点颜色变黑
                    setColor(sib, BLACK);
                    //父亲节点颜色变红
                    setColor(parentOf(x), RED);
                    //父节点右旋
                    rotateRight(parentOf(x));
                    //旋转完成以后，兄弟节点发生了变化，重新获得兄弟节点
                    sib = leftOf(parentOf(x));
                }
				//兄弟节点的左，右孩子都为黑色
                if (colorOf(rightOf(sib)) == BLACK &&
                    colorOf(leftOf(sib)) == BLACK) {
                    //设置兄弟节点为红涩
                    setColor(sib, RED);
                    //当前节点指向父亲节点
                    x = parentOf(x);
                } else {
                //如果兄弟节点只有一个孩子为黑色，或者都不为黑色的情况
                	//如果兄弟节点的左孩子为黑色,右孩子为红色
                    if (colorOf(leftOf(sib)) == BLACK) {
                    	//设置兄弟节点右孩子为黑色
                        setColor(rightOf(sib), BLACK);
                        //设置兄弟节点为红色
                        setColor(sib, RED);
                        //兄弟节点左旋
                        rotateLeft(sib);
                        //旋转完成以后，兄弟节点发生了变化，重新获得兄弟节点
                        sib = leftOf(parentOf(x));
                    }
                    //设置兄弟节点的颜色为父节点的颜色
                    setColor(sib, colorOf(parentOf(x)));
                    //设置父节点颜色为黑色
                    setColor(parentOf(x), BLACK);
                    //设置兄弟节点左孩子为黑色
                    setColor(leftOf(sib), BLACK);
                    //将父节点右旋
                    rotateRight(parentOf(x));
                    //当前节点指向根节点
                    x = root;
                }
            }
        }
		//将x节点设置为黑色
        setColor(x, BLACK);
    }
```
&#160; &#160; &#160; &#160;如果你要分析删除修复的情况，那相当多，我刚开始看网上的教程时，就在删除修复的地方直接被整懵了，相信很多学好红黑树的朋友也在这块放弃的。但是通过阅读代码以后，发现这块其实也没想象的那么复杂，而且很多也都是对称操作。所以我又做了一张流程图供大家参考。
![treeMap删除修复.png](https://upload-images.jianshu.io/upload_images/1870458-8c19356cdf58c1f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&#160; &#160; &#160; &#160;我们要明白一个道理，就是如果删除的节点是黑色的，那么就有可能跟红黑树的特性4发生冲突，这时候不能保证每条路径的黑节点都相等了。删除红色节点则没有关系。所以修复的前提都是基于，删除的节点是黑色的。
&#160; &#160; &#160; &#160;我们在上面删除方法中说到，当删除节点有孩子，并且删除节点是黑色的情况下，我们修复的是删除节点的孩子，如果删除节点的孩子是红色的，那么根据流程图，直接把孩子节点变成黑色，这是最简单的一种情况。
&#160; &#160; &#160; &#160;除了判断当前节点是左子树还是右子树，就剩下3种情况，兄弟节点是否为红？兄弟节点的俩孩子是否全是黑色？兄弟节点其中一个孩子是否为黑？（左子树看兄右，右子树看兄左）。
&#160; &#160; &#160; &#160;下面放一张实例图，大家对照流程图理解一下整个过程。
![image.png](https://upload-images.jianshu.io/upload_images/1870458-5827856f79c95cc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&#160; &#160; &#160; &#160;图分为两部分，分别为两种情况。先看左边，再看右边。顺序是从上往下看。

&#160; &#160; &#160; &#160;到此为止TreeMap的重要代码我们都分析过了，并且做了详细的注释，建议学习红黑树的童鞋，可以多参照TreeMap的代码研究。

&#160; &#160; &#160; &#160;接下来是广告环节，如果你觉得本文帮助到了你，请大爷们赏瓶买水钱。

本文参考：
1.[图解红黑树](https://www.jianshu.com/p/0eaea4cc5619)
2.[史上最清晰的红黑树讲解（上）](https://www.cnblogs.com/CarpenterLee/p/5503882.html)
3.[史上最清晰的红黑树讲解（下）](https://www.cnblogs.com/CarpenterLee/p/5525688.html)
注：本文的红黑树图来自于图解红黑树+史上最清晰的红黑树讲解，流程图自绘。