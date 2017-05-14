title: Thinking in Java
date: 2016-07-09 23:37:53
tags: Java
---
写一些看书的笔记
##第二章 一切都是对象
###JAVA堆栈和堆
&emsp;&emsp;堆栈就是平时所说的栈，位于通用的ROM中，从堆栈指针从处理器那里获得直接支持。堆栈的指针向下移动，分配新的内存，向上移动，释放内存。在创建程序时，系统必须知道存储在盾战中所有的项目的确切生命周期，以便赏析移动堆栈的指针。这一约束限制了程序的灵活性，所以虽然某些JAVA数据存储在堆栈中--特别是对象引用，但是JAVA对象并不存储其中（~~然而我并不怎么理解~~。。）。
&emsp;&emsp;堆是一种通用的内存池，也是位于RAM中，用于存放所有的JAVA对象。堆对于栈的好处就是：编译器不需要知道存储的数据在堆里存货多长时间。当需要一个对象时，只需要去new，当执行这个代码会自动在堆内进行储存分配。但是用堆进行存储分配和清理可能比用栈分配需要更多的时间。
<!--more-->
###基本类型
基本类型是存储在堆里，不用new来创建变量，而是创建一个并非是引用的"自动"变量（~~看不懂...555~~）。基本类型具有包装器类，使得可以在堆中创建一个非基本类型，用来表示对应的基本类型。
可以这么用`int i = 5`也可以`Integer i = new Integer(5)`。在JAVA se5之后的自动包装功能能自动的将基本类型转换为包装器类型：`Character ch = 'x'`
###基本成员默认值
若类的某个成员是基本数据类型，即使没有初始化，JAVA也会确保它获得一个默认值。但是局部变量不会初始化。
##第三章 操作符
###赋值
赋值操作的是一个对象的引用，看一个例子：

```
  class Tank{
	int level;
    }
      public class Assignment{
      public static void main(String[] args){
		Tank t1 = new Tank();
		Tank t2 = new Tank();

		t1.level = 9;
		t2.level = 21;

		System.out.println("原本的t1:"+t1.level+"  原本的t2:"+t2.level);

		t1 = t2;

		System.out.println("现在的t1:"+t1.level+"  现在的t2:"+t2.level);

		t1.level = 40;
		
		System.out.println("后来的t1:"+t1.level+"  后来的t2:"+t2.level);

	}
  }
  
```

  


看输出：

```
    原本的t1:9  原本的t2:21
    现在的t1:21  现在的t2:21
    后来的t1:40  后来的t2:40
```
   
正确的给t1赋值应该这样：

    t1.level = t2.level
###自动递增和递减
&emsp;&emsp;这个是我一直搞混的东西，总是弄混前缀和后缀的区别，现在写一下：

```
int i = 1;
System.out.println("i:"+i);
System.out.println("i++ :"+ i++);//1
System.out.println("++i :" + ++i);//3
```
 i++是先生成值后计算，也就是说在`i++`这里面的i是上一个状态的i,结果要在下一个状态才可以获取。前缀写法是先执行计算然后生成值，所以这里的i就等于三。
###关系操作符
&emsp;&emsp;关系操作符==和!=比较的是对象的引用，这个和在使用=进行对象赋值时一样，都是对象的引用而不是内部值。例子：

```
Integer n1 = new Integer(1);
Integer n2 = new Integer(1);
System.out.println(n1 == n2);//false
System.out.println(n1 !=n2);//true
```
要是想比较实际内容，可以使用所有对象都适用的方法`equals()` 。但是这个方法不适用于基本类型，基本类型直接使用`==`和`!=`即可。但是在自己创建的类中，还要注意`euqals()`的坑：

```
class Value{
			int i;
		}
		Value v1 = new Value();
		Value v2 = new Value();
		System.out.println(v1.equals(v2));//false
```
    
这是为啥？因为JAVA的设计者把对关系比较的权利完全给了类的设计者，因为`equasl()`方法对所有的对象都可以使用，所以他直接是卸载`Object`类里面。要想实现我们希望的功能，就要在自己的类中覆盖这个方法。这个方法的默认是返回对引用的比较。

     public boolean equals(Object obj) {
        return (this == obj);
    }

