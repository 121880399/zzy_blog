JAVA8你只需要知道这些（2）

#前言
上篇文章我们讲了JAVA8中接口和匿名内部类的一些新特性。这篇文章我们将继续深入的研究JAVA8。

#１．Function包
　　在JAVA8中引入了java.util.function包，这个包里面全是接口，其中有四个接口值得我们注意：
1.Function 功能型接口
```
@FunctionalInterface
public interface Function<T, R> {
 R apply(T t);
}
```
　　需要注意的是@FunctionalInterface注解说明这是一个函数式接口，函数式接口中只能包含一个抽象方法，如果多于一个会报错，但是可以有默认方法，静态方法。
从Function接口来看，该接口接收一个参数返回一个参数。

2.Consumer 消费型接口
```
@FunctionalInterface
public interface Consumer<T> {
  void accept(T t);
}
```
Consumer接口接收一个参数，但是没有返回值。

3.Supplier 供给型接口
```
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```
Supplier接口不接收参数，但是有返回值。

4.Predicate 断言型接口
```
@FunctionalInterface
public interface Predicate<T> {
   boolean test(T t);
}
```
Predicate接口接收一个参数，返回一个boolean类型。
　　知道这4个接口有什么用呢？这4个接口是我们接下来要讲的stream的基础，在stream中，有大量的方法用到了以上的接口。

