# Java语言基础

## 一、基本数据类型

### Ⅰ、数据类型

#### 1、整型

| 类型  | 存储需求 |
| ----- | -------- |
| int   | 4字节    |
| short | 2字节    |
| long  | 8字节    |
| byte  | 1字节    |

注意事项：

1. 如4字节，最高位表示的是2³¹而不是2³²
2. Java用补码来表示数据，在后面讲移位操作时要用到这个前置知识。其中正数的补码是他本身，而负数的补码就是除符号位外，其他二进制位取反加1

#### 2、浮点类型

| 类型   | 存储需求                                                     |
| ------ | ------------------------------------------------------------ |
| float  | 4字节，其中1位作为符号位，其中8位是指数位(用移码表示的)，23位是尾数位 |
| double | 8字节，其中1位作为符号位，其中11位是指数位，52位是尾数位     |

##### ①、十进制转二进制

对十进制数与2求余，每次得出来的余数就是要的二进制位，这个循环直到被除数等于0位置。

##### ②、八进制转十进制

如72，那就是`2*8⁰+7*8¹`

##### ③、十进制小数转二进制

对小数乘2，去整数部分位二进制位，如果乘得的结果大于1，则使得整数位为0继续小数部分的2乘；如果等于1则计算结束。

如0.9转换成二进制：

0.9*2=1.8.....取整1

0.8*2=1.6…....取整1

0.6*2=1.2.......取整1

0.2*2=0.4........取整0

0.4*2＝0.8...取整0

0.8*2=1.6....取整1

····

最后结果就是0.111001

##### ④、计算机对浮点数的表示形式

计算机采用科学计数法的形式对浮点数进行表示。

如十进制数1.02×10²，1.02代表的则是尾数，2则代表指数位。

下面以8.25做例子：

1. 经过转换得出二进制科学计数法：1.00001×2³
2. 由于尾数的第一位肯定是1（如果原来是0，如0.001，则要表示为1×2^(-3)），所以舍掉这一位，只保存小数点后面的位数00001作为尾数部分
3. 取3作为阶码(指数位),还要涉及一些移码的转换

##### ⑤、判断浮点数相等

先来说一个老是出的问题：`0.1f==0.1d`和`1.0f==1.0d`。答案是什么？前者是false而后者是true。由于java对不同基本类型的二元操作会触发宽化类型机制，所以这里的0.1f将会变成0.1d。但是根据上面的知识我们知道，计算机以二进制位来保存数据，算一下0.1能够完全转化为二进制位吗？显然不能，没有这个2的倍数，所以一旦宽化，宽化后的数据有精度缺失，因为宽化就是对多出来的那几位写0而已，所以就丢失了这部分精度，所以就不会相等。而1.0是可以被完全二进制位表示并且不存在float精度问题的，所以宽化才没有影响

1. 如果不需要完全的值相等，允许误差的话，那就相减计算误差值
2. 如果要完全相等，就使用`Double.toString()`将其转换成字符串比较
3. 如果给的是浮点字符串(Java的浮点数没有能力表示太高精度),那么就构造`BigDecimal`对象,使用它的equals比较,其内部会分别比较两个对象的精度和值是否相等,并且`BigDecimal`对象可以表示任意精度的浮点数。

当然，在讨论这个问题之前就肯定先要确定输入，如果输入的是double，那直接等于也没毛病，因为在浮点数赋值给double引用的时候就已经舍弃了部分精度了，所以这不是我判断相等方法的锅；而如果要自己处理输入，就尽量使用字符串构造`BigDecimal`

#### 3、char类型

| 类型 | 存储需求                   |
| ---- | -------------------------- |
| char | 2字节，根据Unicode字符判断 |

#### 4、boolean类型

| 类型    | 存储需求 |
| ------- | -------- |
| boolean | 1字节    |

### Ⅱ、数据类型之间的转换

当有两个数值进行二元操作或者是方法调用作为参数传入（包括==，赋值不算二元操作，也就是说不能将byte赋值去int）时，虚拟机先要将两个操作数转换为同一个类型，再进行计算。

若两操作数有一个是double类型，则另一个操作数就会转换为double类型。

否则，若两操作数有一个是float类型，则另一个操作数就会转换为float类型。

否则，若两操作数有一个是long类型，则另一个操作数就会转换为long类型。

否则，两个操作数都将被转换为int类型。

### Ⅲ、强制类型转换

通过上面的数据类型自动转换可以发现，自动转换都是宽化操作。若是要窄化操作就需要手动进行强制类型转换，因为这有可能丢失部分数据。并且long==int是肯定返回false的。

### Ⅳ、位运算符

七种运算符：

1. `&`：与运算
2. `|`：或运算
3. `^`：异或运算
4. `~`：同或运算

Core Java有一个挺好的例子，可以拿来当作算法思路用。

```java
int fourthBitFormRight=(n&0b1000)/0b1000;
```

这就是使用掩码技术，获得某个整型数值的某位二进制位。`(n&0b1000)`将其他位掩盖掉只保留从右数第四位的数据,再相除就能够判断0还是1。

还有三种位运算符如下：

5.` >> `和`<<`：带符号右移和带符号左移。在前面说过，Java用补码来表示数据，正数的补码是其本身，显现出来的符号位就是0；而负数的补码是其正数表示的取反加一，显现出来的符号位就是1。所以带符号的意思就是移动过程中对空出来的位置以其符号位补全，正数用0，而负数用1。

6、`>>>`：无符号右移（不存在无符号左移）。无论正负，该运算符都用0来填充高位，所以正数的`>>`和`>>>`得到的结果相同,而负数的符号位就会变成0从而变成一个极大的正数。

### Ⅴ、溢出

不知道之前是没想过这个问题还是忘记了，真正用起来的时候发现自己像个弱智，遂作笔记。

看一个现象：

> long a = 2⁶⁴-1 = 9223372036854775807
>
> a<<1 = -2 ①
>
> a+1 = -9223372036854775808 ②