总结就是：想比较实际内容，用`equals()`，想比较引用，使用`==`。（~~感觉一般也很少直接进行对象的比较吧~~）。
###指数计数法
在JAVA中，看到1.39e^{-43}f这样的表达式，真正的意义是1.39*10^{-43}，这个历史原因我也懒得管，要使用e有现成的公式调用（~~我还没在开发中用过e..~~）。
###截尾和舍入
在将`float`和`double`转化为`int`的时候，默认是对该数字进行**截尾**，就是直接舍弃小数点后面的内容。要想得到舍入的结果，可以使用`Math.round()`方法；

##第五章 初始化与清理
###区别重载方法
区别重载方法是靠方法的参数列表的不同来区分，甚至参数的顺序不同也可以区分，但是不建议这么做，太特么混乱了。
###清理:终结处理和垃圾回收
JAVA主要要gc进行自动的垃圾回收，但是如果使用的对象不是new出来的，他获得了一块"特殊的"内存区域，由于垃圾回收器只知道回收那些由new分配的内存，所以它不知道如何去释放这块内存。为了对付这样的场景，JAVA允许在类中定义一个`finalize()`的方法，但是又不是像c++中的析构函数。（~~特么编不下去了，我不懂啊，不写这玩意了~~）。书上有三个要点还是写一下吧：

 1. 对象可能不被垃圾回收
 2. 垃圾回收不等于析构
 3. 垃圾回收只与内存有关
 
ps:书上还有很多关于如何使用`finalize()`和JAVA是如何进行垃圾回收的的东西，我对gc没点了解，暂时不看了。。
###构造器初始化
可以使用构造器进行初始化。在运行时刻，可以调用方法或者进行某些动作来确定初值。但是：**无法阻止自动初始化的进行，它将在构造器被调用之前发生**。比如：

``` 
class Counter{
int i;
Counter(){
i = 7;
  }
}
``` 

这里的i先被置为0，然后变成7。
在类的内部，定义变量的顺序决定了初始化的顺序，即使变量定义散步在方法之间，**它们仍然会在任何方法(包括构造器)被调用之前得到初始化**

```
class Window{
		 Window(int i){
			    System.out.println("window:"+i);
			}	
		}
		   class House{
			Window w1 = new Window(1);
			House(){
			System.out.println("House()");
			w3 = new Window(33);
			}
			Window w2 = new Window(2);
			void f(){System.out.println("f()");}
			Window w3 = new Window(3);
			}
	
	House h= new House();
	h.f();
		//---------结果--------
		window:1
        window:2
        window:3
        House()
        window:33
        f()
```
		
 

##第七章 复用类
###组合和继承
&emsp;&emsp;组合和继承都允许在新的类中放置子对象，组合是显示的这么做，继承是隐式的做。组合技术通常用于想在新类中使用现有类的功能而非它的接口这种情形，即新类中嵌入某个对象，让其实现所需要的功能，但新类的用户看到的只是为新类所定义的接口，而非所嵌入对象的接口，让其实现所需要的功能，需要在新类中嵌入一个现有类的private对象。
&emsp;&emsp;判断应该使用继承还是组合，就是是否需要从新类向上转型。如果需要，继承是必须的。
###final

 1. final的数据
 一个永不改变的编译时常量。
 一个运行时被初始化的值，不希望被改变。

final修饰的是基本类型时数值保持不变，但是final修饰的是一个对象时，是对引用保持不变，但是对象又是可以修改的，所以用final修饰对象作用不大。

&emsp;&emsp;空白final
空白final是指声明为final但是又没有给定初始值的域。这样当我们使用不同的构造函数创建不同的类时，就可以根据对象的不同有所不同，但是又保持不变。但是必须在每个域的定义处或者每个构造函数处对final赋值。

 1. final方法

 把方法锁定，以防任何继承类修改它的含义
 确保在继承中使方法的行为保持不变。（~~子类对父类的方法只能看不能摸~~）
 类中的所有的private方法都隐式的指定为final
 
