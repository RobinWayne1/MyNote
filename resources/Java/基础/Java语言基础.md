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
2. Java用补码来表示数据，在后面讲移位操作时要用到这个前置知识

#### 2、浮点类型

| 类型   | 存储需求 |
| ------ | -------- |
| float  | 4字节    |
| double | 8字节    |

#### 3、char类型

| 类型 | 存储需求                         |
| ---- | -------------------------------- |
| char | 1字节~2字节，根据Unicode字符判断 |

#### 4、boolean类型

| 类型    | 存储需求 |
| ------- | -------- |
| boolean | 1字节    |

### Ⅱ、数据类型之间的转换

当有两个数值进行二元操作时，虚拟机先要将两个操作数转换为同一个类型，再进行计算。

若两操作数有一个是double类型，则另一个操作数就会转换为double类型。

否则，若两操作数有一个是float类型，则另一个操作数就会转换为float类型。

否则，若两操作数有一个是long类型，则另一个操作数就会转换为long类型。

否则，两个操作数都将被转换为int类型。

### Ⅲ、强制类型转换

通过上面的数据类型自动转换可以发现，自动转换都是宽化操作。若是要窄化操作就需要手动进行强制类型转换，因为这有可能丢失部分数据。

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
  * 多态的含义就是一个对象引用变量可以指示多种实际类型的现象，在编译期时无法确定某个方法的实际接收者（实际类型），只有在运行期的方法调用过程中才能得知到底是哪个类中的方法，而这个自动选择调用方法的现象则称为动态绑定。
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

没什么好注意的。用`Object.hash()`重写就是了,其他去看`hashcode()与equals().md`。

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

* 浅拷贝：对基本数据类型进行值传递，对引用数据类型进行引用传递般的拷贝
* 深拷贝：对基本数据类型进行值传递，对引用数据类型，创建一个新的对象，并复制其内容

### Ⅶ、抽象类

1. 抽象类可以包含具体数据和具体方法
2. 包含一个或多个抽象方法的类本身必须被声明为抽象的
3. 抽象类不能被实例化。也就是说可以定义一个抽象类的对象变量，但是他只能引用非抽象子类的对象。

### Ⅷ、接口

#### 1、基本特性

1. 接口中所有方法自动地属于public
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

1. 接口中所有方法自动地属于public，抽象类则不一定
2. Jdk1.8之前接口不能由方法实现，而抽象类中可以有抽象方法也可以有非抽象方法。
3. 接口不能有实例域，接口中的域将被自动设为`public static final`。而抽象类则可以有实例域。
4. 不能使用new实例化接口,接口并不能被实例化
5. 一个类只能继承一个抽象类，但可以实现多个接口。接口本身可以用extends扩展。

## 三、异常

假设一个Java程序运行期间出现了错误。这个错误可能是由于文件包含错误信息，亦或者是程序逻辑错误等。异常处理机制就是要达到一下两个目的：

* 将出现异常的程序返回到一种安全状态，并能够让用户执行一些其他命令
* 或者允许用户保存所有操作结果，以妥善的方式终止程序

### Ⅰ、异常分类

异常对象都是派生与`Throwable`类的一个实例。

![](E:\Typora\MyNote\resources\Java\基础\Throwable.png)

`Error`类层次结构描述了Java运行时系统内部错误和资源耗尽错误,如`OutOfMemoryError`、`StackOverFlowError`，出现了这样的内部错误除了通告给用户并尽力使程序安全终止外也无能为力了。

`Exception`层次结构是设计程序时需要关注的地方,该层次结构又分为两个分支:一个分支派生于`RuntimeException`,另一个分支则包含其他异常。两个分支的重要区别是：由程序员编写的程序错误导致的异常属于`RuntimeException`；而程序本身没有问题，但由于像I/O错误这类问题导致的异常属于其他异常。

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

这样做的目的是改变异常的类型和信息描述,使得用户能更好的分析异常并定位异常。