java用补码来表示数值，因为用补码表示可以使得符号位和数值域统一处理。

先来解释①：a此时在java中是表示成`0111······（省略60位1）`，正数的补码与原码相同，这里没有异议。当进行左移时，此时a就变成了`1111（省略59位1）0`了。此时符号位变成了负数，因此此时的原码的计算变成了除符号位外，其他位取反+1，也就是原码变成了`1000(省略58位0)10`,就是-2了。

②:有没有想过为什么byte范围是[-128~127],因为此时`10000000`也能表示成一个数值也就是-128,而`00000000`那就只能表示0了

### Ⅵ、分支语句

```java
switch(变量){
case 变量值1:
    //;
    break;
case 变量值2:
    //...;
    break;
  ...
default:
    //...;
    break;
}
```

swtich()变量类型只能是**int、short、char、byte和enum**类型（**JDK 1.7 之后，类型也可以是String了**）。

其中case的变量值只能是常量或者是表达式，但一定要符合正确的类型。不允许变量放在case中，即使已赋值。

注意一旦进入了case,那么无论如何都不会进入default。虽然如此,但是如果不加break还是有可能触发多个case分支的。

与if···else比较

1. 当分支较多时，当时用switch的效率是很高的。因为switch是随机访问的，就是确定了选择值之后直接跳转到那个特定的分支，但是if。。else是遍历所以得可能值，知道找到符合条件的分支。如此看来，switch的效率确实比ifelse要高的多。
2. switch…case只能处理case为常量的情况，对非常量的情况是无能为力的。例如 if (a > 1 && a < 100)，是无法使用switch…case来处理的。所以，switch只能是在常量选择分支时比ifelse效率高，但是ifelse能应用于更多的场合，ifelse比较灵活。

## 二、继承

### Ⅰ、成员变量

在编写继承关系时，子类会继承父类的成员变量。然而我们仍然可以在子类中声明一些**新的成员变量**，其中有一种特殊的情况就是，所**声明的成员变量**的名字和从父类**继承来的成员变量的名字相同**（声明的类型可以不同），在这种情况下，==子类就会隐藏所继承的成员变量，这就叫**成员变量的隐藏**==。需要注意的是，子类**继承自父类的方法**只能操作**子类继承和隐藏的成员变量**。子类**新定义的方法**可以操作子类继承和子类新声明的成员变量，但是**无法操作子类隐藏的成员变量**，如果非要操作隐藏的成员变量，则需要用到**关键字super**来访问被隐藏的成员变量。

还有一个结论就是：**成员变量并不具有多态性**。

```java
class Fu{
    int i = 10;
    public void say()
    {
        System.out.println("Fu");
    }
}
class Zi extends Fu{
    int i = 20;
    public void say()
    {
        System.out.println("Zi");
    }
}
class Test{
    public static void main(String[] args)
    {
        Fu test = new Zi();
        test.say();
        System.out.println(test.i);
    }
}
```

由于方法具有多态性,所以`say()`方法将在运行时动态绑定到它的实际类型`Zi`的方法中,毋庸置疑`test.say()`将输出`Zi`。而`test.i`呢?这个时候输出的值就是`Fu.i`也就是10了。所以**成员变量不具备多态性，通过引用变量来访问其包含的实例变量，系统总是试图访问它引用类型所定义的成员变量，而不是实际类型所定义的成员变量。**

==**static和成员变量的原理是一样的！！！**==

### Ⅱ、this和super关键字的作用

this的用途：

1. 引用隐式参数。其中在`test.say();`调用中,`test`称之为隐式参数,而`say()`方法中的方法参数则称之为显式参数。
2. 调用该类其他的构造器。

super的用途：

1. 调用超类的方法或引用超类的成员。
2. 调用超类的构造器。

### Ⅲ、Java面向对象的特性

* 封装
  * 封装的含义就是将数据和行为组合在一个类中，并对对象的使用者隐藏数据的实现方式。封装给对象赋予了“黑盒”的特征，这是提高重用性和可靠性的关键。
* 多态
  * 所谓多态就是指程序中定义的引用变量所指向的具体类型和通过该引用变量发出的方法调用在编程时并不确定，而是在程序运行期间才确定，即一个引用变量到底会指向哪个类的实例对象，该引用变量发出的方法调用到底是哪个类中实现的方法，必须在由程序运行期间才能决定。
* 继承
  * 继承是已存在类作为基础建立新类的技术，子类可以增加新的数据和功能，同时也可以使用父类的功能。通过继承关系可以将比较通用的功能放在父类中，以达到各个子类的复用目的。

### Ⅳ、阻止继承

如果某个类不允许子类覆盖它的某个方法，则可以用`final`修饰符声明该方法;而若某个类不允许被继承,则可以用`final`修饰符声明这个类，这样若有子类`extends`该类时则会出现编译错误(**注意:final类中的所有方法将自动的成为final方法,但它的域是不受影响的**)。

### Ⅴ、访问修饰符

1. 仅对本类可见——`private`

2. 对所有类可见——`public`

3. 对本包和所有子类可见——`protected`

4. 对本包可见——缺省

### Ⅵ、Object：所有类的超类

####  1、equals方法

Java语言规范要求equals具有下面的特性：

1. 自反性。对于任何非空引用`x.equals(x)`应该返回true
2. 对称性。对于任何引用x和y，当且仅当`y.equals(x)`返回true,`x.equals(y)`也应该返回true
3. 传递性。对于任何引用x、y和z，如果`x.equals(y)`返回true，`y.equals(z)`也返回true，那么`x.equals(z)`也应该返回true。
4. 一致性。如果x和y引用的对象没有发生变化，反复调用`x.equals(y)`应该返回同样的结果。
5. 对于任意非空引用x，`x.equals(null)`应该返回false。

接下来我来讨论一下隐式和显式参数不属于同一个类,equals方法该如何处理。

由于对称性的存在，`Employee.equals(Manager)`的返回结果必须和`Manager.equals(Employee)`相同（Manager是Employee的子类）。 这就引出一个该使用`instanceof`还是`getClass()`来比较两个类的问题。使用`getClass()`比较的例子如下:

```java
class Employee
{
	
	public boolean equals(Object otherObject)
	{
		
		if (this == otherObject) return true;
		
	
		if (otherObject == null) return false;
		
		//这里使用了getClass()来判断两个对象是否属于同一个类
		if (getClass() != otherObject.getClass())
			return false;
		
		
		Employee other = (Employee)otherObject;
		
	
		return name.equals(other.name);
			&& salary == other.salary;
			&& hireDay.equals(other.hireDay);
	}
}
class Manager extends Employee
{
	
	public boolean equals(Object otherObject)
	{
        //先使用父类的equals比较
		if(!super.equals(otherObject)) return false;
		//然后比较子类Manager的实例域
		Manager other = (Manager)otherObject;
		return bonus = other.bonus;
	}
}
```

使用`instanceof`的例子如下所示:

```java
class Employee
{
	
	public boolean equals(Object otherObject)
	{
		
		if (this == otherObject) return true;
		
	
		if (otherObject == null) return false;
		
		
		if (otherObject instanceof Employee)
			return false;
		
		
		Employee other = (Employee)otherObject;
		
	
		return name.equals(other.name);
			&& salary == other.salary;
			&& hireDay.equals(other.hireDay);
	}
}
class Manager extends Employee
{
	
	public boolean equals(Object otherObject)
	{
        //只用父类equals进行比较
		return super.equals(otherObject));
		
	}
}
```

可能你会产生疑问:为什么`instanceof`不比较自己的bouns成员呢?因为对称性。如果`Employee.equals(Manager)`返回true，此时`Employee.equals()`方法比较的只有Employee类内的成员而没有比较Manager的成员,反过来由于`Manager.equals(Employee)`也要返回true,那么`Manager.equals()`方法就不可能比较Manager的成员了,因为一旦比较返回的就是false不满足对称性（ClassCastException）。这就是使用`instanceof`的缺陷,由于要使子类对象和父类对象可比较且满足对称性,那么就只能舍弃掉子类对象的成员了。

而使用`getClass()`时为什么就可以比较bouns成员呢?同样因为对称性。因为使用了两者的`getClass()`来比较,所以`Employee.equals(Manager)`这种父类对象和子类对象必定是返回false的,因为这两个并不属于一个Class对象。`getClass()`就这样规避了父类对象和子类对象的比较,它要求两者都属于同一个类的时候这两个对象才是可比较的，这样同样也能满足对称性。

所以为了满足对称性，业务中就要根据实际情况来选择使用哪种判断：

* 如果子类需要拥有自己的相等概念，则对称性需求将要强制采用`getClass()`进行检测。
* 如果有超类决定相等的概念，那么就可以使用`instanceof`进行检测(如在超类中添加成员id),这样就可以在不同的子类对象或父子类对象之间进行比较了

下面给出equals方法的编写规范：

1.  显示参数命名为otherObject，稍后需要将它转换成另一个叫做other的变量。

2. 检测this与otherObject是否引用同一个对象：
	`if (this == otherObject) return true;`
	这条语句只是一个优化。实际上，这是一种经常采用的形式。因为计算这个等式要比一个一个地比较类中的域所付出的代价小得多。

3.  检测otherObject是否为null，如果为null，返回false、这项检测是很必要的。

    ` if (otherObject == null) return false;`

4. 比较this与otherObject是否属于同一个类。

   1. 如果equals的语义在每个子类中有所改变，就使用`getClass()`检测：

       ` if (getClass() != otherObject.getClass()) return false;`

   2. 如果所有的子类都拥有统一的语义，就使用`instanceof`检测：

      `  if(!(otherObject instanceof ClassName)) return false;`

5. 将otherObject转换为相应的类类型变量：

 	 `ClassName other = (ClassName) otherObject;`

6. 现在开始对所有需要比较的域进行比较了。使用==比较基本类型域，使用equals比较对象域。如果所有的域都匹配，就需要返回true；否则返回false。

	```java
 return field1 == other.field1
		 //⭐注意：用Objects.equals比较的原因是防止this.field2对象引用为null,这样就不会出现NullPointerExecption了
	     &&Objects.equals(field2,other.field2)
	
	     &&...;
	```
	
	  如果在子类中重新定义equals， 就要在其中包含调用`super.equals(other)`。

#### 2、hashcode方法

没什么好注意的。用`Objects.hash()`重写就是了,其他去看`hashcode()与equals().md`。

#### 3、toString方法

就更没什么好说的了。

#### 4、clone方法

看一下Object类中的`clone()`方法

```java
protected native Object clone() throws CloneNotSupportedException;
```

可以看到它使用了`protected`修饰符,所以如果子类需要提供`clone()`方法则必须:

1. 实现`Cloneable`接口,不实现这个接口会报错
2. 根据所需浅拷贝还是深拷贝重新定义`clone()`方法,并指定`public`修饰符,这样才可以给外部使用。

例子如下：

```java
class Employee implements Cloneable
{
    //可协变的返回类型
    @Override
    public Employee clone()
    {
        Employee cloned=(Employee)super.clone();
        //开始拷贝成员，这就是深拷贝
        cloned.hireDay=(Date)hireDay.clone();
    }
}
```

顺便说一下浅拷贝和深拷贝的定义：

* 浅拷贝：对基本数据类型进行值传递，对引用数据类型进行地址的拷贝
* 深拷贝：对基本数据类型进行值传递，对引用数据类型，创建一个新的对象，并复制其内容

### Ⅶ、抽象类

1. 抽象类可以包含具体数据和具体方法
2. 包含一个或多个抽象方法的类本身必须被声明为抽象的
3. 抽象类不能被实例化。也就是说可以定义一个抽象类的对象变量，但是他只能引用非抽象子类的对象。

### Ⅷ、接口

#### 1、基本特性