##第八章 多态
&emsp;&emsp;多态是继 数据抽象 和 继承 之后的第三种基本类型。"封装"通过合并特性和行为来创建新的数据类型。多态的作用消除**类型**之间的耦合关系。
###域和静态方法
只有普通的方法的调用是多态的，成员变量没有多态。（通过get和set方法去调用，而不是直接访问成员）。
如果方法是静态的，也没有多态性。（一般的工具类中的静态方法，子类没法重写）。
###构造函数和方法
构造函数不具有多态性，实际上他们是隐式的static方法。
复杂对象的构造方法调用顺序：

 1. 基类的构造函数（一直递归下去，直到子类）
 2. 按声明顺序调用成员的初始化方法。
 3. 调用子类的构造函数主体。
 

###构造函数内部的多态方法的行为
先看个例子：

```
public class Main1 {
	public static void main(String[] args) {
		new RoundGlyph(5);
	}
     }
     class Glyph{
	void draw(){System.out.println("Glayph.draw");}

	Glyph(){
		System.out.println("before draw");
		draw();
		System.out.println("after draw");
	}
     }
    class RoundGlyph extends Glyph{
	private int radius = 1;
	RoundGlyph(int r){
		this.radius = r;
		System.out.println("RoundGlyph.RoundGlyph(): "+radius);
	}

	@Override
	void draw() {
		System.out.println("RoundGlyph.draw(): "+radius);	
	}
    }
```

    


输出：

    before draw
    RoundGlyph.draw(): 0//此时，还没有调用子类的构造方法，也就是还没有初始化，radius默认被赋值0
    after draw
    RoundGlyph.RoundGlyph(): 5

初始化的实际过程：

 1. 在其他任何事物发生之前，将分配给对象的储存空间初始化为二进制的零。
 2. 调用父类的构造函数
 3. 按声明顺序调用成员的初始化方法
 4. 调用子类的构造函数
 
因此要避免在构造函数内调用可能会重写的方法，安全的做法是调用final的方法。

##第九章 接口
&emsp;&emsp;接口表示：所有实现了该特定接口的类看起来都像这样。接口被用来创建类与类之间的协议。接口不仅仅是一个极度抽象的类，因为他允许人们通过创建一个能够向上转型为多种基类的类型，来实现某种类似多重继承变种的特性。
&emsp;&emsp;如果接口不添加public修饰，那么他默认是包访问权限。
&emsp;&emsp;这里还有对策略模式和适配器模式的介绍，这里对会在设计模式系列里面介绍。
&emsp;&emsp;使用接口的核心原因：为了能够向上转型为多个基类型。第二：防止创建该类的对象，并确保这仅仅是建立一个接口。
&emsp;&emsp;可以通过对接口的继承来进行接口的拓展，而且接口可以多重继承。
    
```
 interface Monster{
	void menace();
    }
    interface DangerousMonster extends Monster{
	void destory();
    }
     interface Lethal{
	void kill();
    }

    interface Vampire extends Lethal,DangerousMonster{
	void drinkblood();
    }
```


&emsp;&emsp;接口最吸引人的地方是允许同一个接口具有多个不同的实现。一个类 可以实现多个接口就可以实现向上转型为多个基类型   
&emsp;&emsp;有接口，肯定就会想到工厂模式，这也会在设计模式系列中写到（~~但愿我不拖更555~~）。
###抽象类与接口
（~~不负责任总结~~）
让抽象类去实现接口 不需要实现全部的方法 可以只实现一部分  然后剩下的子类去实现 子类必须实现抽象类中的抽象方法 
感觉抽象类可以作为普通类的一个缓冲   接口是完全的抽象  当只需要具有接口的部分特性  可以使用抽象类
一个类实现了一个接口  就可以把这个类当做接口的实例穿进去

##第十章 内部类
* 内部类就是将一个类的定义放在另一个类的内部。内部类和组合是不一样的概念。
* 内部类可以访问到外围对象的所有成员而不需要任何的特殊条件，此外，内部类还拥有外部类的所有元素的访问权。
* 在内部类中生成对外部类的引用，可以使用`Outside.this`
* 创建内部类的对象 `Out.Inner in = out.new Inner();`
* 在拥有外部类对象之前是不可能创建内部类的对象的，除非是嵌套类（静态内部类）
* 将内部类向上转型为基类型，尤其是转型为一个接口时候，内部类就很有用了