#２．Stream
　　Stream API真正的把函数式编程风格引入了JAVA。Stream主要用来对数据进行处理，尤其是对集合数据的处理。Stream Api对应的是java.util.stream包，在这个包下面有一个Stream接口，是我们主要研究的对象。
我们先来看看创建Stream的几种方式：
```
//1.使用Stream接口中的方法
Stream stream = Stream.of("1", "2", "3");
// 2. 使用Arrays.stream方法
String [] strs = new String[] {"1", "2", "3"};
stream = Arrays.stream(strs );
// 3. 使用Collection接口中的stream方法
List<String> list = Arrays.asList(strs );
stream = list.stream();
```
但是不论是用哪一种方式创建Stream，其内部都是通过以下代码实现：
```
 public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
        Objects.requireNonNull(spliterator);
        return new ReferencePipeline.Head<>(spliterator,
                                            StreamOpFlag.fromCharacteristics(spliterator),
                                            parallel);
    }
```
知道如何创建Stream以后，我们就要开始使用它：
```
String [] strs = new String[] {"java", "android", "ios"};
Arrays.stream(strs).filter((x)-> x.length()>3).forEach(System.out::println);
```
　　以上代码很简单，首先创建了一个String数组，然后通过Arrays.stream创建一个Stream，再通过filter方法把长度大于3的字符串过滤出来，最后通过System.out::println输出到控制台。
到这里就不得不提到Stream的操作了：
![Stream操作.png](http://upload-images.jianshu.io/upload_images/1870458-18acfd97299014db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
　　Stream操作分为中间操作和终端操作，中间操作就比如上面的filter方法，终端操作就比如上面的forEach方法。Stream的中间操作会惰性执行，只有执行到终端操作了才会立即去处理。这样的好处就是可以提升性能。
　　中间操作又分为有状态和无状态两种，有状态的操作必须等到所有元素处理完之后才知道最终结果，比如排序操作，在读取完所有元素前，就不能确定排序的结果。而无状态的操作中，元素的处理不受前面元素的影响，比如filter操作就是无状态的，当前元素的操作不会受到前面元素的影响。
　　终端操作也分为非短路操作和短路操作，短路操作即不用处理全部元素就可以返回结果，比如anyMatch方法，找到第一个满足条件的元素就返回。非短路操作就是要处理完全部元素才可以返回结果。

中间操作包括：map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip、 parallel、 sequential、 unordered

终端操作包括：forEach、 forEachOrdered、 toArray、 reduce、 collect、 min、 max、 count、 anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 iterator
#３．方法引用
　　可能细心的朋友已经发现了，在以上我们的代码中使用了System.out::println而不是System.out.prinlnt。这种语法我们以前似乎没有见过？对，它叫方法引用，这也是JAVA8中的新特性，我们来看看它的语法：
1.引用静态方法：　类名称::静态方法名
2.引用普通方法:　　实例化对象名::普通方法名
3.引用特定类方法:  　特定类::普通方法名
4.引用构造方法:　　类名称::new
　　你可能会说，不就是把以前的方法调用由.变成了::么？有什么可稀奇的？方法引用跟以前方法调用最大的区别在于，你可以用另外一个方法来引用该方法，说通俗一点就是可以给方法取一个别名，例如：

范例：把Int型转换为String
```
@FunctionalInterface
interface Action<P,R>{
	R convert(P p);
}

public static void main(String[]args){
		Action<Integer,String> a=String::valueOf;
		String str=a.convert(123);
		System.out.println(str);
}
```
　　在上面代码中，我们首先定义了函数式接口，关于函数式接口的定义我们上篇文章已经说过了，要是忘记了可以再回上篇文章看看。Action接口中有convert方法，接收一个参数，返回一个值。这跟我们String类中的valueOf方法一样，valueOf方法也是传入一个参数，返回一个值：
```
public static String valueOf(int i)
```
　　所以我们就可以用convert来引用valueOf方法，可以看见我们的convert方法是没有被实现的，他的功能完全取决于valueOf方法。并且因为valueOf方法是静态的，所以我们按照方法引用的语法，可以用类名::静态方法名直接引用。

范例：把小写字符串转换为大写
```
@FunctionalInterface
interface Action<R>{
	R convert();
}
public static void main(String[]args){
		Action<String> a="abc"::toUpperCase;
		String str=a.convert();
		System.out.println(str);
}
```
　　在上面代码中，我们通过方法引用把小写的abc，转换为大写的ABC，在使用方法引用的时候，我们用了"abc"这就是String类的一个实例，而toUpperCase的定义是一个普通方法：
```
public String toUpperCase()
```

范例：比较两个字符的大小
```
@FunctionalInterface
interface Action<P>{
	int  convert(P p1,P p2);
}
public static void main(String[]args){
		Action<String> a=String::compareTo;
		System.out.println(a.convert("a","b"));
｝
```
在以上代码中我们引用了compareTo方法来比较两个字符串的大小。但是我们来看该方法的定义:
```
public int compareTo(String anotherString)
```
　　它竟然不是静态方法，我们却用类名直接引用，这是为什么？正常情况下，类名::方法名，这个方法肯定是静态方法。但是为什么这里普通方法也可以呢？这就要分析一下compareTo这个方法了，这个方法是两个字符串进行对比，正常情况下的写法应该是"a".compareTo("b")，对应的方法引用应该是"a"::compareTo。
如果是这样，我们定义函数式接口的时候，就应该定义成以下这样：
```
@FunctionalInterface
interface Action<P>{
	int  convert(P p1);
}
	
	public static void main(String[]args){
		Action<String> a="a"::compareTo;
		System.out.println(a.convert("b"));
}
```
　　如果接口定义成这样，就可以使用"a"::compareTo这种形式的方法引用。到这里聪明的你应该明白，为什么在上面可以通过类名引用普通方法了吧？我们想想在String类里面除了compareTo还有其他方法没？很多是不是？equals，endsWith都属于，所以他们都可以通过把方法参数定义为两个，然后用类名::方法名。

```
class ShopCar{
	private String name;
	private double price;
	private int amount;
	public ShopCar(String name,double price,int amount){
		this.name=name;
		this.price=price;
		this.amount=amount;
	}
	
	public int getAmount() {
		return amount;
	}
	
	public String getName() {
		return name;
	}
	
	public double getPrice() {
		return price;
	}
	
	@Override
	public String toString() {
		// TODO Auto-generated method stub
		return "商品名:"+this.name+" 价格:"+this.price+" 数量:"+this.amount;
	}
}

@FunctionalInterface
interface Action<C>{
	C  create(String name,double price,int amount);
}

public static void main(String[]args){
		Action<ShopCar> a=ShopCar :: new;
		ShopCar car=a.create("苹果",5,2);
		System.out.println(car);
}
```
以上代码演示了如何通过类名称::new 来构造一个ShopCar对象。

结语：
　　今天的文章主要讲了JAVA8新引入的两个包和方法引用，其中Function包是基础，大家一定要多多理解。Stream包是函数式编程风格的主要提现，下一篇文章我们会着重介绍Stream接口中的方法。
如果你觉得本篇文章帮助到了你，希望大爷能够打赏。
本文为原创文章，转载请注明出处！