1. 接口中所有方法自动地属于`public abstract`
2. 接口不能有实例域，接口中的域将被自动设为`public static final`
3. 不能使用new实例化接口,接口并不能被实例化
4. 每个类可以实现多个接口
5. 接口可以被扩展

#### 2、Jdk1.8新特性

##### ①、静态方法

在Jdk1.8中允许在接口中增加静态方法。

##### ②、默认方法

在Jdk1.8中可以为接口方法提供一个默认实现。必须用default修饰符标记这个方法。

```java
public interface Comparable<T>
{
	default int compareTo(T other)
	{
        return 0;
	}
}
```

默认方法可以不被实现类实现,也就是说可以被当成父类的方法直接调用。

既然默认方法可以被当成父类的方法来调用，由于实现类可以实现多个接口，那么多个接口中的相同参数列表默认方法该如何选择？这时候就要讨论如何解决默认方法冲突。

* 超类/子接口优先。如果一个类继承了一个父类且这个父类实现了和当前类一样的接口，且父类实现了该接口的默认方法。若此时子类调用该方法，Java在父类的实现和接口的默认方法中将会会选择执行前者。
* 接口冲突。如果一个接口提供了一个默认方法，另一个接口也提供了一个相同参数列表的相同方法，此时实现这两个接口的实现类就必须覆盖这个方法来解决冲突。

### Ⅸ、抽象类与接口的区别

1. 接口中所有方法自动地属于public abstract，抽象类则不一定
2. Jdk1.8之前接口不能由方法实现，而抽象类中可以有抽象方法也可以有非抽象方法。
3. 接口不能有实例域，接口中的域将被自动设为`public static final`。而抽象类则可以有实例域。
4. 不能使用new实例化接口,接口并不能被实例化
5. 一个类只能继承一个抽象类，但可以实现多个接口。接口本身可以用extends扩展。
6. 抽象类继承自Object类，而interface不继承自Object类


### Ⅹ、方法参数

按值调用表示方法接受的是调用者提供的值，按引用调用表示方法接受的是调用者提供的变量地址。一个方法可以修改传递引用对应的变量值，而不能修改传递值调用所对应的变量值。Java总是采用按值调用，也就是说方法得到的是所有参数值的一个拷贝，方法不能修改传递给它的任何参数的变量内容。

## 三、异常

假设一个Java程序运行期间出现了错误。这个错误可能是由于文件包含错误信息，亦或者是程序逻辑错误等。异常处理机制就是要达到一下两个目的：

* 将出现异常的程序返回到一种安全状态，并能够让用户执行一些其他命令
* 或者允许用户保存所有操作结果，以妥善的方式终止程序

### Ⅰ、异常分类

异常对象都是派生与`Throwable`类的一个实例。

![](E:\Typora\MyNote\resources\Java\基础\Throwable.png)

`Error`类层次结构描述了Java运行时系统内部错误和资源耗尽错误,如`OutOfMemoryError`、`StackOverFlowError`，出现了这样的内部错误除了通告给用户并尽力使程序安全终止外也无能为力了。**还有一个比较重要的Error就是`NoClassDefFoundError`,表示当JVM在加载一个类的时候，如果这个类在编译时是可用的，但是在运行时找不到这个类的定义的时候(删掉生成的class文件)，JVM就会抛出一个NoClassDefFoundError错误。**

`Exception`层次结构是设计程序时需要关注的地方,该层次结构又分为两个分支:一个分支派生于`RuntimeException`,另一个分支则包含其他异常。两个分支的重要区别是：由程序员编写的程序错误导致的异常属于`RuntimeException`；而程序本身没有问题，但由于像I/O错误这类问题导致的异常属于其他异常，**比较有代表性的就是`ClassNotFoundException`,表示当应用程序运行的过程中尝试使用类加载器去加载Class文件的时候，如果没有在classpath中查找到指定的类，就会抛出`ClassNotFoundException`。一般情况下，当我们使用`Class.forName()`或者`ClassLoader.loadClass`以及使用`ClassLoader.findSystemClass()`在运行时加载类的时候，如果类没有被找到，那么就会导致JVM抛出`ClassNotFoundException`。**

Java语言规范将派生于`Error`类或`RuntimeException`类的所有异常称为非受查异常，所有其他的异常称为受查异常。

### Ⅱ、异常处理

对于受查异常的处理方式要不就抛出要不就捕获（总之强制要求程序内必须处理,或向上传递或捕获），不然编译器会发出错误信息；而对于非受查异常要么不可控制要么就应该避免发生，所以非受查异常一般都不用显式处理，可以由Java内部直接交由虚拟机显示错误。

#### ①、异常抛出

对于异常抛出有两个关键字：`throw`与`throws`。

当在程序运行过程中发现错误，此时程序需要声明抛出一个(受查/非受查)异常时，也就是使用在方法中使用`throw new EOFException();`

当方法调用内部抛出一个受查异常时,若当前方法并不知道如何处理该受查异常,那么就应该继续向上传递该异常,也就是在方法头部标识`throws EOFException`

#### ②、异常捕获

如果某个异常发生后没有任何地方进行捕获（也就是一直向上传递），那么程序就会终止执行。如果想让程序继续执行则必须用`try···catch···finally`对异常进行捕获。

如果在try块中任何代码抛出了一个在catch子句中说明的异常类，则程序将跳过try块中剩余的代码并执行catch自居中的处理器代码，最后执行finally块逻辑。catch块则被称为异常处理器。当执行完catch块代码后程序就会继续执行。

在catch块中除了直接控制台输出错误或记录日志外，还可以再次抛出异常与异常链。

```java
try
{
    //access the database
}
catch(SQLException)
{
    Throwable se=new ServletException("database error");
    se.initCause(e);
    throw se;
}
```

这样做的目的是改变异常的类型和信息描述,使得用户能更好的分析异常并定位异常。（其中所有`Throwable`子类都可以被捕获）

## 四、反射

只要有类名就可以获取类的全部信息

JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。