```
	public class Main3 {
			public static void main(String[] args) {
				RelaMain main = new RelaMain();
				Contents c = main.contents();
				Destination d = main.destination("laal");
				
				RelaMain.PDestination pd = main.new PDestination("hahah");
				//RelaMain.PContents pc = main.new PContents();访问不到  因为是private的
			}
		
		}
		class RelaMain{
			private class PContents implements Contents{
				private int i  = 1;
				public int value() {
					return i;
				}
			}
			protected class PDestination implements Destination{
				private String label;
				PDestination(String whereto){
					this.label = whereto;
				}
				public String readLabel() {
					return label;
				}
			}
			public Destination destination(String label){
				return new PDestination(label);
			}
			public Contents contents(){
				return new PContents();
			}
		}
		interface Destination{
			String readLabel();
		}
		interface Contents{
			int value();
		}
```
	



* 可以在方法里面或者任意的作用域内定义内部类：
  1：实现某类型的接口，于是可以创建并返回对其的引用；
  2：创建一个类来辅助你的解决方案，又不希望这个类是公共可用的。

* 匿名内部类和工厂方法

```
interface Service{
	void method1();
	void method2();
}
interface ServiceFactory{//抽象工厂接口 用于返回工厂产品
	Service getService();
}

class Implementation1 implements Service{

	public void method1() {
		System.out.println("Implementation1.method1");
	}

	public void method2() {
		System.out.println("Implementation1.method2");
	}
	public static ServiceFactory factory = new ServiceFactory() {

		public Service getService() {
			return new Implementation1();
		}
	};

}

class Implementation2 implements Service{

	public void method1() {
		System.out.println("Implementation2.method1");
	}

	public void method2() {
		System.out.println("Implementation2.method1");
	}
	public static ServiceFactory factory = new ServiceFactory() {

		public Service getService() {
			return new Implementation2();
		}
	};

}
public class Main2 {
	
	public static void serviceConsumer(ServiceFactory factory){
		Service s = factory.getService();
		s.method1();
		s.method2();
		
	}
	public static void main(String[] args) {

		serviceConsumer(Implementation1.factory);
		serviceConsumer(Implementation2.factory);
	}
}
```
###嵌套类 

如果不需要内部类对象对其外围类对象之间的联系，那么可以将内部类声明为static，叫做嵌套类。

 1.要创建嵌套类的对象，并不需要其外围类的对象
 
 2.不能从嵌套类的对象中访问非静态的外围类对象
 
```
	private static class ParcelDestination implements Destination{
		private String label;
		ParcelDestination(String whereto){
			this.label = whereto;
		}
		public String readLabel() {
			return label;
		}
		static class AnotherLevel{
			public static void f(){};
		}

	}
	public static Destination destination(String s){
		return new ParcelDestination(s);
	}
	//in main
	Destination dd = destination("lal");
		dd.readLabel();
		ParcelDestination.AnotherLevel.f();
```

### 接口中的内部类

如果想创建某些公用代码，使得他们可以被某个接口的所有不同实现所公用


```
/**
 * 在接口内可以定义类
 * @author Mr_Wrong
 * 
 * 只有实现了这个接口的类  才能访问到这个内部类  所以可以被实现了这个接口的所以类所共用
 *
 */
public interface ClassInInterface {
	void howby();
	class TestClassInInterface implements ClassInInterface{
		public void howby() {
			System.out.println("这是接口内部的类的howby方法");			
		}
		public static void main(String[] args) {
			new TestClassInInterface().howby();
			System.out.println("这是共用的部分");
		}
	}
}

```
###为什么需要内部类

一般来说，内部类继承自某个类或则实现某个接口，内部类的代码操作创建它的外围类对象，所以可以认为内部类提供了某种进入外围类的窗口。

每个内部类都能独立的继承自一个（接口）的实现，所以无论外围类是否已经继承了某个（接口）的实现，对于内部类都是没有影响的。（内部类也有种多重继承的感觉）

内部类的其他特征：

1.内部类可以有多个实例 ，每个实例都有自己的状态信息，并且与其外围类的信息相互独立。

2.在单个外围类中，可以让多个内部类以不同的方式实现一个接口，或者继承自同一个类。

3.创建内部类对象的时刻并不依赖外围类对象的创建。

4.内部类冰没有令人迷惑的"is-a"关系，它就是一个独立的实体。
###闭包与回调

闭包是一个可调用的对象，它记录了一些信息，这些信息来自创建它的作用域。

这是一个外围类实现一个接口和内部类实现这个接口的区别：

