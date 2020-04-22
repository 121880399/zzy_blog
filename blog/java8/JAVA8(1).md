JAVA8你只需要知道这些（1）

#前言
 JAVA8发布有一段时间了，个人觉得JAVA8里的改动，有很多是颠覆性的。虽然目前用JAVA8的人还比较少，但是既然JAVA8中加入了这些新特性，就应该研究一下，毕竟以后总要用到。

#接口
要说到JAVA8中引入的新特性，我们就得从接口说起。在很长一段时间里，我们对接口的印象是这样的：
```
public interface IMessage {
	public void send(String msg);
}
```
我们一起来回想一下，关于接口的一些特性：
1.接口不能被实例化对象。
2.接口没有构造方法。
3.接口的方法只能定义为public abstract,定义为其他的会报错。
4.接口可以多重继承。
5.接口中不能包含成员变量，除了static final类型的。
6.接口中的方法是不能在接口中实现的，只能由实现接口的类来实现接口中的方法。
7.接口中不能有静态代码块和静态方法。

大概差不多就是这些，那JAVA8中的接口呢？在说JAVA8接口的特性之前，我们先来看这样一个场景：你有200个类都继承了同一个接口，但是现在这个接口需要添加一个新的方法。而且这个方法的实现都是相同的，那你该怎么办？我们只能一个类一个类去粘贴实现方法。你可能会说这种情况很少见，但是他的确存在并且很可能会发生。

所以JAVA8中的接口就引入了默认方法和静态方法。
```
public interface IMessage {
	/**
	 * 默认方法
	 * */
	public default void send(String msg){
		System.out.println(msg);
	}
	
	/**
	 * 静态方法
	 * */
	public static void receive(String msg){
		System.out.println(msg);
	}
}
```
默认方法和静态方法只用在方法名前面加上default和static就可以了。默认方法的用途我们在上面的场景中已经解释过了。关于静态方法的好处就是你可以直接在任何地方使用IMessage.receive调用该方法。

以后面试的时候，问你接口跟抽象类有什么区别？一定要告诉面试官，在JAVA8以后接口中的方法也可以实现了。并且在接口中也可以有静态方法。而这些特性以前只能在抽象类中有。

#匿名内部类
不知道你以前是否仔细观察过匿名内部类？匿名内部类在开发中还是比较常用的，尤其在android开发中，有很多监听事件和接口回调。
```
mButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(this,"Hello",Toast.LENGTH_SHORT).show();
            }
        });
```
这是一段android代码，功能是当点击按钮的时候，屏幕上弹出Hello。实现主要功能的其实就是一句话： Toast.makeText(this,"Hello",Toast.LENGTH_SHORT).show();
我们却写了这么多代码，有什么办法可以让代码精简一点呢？没错，就是JAVA8中引入的Lambda表达式。
```
mButton.setOnClickListener((v)->
                Toast.makeText(BuyContainerFilterActivity.this,"Hello",Toast.LENGTH_SHORT).show();
         );
```
经过Lambda表达式简化过后，就是这么简单，我们可以看见方法参数里面只传入了参数v，还有功能语句，省略了new接口等操作。我们一起来看看Lambda表达式的语法：
1.（参数）->单行语句
2.（参数）->{多行语句}
3.（参数）->表达式
语法倒是挺简单的，就是刚开始用不习惯，得自己反应一下，慢慢的就好了。

除此之外，匿名内部类的另外一个新特性就是：在匿名内部类中访问外部类的局部变量时，变量不用声明为final类型。

#结语
Java8的新特性就算入门啦，从下一篇起就一起来研究一些更复杂的特性。如果你觉得本篇文章帮助到了你，希望大爷能够打赏。
本文为原创文章，转载请注明出处！