要想解剖一个类,必须先要获取到该类的字节码文件对象。而解剖使用的就是Class类中的方法.所以先要获取到每一个字节码文件对应的Class类型的对象 。反射就是把java类中的各种成分映射成一个个的Java对象。

## 五、Java8新特性

### Ⅰ、Lambda表达式

Lambda表达式是用来代替匿名内部类写法的，它的出现让方法中实现接口的写法变得更加易读。Lamda表达式的使用场景是：**==对于只有一个抽象方法的接口==**，需要这种接口的对象时，就可以提供一个Lambda表达式（可以将Lambda表达式看作是匿名内部类的对象创建方式，就很容易理解了）。

#### 1、基本使用

Lambda的基本写法：`()->{}`。其中括号表示接口唯一抽象方法中的参数列表，花括号内就写方法的实现。

例：

```java
String []a={"aasda","asdads","a1d1d1"};
Arrays.sort(a, new Comparator<String>()
{
    @Override
    public int compare(String o1, String o2)
    {
        return o1.length()-o2.length();
    }
});
```

这是匿名内部类的写法，转换成Lambda表达式的写法如下：

```java
  String[] a = {"aasda", "asdads", "a1d1d1"};
        Arrays.sort(a, (String o1, String o2) ->
        {
            return o1.length() - o2.length();
        });
```

始终关注Lambda表达式代表接口对象就容易理解。

#### 2、特殊简写

1. 参数列表的类型可以省略，即

   ```JAVA
   (o1, o2) ->
           {
               return o1.length() - o2.length();
           }
   ```

2. 若抽象方法只有一个参数，还可以直接省略小括号

   ```java
   interface Say
   {
       void say(String s);
   }
   s->System.out.println(s)；
   ```

3. 若方法体只有一行代码，无论是否需要返回值，都可以省略return和花括号

   ```sql
   (o1, o2) -> o1.length() - o2.length()
   ```

#### 3、方法引用

在某些情况下，要实现的抽象方法需要调用某个现成的方法**（也就是说Lambda表达式的实现只做一个传递参数值的作用）**，这种情况就要使用方法引用。

方法引用要用`::`操作符分割方法名与对象或类名。主要有三种情况：

1. `对象::实例方法名`
2. `类名::静态方法名`
3. `类名::实例方法名`:这种是特殊情况。主要用在抽象方法的第一个参数是实例方法的调用者 情况中。

例:

```java
Arrays.sort(a, (o1, o2) -> o1.compareToIgnoreCase(o2));
```

可以简写成：

```java
Arrays.sort(a, String::compareToIgnoreCase);
```

#### 4、构造器引用

构造器引用和方法引用有些类似，只不过格式变成了`类名::new`。

例：

```java
interface PersonFactory
{
    Person getPerson(String s);
}
class Person
{
    public Person()
    {
    }

    public Person(String s, int a)
    {
    }

    public Person(String s)
    {
    }

    public Person(int a)
    {
    }

  
}
```

```java
PersonFactory p=Person::new;
```

还有一个要注意的问题就是构造器的选取是根据抽象方法的参数做分派的,所以上面的例子将会使用Person的`String s`构造器。

#### 5、==变量作用域==

Lambda表达式（或匿名内部类）中可以访问外围方法或类中的变量，但是有一个极其重要的限制，那就是Lambda表达式中只能引用值不会改变的变量（1.8只要不改变变量即可，而1.8之前的版本则要显式添加final关键字才可编译通过），无论是Lambda表达式外修改或是表达式内修改都会出现编译错误。

原理：匿名内部类和Lambda表达式也属于一个类，编译完成后会生成一个Class文件，所以它的生命周期和外部类方法的生命周期并没有关系。然而表达式却要使用外部类方法的局部变量，在这种场景下，局部变量在外部类方法执行完毕后栈帧弹出，此时

#### 6、四大函数式接口

##### ①、`Consumer`接口

从字面意思上理解,消费者接口，意味着这个接口是用来消费入参给的值的。

* 函数式接口定义

  ```java
  @FunctionalInterface
  public interface Consumer<T> {
      void accept(T t);    
  }
  ```

  顾名思义的消费，将其想象成MQ，是没有返回值的。

* 示例：

  ```java
   Stream.of("aaa", "bbb", "ddd", "ccc", "fff").forEach(s -> System.out.println(s.toUpperCase()));
  ```

  分析:`forEach`内所需的参数就是`Consumer`,此时的消费者的功能就是在遍历中输出流的大写字母

##### ②、`Supplier `接口

同样按字面意思理解,供给型接口,意味着这个接口作为一个容器提供数据。

* 函数式接口定义

  ```java
  @FunctionalInterface
  public interface Supplier<T> {
      T get();
  }
  ```

* 示例:

  ```java
  supplier = () -> new Random().nextInt();
  System.out.println(supplier.get());
  ```

  分析:从容器中获取一个随机数

##### ③、`Predicate `接口

`Predicate` 接口是一个谓词型接口，其实，这个就是一个返回值为`boolean`类型的判断接口

* 函数式接口定义

  ```java
  @FunctionalInterface
  public interface Predicate<T> {
      boolean test(T t);
  }
  ```

* 示例:

  ```java
  predicate = (t) -> t > 5;
  System.out.println(predicate.test(1));
  ```

  分析:判断数字是否大于5

##### ④、`Function`接口

这是最难理解也是最常用的接口,他的作用就是对入参进行转换,将一种类型的对象转换成另外一种类型的对象

* 函数式接口定义

  ```java
  @FunctionalInterface
  public interface Function<T, R> {
      R apply(T t);
  }
  ```

* 示例:

  ```java
  Function<String,Integer>func=String::length;
  System.out.println(func.apply("123123"));
  ```

  分析:将输入的字符串转换成其长度输出

### 二、`Stream`与`Optional`

先来说说我对这两个类的理解。这两个类其实非常的相似，其中`Optional`类是基于对象的,而`Stream`类则是基于集合的,他们的作用都是以比较优雅的链式调用代替繁杂的`if···else···`,以`Optional`类为例子,他的功能就有以链式调用代替判空并返回默认值、设置对象成员变量的条件过滤并返回等等；而对于`Stream`类来说,他的功能就是根据条件过滤集合元素等等。