```
class Test2 implements TestInterface{

	public String dothings() {
		return "hasdone";
	}
	public TestInterface getinterface(){
		return new Test2();
	}
}

class Test3{

	public String dothings(){
		return "test3hasdone";
	}
	class InTest3 implements TestInterface{

		public String dothings() {
			return "InTest3hasdone";
		}

	}
	TestInterface getinterface(){
		return new InTest3();
	}

}



public class Test1 {

	public static void main(String[] args) {
		TestInterface interface1 = new Test2();
		System.out.println(interface1.dothings());
		
		
		Test3 test3 = new Test3();
		System.out.println(test3.getinterface().dothings());
	}
}

```

因为在test3中有dothings这方法，我们又不想因为实现了接而覆盖掉它，这时候就可以定义一个内部类去代替它去实现这个接口，然后在通过一个方法暴露出去。这就是一个hook（勾子）

###内部类的继承
内部类的继承有一些特殊的语法，还是直接上代码吧

```
class WidthInner {
	class Inner{}
}

public class InheritInner extends WidthInner.Inner{
//构造函数要传外围类
	InheritInner(WidthInner wi){
		wi.super();
	}
	public static void main(String[] args) {
		WidthInner wi = new WidthInner();
		InheritInner hi = new InheritInner(wi);

	}
}

```

##第十一章 持有对象
###添加元素
当使用Arrays.aslist时候应该注意类型问题：

```
public class Main {
	public static void main(String[] args) {
		
		List<Snow> snow1 = Arrays.asList(new Crusty(),new Powder(),new Slush());
		
		//List<Snow> snow2= Arrays.asList(new Light());//这样提示错误  下面两个可以
		
		List<Snow> snow3 = new ArrayList<Snow>();
		Collections.addAll(snow3, new Light(),new Heavy(),new Slush());
		
		List<Snow> snow4 = Arrays.<Snow>asList(new Light(),new Slush());
		

	   }
	}
	class Snow{};
	class Powder extends Snow{};
	class Light extends Powder{};
	class Heavy extends Powder{};
	class Crusty extends Snow{}
	class Slush extends Snow{}

```

###迭代器



```
	public class Main1 {
		public static void main(String[] args) {
			
			
			List<String> strings = Arrays.asList("hah","lal","en","yes","no");
	
	        //可以这样  包装成真正的list
			List<String> list = new ArrayList<String>(Arrays.asList("hah","lal","en","yes","no"));
			
			Iterator<String> it  = strings.iterator();
	
			while (it.hasNext()) {
				String string = (String) it.next();
				System.out.println(string);
			}
			
			for (String string : strings) {
				System.out.println(string);
			}
			
			it  = list.iterator();
			for (int i = 0; i < 2; i++) {
				it.next();
				it.remove();//这里不能进行对strings进行remove  是因为这个list仅仅是对数组的的包装   数组有确切的长度  不能进行remove操作
				//在使用迭代器的remove之前要先调用next
			}
			System.out.println(list);
		}
	}
```

ListIterator是Iterator的一个子类，只能用于各种list类的访问，它可以双向移动：

```
ListIterator<String> listit = list.listIterator();
		
		while (listit.hasNext()) {
			String string = (String) listit.next();
			System.out.println(string+","+listit.nextIndex()+","+listit.previousIndex());
		}
		while (listit.hasPrevious()) {
			System.out.println(listit.previousIndex());
			listit.set("");
		}
```

###栈

```
public class Main2 {
	public static void main(String[] args) {
		Stack<String> stack = new Stack<String>();
		for (String s : "my dog is dog ha ha ha".split(" ")) {
			stack.push(s);
		}
		while(!stack.empty()){
			System.out.println(stack.pop());
		}
	}
}
class Stack<T>{
	private LinkedList<T> storage = new LinkedList<T>();
	public void push(T v){storage.addFirst(v);}
	public T peek(){return storage.getFirst();}
	public T pop(){return storage.removeFirst();}
	public boolean empty(){return storage.isEmpty();}
	public String tosString(){return storage.toString();}
}

//输出
ha
ha
ha
dog
is
dog
my
```
###Set
set不保存重复的元素，查找就是set中最重要的操作。通常使用一个hashset，它专门为快速查找进行了优化：