#### Ⅰ、`Optional`

##### 1、判空场景

要使用`Optional`来避免`NullPointerException`,就要先构造`Optional`对象。

有三种构造方法：

```java
Optional<User>opt1=Optional.empty();
Optional<User> opt2 = Optional.of(user);
Optional<User> opt3 = Optional.ofNullable(user);

User user=opt1.get();
```

其中第一种方法则是相当于传入一个null,而第二种方法如果传null进去则会抛出`NullPointerException`,第三种方法则允许传null或者对象进去,所以一般是用`Optional.ofNullable()`。

当然，`Optional`只是一个具有判空API的类，还需要一个将原对象获取出来的方法，那就是`get()`。然而如果`Optional`类内的`value`是null,则调用`get()`的时候依然会抛出`NullPointerException`,**那么如何才能进行判空呢?**

```java
public boolean isPresent() {
        return value != null;
    }

 public void ifPresent(Consumer<? super T> consumer) {
        if (value != null)
            consumer.accept(value);
    }
```

两种方法:第一种很简单,就是进行判空;而第二种就说得上是代替了`if···else···`的一个接口了:如果值不为空,则调用消费者接口方法以消费该消息。

##### 2、返回默认值场景

三种方法：

```java
public T orElse(T other) {
        return value != null ? value : other;
    }
public T orElseGet(Supplier<? extends T> other) {
        return value != null ? value : other.get();
    }

//示例：
        User user = null;
        User result = Optional.ofNullable(user).orElse(new User("WWY", 1));

      	User result = Optional.ofNullable(user).orElseGet(()->new User("WWY", 1));

		User result = Optional.ofNullable(user)
      			.orElseThrow( () -> new IllegalArgumentException());
```

对于`orElse()`来说,其逻辑就是如果为空,则返回给定入参默认值;而对于`orElseGet()`来说,其逻辑就是如果为空,则调用供给接口获取值。而这两者最大的不同如果值为不为空，`orElse()`会创建默认对象而`orElseGet()`不会创建默认对象,也就是饿汉式和懒汉式的问题。而至于该机制的原理，很明显一个传的是`new`方法一个传的是`Supplier`对象,`new`方法肯定会在`orElse()`调用前就执行了。

第三种方法就是抛出异常了

##### 3、转换值

```java
public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        return Optional.ofNullable(mapper.apply(value));
    }
}

//示例
String str=Optional.ofNullable(user).map(User::getName).orElse("null");
//相当于
if(user!=null)
{
    return user.getName()==null?"null":user.getName();
}
```

调用时传入一个转换接口，返回的将是转换之后的值的`Optional`。

##### 4、过滤值

`filter() `接受一个 Predicate 参数，返回测试结果为 true 的值。如果测试结果为 false，会返回一个空的 `Optional`。

```java
Optional.ofNullable(user).filter(user1->user1.getName().equals("WWY")).ifPresent(System.out::println);
```

相当于

```java
if(user!=null)
{
    String name=user.getName();
    if(name!=null)
    {
        if(name.equals("WWY"))
        {
            System.out.println(name);
        }
    }
}
```

#### Ⅱ、`Stream`

`Stream`提供的API与`Optional`有些类似.

##### 1、过滤集合值场景

###### ①、`filter()`

```java
Stream<T> filter(Predicate<? super T> predicate);
```

遍历流并根据谓词接口过滤数据

###### ②、`distinct()`

```java
Stream<T> distinct();
```

数据去重,底层调用的是`equals()`去判断

##### 2、输出集合元素的部分成员

###### ①、`map()`

```java
List<String> names = students.stream()
                            .filter(student -> "计算机科学".equals(student.getMajor()))
                            .map(Student::getName).collect(Collectors.toList());
```

`map()`的作用就是将流数据映射成某一类型的数据,上面的示例就是将`Stream<Student>`映射成了`Stream<String>`,最后调用`collect()`方法返回相应的集合对象

## 七、输入输出流

`java.io`包下的流分为两大类：字符流（`Writer`和`Reader`）与字节流（`OutputStream`和`InputStream`）。字节流的特点是支持单个字节读取或者是读取一个字节数组（不依赖于读单个字节的本地方法`read0()`（以`FileInputStream`为例）,而字符流特点是支持单个Unicode码元的读取（以`FileReader`为例）。

在讲四个主要接口之前，先来了解`File`类。File类是对文件系统中文件以及文件夹进行封装的对象，可以通过对象的思想来操作文件和文件夹，其保存了文件或目录的各种元数据信息，并提供了创建、删除文件和目录的方法。

### Ⅰ、`InputStream`

`InputStream`是一个抽象类,它提供了两个最重要的抽象方法给子类实现——`read()`和`read(byte[]b)`。其中`read()`用于读取一个字节，并将读取指针向后移动一位，若已经读取完毕`read()`就会返回-1，通常将该方法放在`while`循环中;而 则会将输入流中的数据复制到byte数组b中。

#### 1、`FileInputStream`

这是基本的`InputStream`实现类,通过在构造函数中传入`File`对象以读取文件的信息.

#### 2、`BufferedInputStream`

这是一个装饰器。它的原理就是在内部维护一个8K的字节数组缓冲区,每当调用其`read()`时就移动字节数组缓冲区的pos指针。当pos指针移动到了字节数组末尾，则`read()`方法会自动调用被装饰者`InputStream.read(cache)`方法进行一次操作系统IO读取字节数组以填充缓冲区的数据,当没有数据可读时返回-1。

该`InputStream`的优点是大大提高了读取的效率,如果单纯的用`FileInputStream.read()`,则要进行多次系统IO才会将数据读取完。而`BufferedInputStream`最主要的功能就是通过编码使得==缓冲区数据读取完后自动进行==**数组**读取系统IO，相比循环调用`FileInputStream.read()`提高了效率。

#### 3、`PipedInputStream`