```
	Random rand = new Random();
		Set<Integer> intset = new HashSet<Integer>();

		for (int i = 0; i < 100; i++) {
			intset.add(rand.nextInt(10));
		}
		System.out.println(intset);
		//输出
		[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

其实hashset的输出是没有顺序的，要想有顺序可以使用treeset，但是这里为什么是有顺序的呢，这也算是一个巧合吧，具体的可以看这里  知乎上的解释[Java遍历HashSet为什么输出是有序的？](http://www.zhihu.com/question/28414001)

treeset将元素储存在红黑树数据结构中，而HashSet使用的是散列函数。LinkedList因为查询速度的原因也使用了散列，但是看起来它使用了链表来维护元素的插入顺序。

使用treeset可以指定排序方式。

    		//按字母表排序
    		Set<String> words = new TreeSet<String>(String.CASE_INSENSITIVE_ORDER);


###Map

###Queue

队列是一个典型的先进先出的容器。

LinkedList提供了方法以支持队列的行为，并且它实现了queue接口，因此linkedklist可以作为queue的一种实现。

一些queue的常用方法：

offer()将一个元素插入到队尾或者返回false;
peek()和element()都在不移除的情况下返回队头，但是peek()方法在队列为空时返回Null，element()会抛出NoSuchElementException异常。
Poll()和remove()移除并返回队头，当队列为空时，同上。

###Collection和Iterator
Collection是描述所有序列容器的共性根接口。
要实现Collection就意味着需要提供iterator()方法。

###Foreach与迭代器

如果你创建了任何实现了Iterable的类，都可以将它用于foreach语句中。

尝试把数组当做一个Iterable参数传递会导致失败。这说明不存在任何从数组到Iterable的自动转换，必须手动执行这种转换。


###总结

 1. 数组将数字与对象联系起来，但是数组一旦生成，其容量就不能改变。
 2. Collection保存单一的元素，Map保存相关联的键值对。容器不能持有基本类型，但是自动包装机制会执行双向转换。
 3. 进行大量随机的访问，使用ArrayList。经常从表中间插入或者删除元素，使用LinkedList.
 4. 各种Queue以及栈的行为，由linkedlist提供支持。

![](http://img.my.csdn.net/uploads/201012/10/0_129197029221PZ.gif)

##第十二章 通过异常处理错误


这章暂时先不看了，处理异常的内容我觉得先暂时了解了一些，在写Android用处没那么多，以后要用的时候再来从头看。

##第十三章 字符串

* String对象是不可变的
* 在使用"+"进行String的拼接的时候，隐式的调用了StringBuilder来进行处理。这不意味着可以随便的使用"+"。显式的调用效率会更高，使用`stringbuilder.append(string)`来进行拼接。同时StringBuilder还提供了丰富的处理字符串的方法。
* 当想打印对象的内存地址的时候，或许会考虑使用this。比如

```
@Override
	public String toString() {
		return "InfiniteRecursion address :"+this;
	}
```
这时候会出现一大堆的异常信息。这是因为发生了无意识的递归。在使用"+"this的时候，编译器尝试把this转换为一个字符串，就又去调用了tostring方法。所以正确的方法是调用`super.tostring()`

###String对象有一下基本的方法，
  * `equalsIgnoreCase()`  忽略大小写
  * `startswith(),endswith()`  起始和结束的string
  
###String的格式化输出

Formatter类，可以看做是一个翻译器，它将你的格式化字符串与数据翻译成需要的结果。
格式化说明符：

    %[argument_index$][flags][width][.precision]conversion
    
width是指定一个域的最小尺寸，Formatter对象通过添加空格来保证最小尺寸。默认是右对齐，可以使用-来进行左对齐。precision的来指定最大的尺寸，可以用于浮点数取位。无法用于整数，会引发异常。例如：
```
s=String.format("%-10.2f %-5s", 1.2334,"后面的");
System.out.println(s);
//结果
1.23       后面的  
```
###正则表达式
* ？表示可能有
* \d 表示一个数字
* + 表示有一个或多个
* \w 表示分隔单词
这个我也只是看了点简单的，后面复杂的也没去了解，一则因为目前开发中还用不到那么复杂的正则表达式，一般就是处理一下字符串。二则网上有很多在线生成表达式的网站。暂时先放着吧。

## 第十四章 类型信息

 - 运行时类型信息使得你可以在程序运行时发现和使用类型信息。
 - RTTI（Run-Time Type Identification）：就是在运行时识别一个对象的类型。
###Class对象
 - 类型信息在运行时的表达是由称作Class对象的特殊对象完成的。所有的类都是在对其第一次使用时，动态加载带JVM中的。当程序创建第一个对类的静态成员的引用时，就会加载这个类，这也说明了构造函数是类的静态方法。
 - `Class.forName("")`这个方法是class类（所有class对象都属于这个类）的一个static成员。这个是获取class对象的引用的一种方式。如果类没有被加载就去加载它。
 - `Class`的`newInstance()`方法是实现"虚拟构造器"的一种途径，"我不知道你的具体类型，但是无论如何要正确的创建自己"。
###类字面常量
 - 也是用来生成对class对象的引用，`Toy.class`。这个不仅简单，而且更安全，编译期就会收到检查。但是他不会自动初始化该class对象。
####泛化的class引用
 - 类似```Class<?> intclass = int.class ```,向class引用添加泛型语法的原因仅仅是为了提供编译期类型检查。
####类型转换前先做检查
已知的RTTI形式包括
 1. 传统的类型转换。
 2. 代表对象的类型的class对象。
 3. 还有就是使用`instanceof`，告诉我们对象是不是某个特定类型的实例。

###反射：运行时的类信息
这个关于反射的后面写了一篇专门的，[java反射](http://70kg.info/2015/09/14/java%E5%8F%8D%E5%B0%84/)
###动态代理
之前也过代理模式，去那里看吧。[代理模式](http://70kg.info/2015/08/04/%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F/)
###空对象
这个不知道怎么写了。。
###接口和类型信息
这个怎么说呢，还是在讲反射的应用，还有接口，类转型的问题。

##第十五章 泛型

 - 一般的类和方法，只能使用具体的类型：要么是基本类型，要么是自定义的类。如果要编写可以应用于多种类型的代码，这种刻板的限制对代码的束缚就会很大。所以可以使用泛型，这个意思就是适用于许许多多的类型。
###简单泛型
引入泛型的最引人注目的原因就是创造容器类。
这是一个简答的泛型例子：
```
public class Holder<T> {
	private T a;
	public Holder(T a){this.a = a;}
	public T get(){return a;}
	public void set(T a){this.a = a;}