管道字节输入流，它和PipedOutputStream一起使用，能实现**多线程间的半双工管道通信**

#### 4、`ObjectInputStream`

序列化对象输入流。

### Ⅱ、OutputStream

同样,`OutputStream`也是一个抽象类。他也有两个最重要的方法——`write(int)`写入字节与`write(byte[])`写入字节数组。

#### 1、`FileOutputStream`

和`FileInputStream`类似

#### 2、`BufferedInputStream`

这里就要注意一下。调用`BufferedInputStream.write()`并不会立刻进行系统IO,而是同样的将要写入的字节放入内部的缓冲区中。当调用的方法确定要进行系统IO写入文件时，调用`BufferedInputStream.flush()`将该缓冲区的数据通过`outputStream.write(byte[])`方法一次性写入文件,同样也提高了效率。

#### 3、`PipedOutputStream`

#### 4、`ObjectOutputStream`

### Ⅲ、Reader

Reader是处理字符流的一个接口，它同样提供了`read()`和`read(char[]c)`，然而这两个方法返回的都是char而不是byte。

#### 1、`FileReader`

以字符流的形式读取文件。下面来讨论一下它的原理。

`FileReader`继承自`InputStreamReader`,而`InputStreamReader`的大致运行机制其实还是通过`InputStream`读入字节,只不过在读入字节之后将字节根据选择的解码方式将字节流转换为字符流。而默认的`InputStreamReader`的解码方式则是utf-8，如果想要转换该解码方式则要用自定义的`InputStreamReader`来包装新建的`FileInputStream`，从而使用`InputStreamReader`来读取数据。

#### 2、`InputStreamReader`

装饰者,用于对读入的**字节流**根据**解码方式**解码成字符，是连接字节流和字符流的桥梁。其构造器如下

```java
 public InputStreamReader(InputStream in, Charset cs)
```

其Charset有以下几种:`StandardCharsets.UTF_8`、`StandardCharsets.ISO_8859_1`、`StandardCharsets.UTF_16BE`、`StandardCharsets.US_ASCII`、`StandardCharsets.UTF_16LE`、`StandardCharsets.UTF_16`。

#### 3、`BufferedReader`

与`BufferedInputStream`类似。

### Ⅳ、`Writer`

与Redaer相类似

## 八、序列化

当两个Java进程要通信，或需要将一个对象存储到磁盘时，就可以使用序列化机制。序列化机制就是将对象的数据序列化写入到输出流中，并在之后将其读回就可以直接反序列化为一个对象。

### Ⅰ、保存和加载序列化对象

最简单的序列化对象的方法只有三步：

1. 使需要序列化的对象和该对象的成员所属的类都实现`java.io.Serializable`接口
2. 使用`objectOutPutStream.writeObject(Object)`方法将所需序列化的对象保存到输出流中。
3. 使用`Employee e=(Employee)objectInPutStream.readObject()`方法,按照这些对象写出时的顺序将保存到流的对象反序列化回来。

在幕后,是`ObjectOutPutStream`在浏览对象的所有域,并对这些域的对象进行存储。但是`ObjectOutPutStream`是怎么处理多个对象引用着同一个对象（相同对象引用）这个关系的呢？这就要讨论到它的序列化算法。

`ObjectOutPutStream`的序列化算法：

1. `ObjectOutPutStream`对遇到的每一个  **对象引用** 都会在输出流中关联一个序列号
2. 对于每一个对象引用，在第一次遇到时，保存其对象的数据到输出流中（如此时的序列号是21）
3. 如果在之前某个对象已经被保存过，在之后如果再次遇到该对象引用，`ObjectOutPutStream`就会直接往输出流中写出 序列号21，不会再次存储该对象的数据。

`ObjectInPutStream`的反序列化算法：

1. 对于输入流中的对象，在第一次遇到其序列号时就创建这个对象，并使用流中的数据来对其初始化，然后记录该序列号与此对象的关联。
2. 在其后若遇到 序列号21的标志时，此时就获取 该序列号对应的已创建的对象，并使引用变量指向这个对象。

**注意,静态变量不会被序列化。**

### Ⅱ、修改默认的序列化机制

在上面说序列化对象的步骤时曾说过，所需序列化的对象与其成员全都要实现`Serializable`接口,那么如果刚好某个成员没有实现这个接口应该怎么办?又或者某些只对本地方法有意义的文件句柄整数值,这些整数值到另外一台服务器上是没有作用的,在反序列化时仍按照原机器的赋值还有可能会出错。所以这些对象是无法被序列化的。

Java提供了两个机制用来解决这个问题。

#### 1、`readObject()`和`writeObject()`方法

如果要修改域的序列化机制,就要在所需序列化的对象中写出这两个方法。在序列化过程中会调用`writeObject()`方法,而在反序列过程中会调用`readObject()`方法。

例如对于`Graph`类,`Point`成员无法被序列化。

```java
class Point
{
    private int x;
    private int y;

    public Point(int x, int y)
    {
        this.x = x;
        this.y = y;
    }
}
```

在`Graph`类中就要写出`readObject()`和`writeObject()`以自定义其序列化机制。

```java
class Graph implements java.io.Serializable
{

    public transient  Point point=new Point(1,3);

    //实现这两个方法之后，还是会自动序列化域，但是之后会调用这两个方法。
    //如Point没有实现序列化接口，那么就可以将其标记为transient使得不自动序列化，而是在
    //writeObject中又我们自己定义序列化逻辑
    private void readObject(ObjectInputStream in)throws IOException,ClassNotFoundException
    {
        in.defaultReadObject();
        int x=in.readInt();
        int y=in.readInt();
        point=new Point(x,y);
    }
    private void writeObject(ObjectOutputStream out)throws IOException,ClassNotFoundException
    {
        out.defaultWriteObject();
        out.writeInt(point.x);
        out.writeInt(point.y);

    }
}
```

首先需要用`transient`标识`Point`成员,这样`ObjectOutPutStream`在序列化时就会跳过该成员。然后在`readObject()`和`writeObject()`定义序列化和反序列化逻辑,并在`readObject()`方法中根据序列化的信息创建对象。

要注意的是，这两个方法并不是全盘负责序列化机制的，==`ObjectOutPutStream.writeObject()`仍然会自动扫描域并序列化域的对象，只不过那些特殊的无法被序列化的域被标记上了`transient`后`ObjectOutPutStream`就会跳过它，这就让我们有机会自定义这==些域的序列化。

#### 2、`Externalizable`接口

`Externalizable`有两个接口方法要实现。

```java
public interface Externalizable extends java.io.Serializable {

    void writeExternal(ObjectOutput out) throws IOException;

    void readExternal(ObjectInput in) throws IOException, ClassNotFoundException;
}
```

`writeExternal`和`readExternal`所需要写的逻辑大致相同。但是与`readObject()`和`writeObject()`最大的区别就是：`Externalizable`的两个方法需要负责整个对象包括超类所有域的序列化，在实现了`Externalizable`后自动序列化机制就已经失效了，也就是说就算在对象内定义`readObject()`和`writeObject()`也不会被调用了，因为他们属于自动序列化的一环。

### Ⅲ、安全的单例模式

这里先做一个结论:最安全的单例模式是枚举单例模式。为何他是最安全的？那首先就要说一下其他实现为什么都是不安全的。

#### 1、其他单例为何不安全

其一，其他的单例模式都使用了`private`来修饰构造器,但事实上反射机制的`cons.setAccessible(true)`是可以跳过这`private`修饰符直接实例化的,所以对于反射攻击这些单例模式没有安全性可言。

其二，对于实现了`Serializable`的单例,序列化机制可以创建一个单例对象。原因是在`ObjectInputStream`内本质上还是会通过反射创建单例对象。但是不同于直接反射，序列化机制为这种情况留了一条后路，也就是`readResolve()`方法。

```java
//在单例中写这个方法
protected Object readResolve()throws ObjectStreamException
{
    //只需要在这里编写返回单例的代码即可
}
```

这样在调用`ObjectInputStream.readObject()`时就不再会使用反射创建对象,而是会返回`readResolve()`的单例。

但由于还是能直接通过反射跳过`private`修饰符,所以其实还是不安全。

#### 2、枚举单例的安全性

单例模式一直以来要考虑的问题无非就是：多线程安全性、反射攻击安全性、序列化攻击安全性。而通过Java自身的支持，这三点`enum`都天然满足了。

先来说多线程安全性。我们知道，枚举实际上会在前端编译后解语法糖，最终会将声明在`enum`内的成员编译成静态常量,而这些静态常量会在一个static块中赋值。由于static块只会在类加载的初始化阶段被调用一次，而类加载过程是由JVM保证互斥的，所以`enum`的多线程安全性已有JVM保证。

而对于反射攻击的安全性，我们来直接看看`Constructor.newInstance()`源码，有这样的一个判断。

```java
//Constructor.java
if ((clazz.getModifiers() & Modifier.ENUM) != 0)
            throw new IllegalArgumentException("Cannot reflectively create enum objects");
```

这里就会判断如果要实例化的类是`ENUM`修饰的,则直接抛出一个异常。反射攻击安全性也是由Java类库保证了。

最后对于序列化攻击安全性，`ObjectOutPutStream`对枚举做了一个特殊处理。==在序列化的时候仅仅是将枚举对象的name属性输出到流中，反序列化的时候则是通过`java.lang.Enum.valueOf()`方法来根据名字查找枚举对象，此时返回的就是在`enum`的静态常量,也就是单例(这些源码在`readEnum()`中)。同时，编译器是不允许任何对这种序列化机制的定制的，因此禁用了writeObject、readObject、readObjectNoData、writeReplace和readResolve等方法。==至此我们就可以得知,为了枚举的序列化安全性,Java禁止了一切自定义序列化的方式（那是因为在`Enum`类中实现了这几个方法并将其标识为final）。

所以,枚举单例就是最安全也是最简单的单例写法。

最后附上这个单例。

```java
public enum EnumSingleton
{
    INSTANCE(1);
    private int state;
    EnumSingleton(int state)
    {
        this.state=state;
    }
    public void whateverMethod()
    {
        System.out.println("I am singleton!");
    }
}
```

### Ⅳ、版本管理

存储一个对象时，这个对象所属的类也必须被存储。`ObjectOutPutStream`和`ObjectInPutStream`通过一个类的指纹唯一标识一个类。而这个指纹又是通过对该对象所属的类、超类、接口、成员变量类型和方法签名按照规范方式排序，然后用SHA安全散列算法应用于这些数据获得的。

在`ObjectInPutStream`读入一个对象的时候，会拿流中对象所属类的指纹和当前在虚拟机方法区的对应类信息的指纹作对比，若是两者的指纹不匹配，那么就说明这个类的定义和本Java进程中的目标类定义不相同，此时`ObjectInPutStream`会抛出一个异常。也就是说在两个Java进程使用序列化传输对象过程中，如果输出方Java进程在旧版本的类基础上新增了一个int域，仍是旧版本的类的接收方就无法读取序列化数据创建对象。

但是在大多数时候一些新版本所添加的方法和域是不会影响旧版本类的运行的，这个时候就需要一个办法以跳过这个指纹检查。那就是`private static final long serialVersionUID`的作用。如果在类中定义了这个成员，那么序列化时就会将这个`serialVersionUID`作为指纹放到输出流中(这是个特例,其他静态变量是不会被序列化的),在`ObjectInPutStream`读入一个对象的时候就会直接比较两者的`serialVersionUID`，不在根据类信息生成指纹比较。如果两者`serialVersionUID`相同就说明指纹相同可以反序列化创建对象，此时`ObjectInPutStream`就会尽力将新版本的对象数据填充到旧版本的域中（但是旧版本→新版本的过程就有可能会出现对象某些域的值为null，这个时候就需要`readObject()`来处理这个不兼容了）。