	public static void main(String[] args) {
		Holder<Mobile> holder = new Holder<Mobile>(new Mobile());
		holder.get();
		holder.set(new Mobile());
	}
}
class Mobile{}
```
####元组

 - return语句只能返回一个对象，当我们需要返回多个对象的时候，可以创建一个对象，让他来持有想要返回的多个对象，这就是元组的概念。我们可以使用泛型来优化使用。
```
public class Tuple<A,B> {
	public final A first;
	public final B second;
	public Tuple(A a,B b ){
		this.first = a;
		this.second= b;
	}
	public String toString(){
		return first+","+second;
	}
}
```
在上面的程序中，`first`和`second`都是public，理论上应该是private的，然后通过get和set方法去访问他们的值，但是这里使用了final。这样就只允许外界去读取而不允许去改变值。同时我们还可以使用继承去拓展元组，让它去支持更多的参数。
要使用元组也是很方便：
```
	static Tuple<String,Integer> f(){
		return new Tuple<String, Integer>("hahah", 12);
	}
	public static void main(String[] args) {
		System.out.println(f());
	
```
###泛型接口

 - 泛型可以应用于接口，例如生成器，这样一种专门负责创建对象的类，实现上实际是工厂方法设计模式的一种应用。例如
```
public interface Generator<T> {
	T next();
}
```
下面演示一下如何使用
```
public class CoffeeGenerator implements Generator<Coffee>,Iterable<Coffee>{

	private Class[] types = {Mocha.class,Coffee.class,Mocha.class};
	private static Random random = new Random(47);
	private int size = 0;
	public CoffeeGenerator(){}
	public CoffeeGenerator(int size){this.size= size;}
	public Iterator<Coffee> iterator() {
		return new CoffeeIterator();
	}
	public void forEach(Consumer<? super Coffee> action) {
	}

	public Spliterator<Coffee> spliterator() {
		return null;
	}

	public Coffee next(){
		try {
			return (Coffee) types[random.nextInt(types.length)].newInstance();
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}

	class CoffeeIterator implements Iterator<Coffee>{
		int count = size;

		public boolean hasNext() {
			return count>0;
		}

		public Coffee next() {
			count--;
			return CoffeeGenerator.this.next();
		}

		public void remove() {
			throw new UnsupportedOperationException();
		}

		public void forEachRemaining(Consumer<? super Coffee> action) {

		}

	}
	public static void main(String[] args) {
		CoffeeGenerator gen = new CoffeeGenerator();
		for(int i = 0;i<5;i++){
			System.out.println(gen.next());
		}
		for(Coffee c:new CoffeeGenerator(5)){
			System.out.println(c);
		}
	}

}
```
###泛型方法

 - 泛型方法使得方法可以独立于类而产生变化：无论何时，只要你能做到，就应该尽量使用泛型方法。例子：
```
public class GeneircMethods {
	public <T> void f(T x){
		System.out.println(x.getClass().getSimpleName());
	}
	public static void main(String[] args) {
		GeneircMethods m = new GeneircMethods();
		m.f("");
		m.f(1);
		m.f(1.0);
		m.f(m);
	}
}
//输出
String
Integer
Double
GeneircMethods

```
泛型是不支持基本类型作为类型参数的 ，但是由于JAVA的自动打包机制，我们并不需要太多担心。

 - 可变参数和泛型方法可以很好的共存：
```
public static <T> List<T> makelist(T...args){
		List<T> result = new ArrayList<T>();
		for(T item: args){
			result.add(item);
		}
		return result;
	}
```
####用于Generator的泛型方法
 - 使用生成器，可以很方便的填充一个Collection。
```
public class Generators {
	public static <T> Collection<T> fill(Collection<T> coll,Generator<T> gen,int n){
		for(int i = 0;i<n;i++){
			coll.add(gen.next());
		}
		return coll;
	}

	public static void main(String[] args) {
		Collection<Coffee> coffee= fill(new ArrayList<Coffee>(), new CoffeeGenerator(), 4);
		for(Coffee c:coffee){
			System.out.println(c);
		}
	}
}
```
####一个通用的Generator
```
public class BaseGenerator<T> implements Generator<T>{
	private Class<T> type;
	public BaseGenerator(Class<T> type){this.type = type;}

	public T next() {
		try {
			return type.newInstance();
		} catch (Exception e) {
			throw new RuntimeException();
		} 
	}
	public static <T> Generator<T> create(Class<T> type){
		return new BaseGenerator<T>(type);
	}
}
```
要使用这个生成器，这个类必须声明为public和必须有默认的构造函数。例如使用：
```
public class CountedObject {
	private static long count = 0;
	private final long id = count++;
	private long id(){return id;}
	public String toString(){return "CountedObject"+id;}
}
//---
public class BaseGeneratorDome {
	public static void main(String[] args) {
		Generator<CountedObject> gen = BaseGenerator.create(CountedObject.class);
		for(int i = 0;i<5;i++){
			System.out.println(gen.next());
		}
	}
}
```
同时我又改造了一下，使他可以支持有参构造函数😁：
```
public class BaseGenerator<T> implements Generator<T>{
	private Class<T> type;
	private Object[] a;
	public BaseGenerator(Class<T> type){this.type = type;}
	public BaseGenerator(Class<T> type,Object[] a){this.type = type;this.a= a;;}

	public T next() {
		try {
			return type.newInstance();
		} catch (Exception e) {
			throw new RuntimeException();
		} 
	}
	public T next(Class<?>... agrs) {
		try {
			Constructor c = type.getDeclaredConstructor(agrs);
			c.setAccessible(true); 
			return (T) c.newInstance(a);
		}catch (Exception e) {
			throw new RuntimeException();
		} 
	}
	
	public static <T> Generator<T> create(Class<T> type){
		return new BaseGenerator<T>(type);
	}
	public static <T> Generator<T> create(Class<T> type,Object[] a){
		return new BaseGenerator<T>(type,a);
	}
}
//---使用---
Generator<CountedObject> gen1 = BaseGenerator.create(CountedObject.class,new Object[]{5,6});
		gen1.next(new Class[]{int.class,int.class});
	
```
