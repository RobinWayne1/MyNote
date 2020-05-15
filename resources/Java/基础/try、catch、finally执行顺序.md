# try、catch、finally执行顺序

```java
/*代码块1*/
public class ExceptionTest
{
    public int inc()
    {
        int x;
        try{
            x=1;
            return x;
        }catch (Exception e)
        {
            x=2;
            return x;
        }
        finally
        {
            x=3;
        }
    }
    public static void main(String[]args)
    {

        ExceptionTest e=new ExceptionTest();
        int i=e.inc();
        System.out.println(i);
    }
}

```

```assembly
/*代码块2*/
{
  public ExceptionTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 5: 0

  public int inc();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=5, args_size=1
         0: iconst_1               /*声明常量1，并将其放入操作数栈顶*/
         1: istore_1
         2: iload_1					
         3: istore_2			   /*将1复制到Slot 2中*/ /*现在Slot1和Slot2都是1*/
         4: iconst_3			   /*finally块中声明的常量3*/
         5: istore_1			   /*将操作数栈顶的3放入Slot 1*//*Slot1:3;Slot2:1*/
         6: iload_2				   /*将1复制到操作数栈顶,⭐返回try修改过的slot2的值,而不是finally修改过的slot1值*/
         7: ireturn				   /*返回1*/
         
         8: astore_2				/*catch块开始，此句是将异常对象引用出栈放入Slot2中*/
         9: iconst_2				
        10: istore_1				/*将常量2出栈放入Slot1，⭐后面三行字节码做了个复制的操作*/
        11: iload_1					
        12: istore_3				/*局部变量表情况Slot1:2,Slot2:Exception,Slot3:2*/
        13: iconst_3				/*finally块中声明的常量3*/
        14: istore_1				/*局部变量表情况Slot1:3,Slot2:Exception,Slot3:2*/
        15: iload_3					/*⭐返回catch修改过的slot3的值,而不是finally修改过的slot1值*/
        16: ireturn					/*catch块中的return,此时将返回2*/
        
        17: astore        4			/*出现不属于Exception的异常则会跳转到这里*/
        19: iconst_3				/*finally块中声明的常量3*/
        20: istore_1				/*局部变量表情况Slot1:3,Slot4:不属于Exception的异常*/
        21: aload         4			
        23: athrow					/*抛出异常*/
      Exception table:
         from    to  target type
             0     4     8   Class java/lang/Exception
             0     4    17   any
             8    13    17   any
            17    19    17   any

}

```

```java
/*代码块3*/
public class ExceptionTest
{
    public Instance inc()
    {
        Instance instance=new Instance();
        try{
            instance.flag=1;
            return instance;
        }catch (Exception e)
        {
            instance.flag=2;
            return instance;
        }
        finally
        {
            instance.flag=3;

        }
    }
    public static void main(String[]args)
    {

        ExceptionTest e=new ExceptionTest();
        Instance instance=e.inc();
        System.out.println(instance.flag);
    }
}
class Instance
{
    public int flag=0;

}
```

```assembly
/*代码块4*/
public class ExceptionTest
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #11.#27        // java/lang/Object."<init>":()V
   #2 = Class              #28            // Instance
   #3 = Methodref          #2.#27         // Instance."<init>":()V
   #4 = Fieldref           #2.#29         // Instance.flag:I
   #5 = Class              #30            // java/lang/Exception
   #6 = Class              #31            // ExceptionTest
   #7 = Methodref          #6.#27         // ExceptionTest."<init>":()V
   #8 = Methodref          #6.#32         // ExceptionTest.inc:()LInstance;
   #9 = Fieldref           #33.#34        // java/lang/System.out:Ljava/io/PrintStream;
  #10 = Methodref          #35.#36        // java/io/PrintStream.println:(I)V
  #11 = Class              #37            // java/lang/Object
  #12 = Utf8               <init>
  #13 = Utf8               ()V
  #14 = Utf8               Code
  #15 = Utf8               LineNumberTable
  #16 = Utf8               inc
  #17 = Utf8               ()LInstance;
  #18 = Utf8               StackMapTable
  #19 = Class              #31            // ExceptionTest
  #20 = Class              #28            // Instance
  #21 = Class              #30            // java/lang/Exception
  #22 = Class              #38            // java/lang/Throwable
  #23 = Utf8               main
  #24 = Utf8               ([Ljava/lang/String;)V
  #25 = Utf8               SourceFile
  #26 = Utf8               ExceptionTest.java
  #27 = NameAndType        #12:#13        // "<init>":()V
  #28 = Utf8               Instance
  #29 = NameAndType        #39:#40        // flag:I
  #30 = Utf8               java/lang/Exception
  #31 = Utf8               ExceptionTest
  #32 = NameAndType        #16:#17        // inc:()LInstance;
  #33 = Class              #41            // java/lang/System
  #34 = NameAndType        #42:#43        // out:Ljava/io/PrintStream;
  #35 = Class              #44            // java/io/PrintStream
  #36 = NameAndType        #45:#46        // println:(I)V
  #37 = Utf8               java/lang/Object
  #38 = Utf8               java/lang/Throwable
  #39 = Utf8               flag
  #40 = Utf8               I
  #41 = Utf8               java/lang/System
  #42 = Utf8               out
  #43 = Utf8               Ljava/io/PrintStream;
  #44 = Utf8               java/io/PrintStream
  #45 = Utf8               println
  #46 = Utf8               (I)V
{
  public ExceptionTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 5: 0

  public Instance inc();
    descriptor: ()LInstance;
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=5, args_size=1
         0: new           #2                  // class Instance
         3: dup
         4: invokespecial #3                  // Method Instance."<init>":()V
         7: astore_1
         8: aload_1
         9: iconst_1
         /*putfield的的具体操作时弹出两次栈顶,并根据第二次弹出的对象引用和putfield参数找到目标对象的成员变量,之后将第一次弹出的常量1赋值给这个成员变量*/
        10: putfield      #4                  //将1赋值给instance.flag
        13: aload_1							  
        14: astore_2						  /*局部变量表情况Slot1:instance对象,Slot2:instance对象*/
        15: aload_1								
        16: iconst_3						  /*finally块中的3*/
        17: putfield      #4                  //⭐将3赋值给instance.flag
        20: aload_2							  /*⭐重点注意这里是load Slot2而不是Slot1*/
        21: areturn							  /*返回对象引用*/
        
        22: astore_2						  /*catch块开始执行,此处将异常保存到Slot2*/
        23: aload_1
        24: iconst_2
        25: putfield      #4                  //将2赋值给instance.flag
        28: aload_1
        29: astore_3
        30: aload_1
        31: iconst_3						  /*finally块中的3*/
        32: putfield      #4                  //⭐将3赋值给instance.flag
        35: aload_3							  /*⭐重点注意这里是load Slot3而不是Slot1,即返回的是catch使用过的对象引用而不是finally使用过的对象引用*/
        36: areturn
        
        37: astore        4
        39: aload_1
        40: iconst_3						  /*finally块中的3*/
        41: putfield      #4                  // Field Instance.flag:I
        44: aload         4
        46: athrow
      Exception table:
         from    to  target type
             8    15    22   Class java/lang/Exception
             8    15    37   any
            22    30    37   any
            37    39    37   any

}

```

先讲一下Exception table的读法，如代码块2中的第一行，意义是：**在字节码行数中若第0行到第4行出现了Exception异常，则跳去第8行继续执行，此时操作数栈顶是有一个异常对象的。**

接下来说明普通的try-catch-finally的编译情况。可以看到，==无论是代码块2或4或6，都被分成了(try-finally),(catch-finally)和(finally-抛出未知异常)这三大部分，**⭐即finally块中编译后的代码被插入到了三个部分里，并将return语句移动到finally块里的代码执行完后再执行**,这就是虚拟机对finally块的实现==。**而虚拟机对异常的捕获则是通过查看异常表，根据异常表提供的异常出现位置和异常类型跳转到相应的字节码块中执行。**

接下来说明一下在finally块中没有return语句时虚拟机对代码的编译情况。通过对比代码块2和4中画⭐的注释可以发现，虚拟机对这种情况处理的套路是：**在try块执行完成后(字节码层面的执行完成)，会对要返回的的值进行一次在局部变量表槽的复制。其后若finally块要使用这个变量,则会使用原槽中的变量;return语句则会返回复制后的槽中的变量,即finally块使用的和返回的并非是一个槽中的引用变量。所以这就会产生若是在finally块对对象成员变量修改，return时就有效(因为此时槽中是引用变量,是地址)，而对基本数据类型如整型修改就无效的情况。(因为此时槽中是值)**==**⭐(经测试，这个复制机制的出现条件是有finally块且try或catch块中有返回值，且只对这个try或catch块的返回值进行复制，无论finally块中有没有用到这个返回值)**== 

接下来解释一下finally块中有return语句的编译情况。对比代码块6和代码块4,区别只有在第20行字节码中:**finally块中有return语句时,finally块中的return语句会覆盖前面的return语句（try块、catch块中的return语句）（==即会返回finally块使用过的槽的引用变量==),所以此时对返回值的修改是必然有效的!**

```java
/*代码块5*/
public class ExceptionTest
{
    public Instance inc()
    {
        Instance instance=new Instance();
        try{
            instance.flag=1;
            return instance;
        }catch (Exception e)
        {
            instance.flag=2;
            return instance;
        }
        finally
        {
            instance.flag=3;
            return instance;
        }
    }
    public static void main(String[]args)
    {

        ExceptionTest e=new ExceptionTest();
        Instance instance=e.inc();
        System.out.println(instance.flag);
    }
}
class Instance
{
    public int flag=0;
}
```

```assembly
/*代码块6*/
Constant pool:
   #1 = Methodref          #11.#27        // java/lang/Object."<init>":()V
   #2 = Class              #28            // Instance
   #3 = Methodref          #2.#27         // Instance."<init>":()V
   #4 = Fieldref           #2.#29         // Instance.flag:I
   #5 = Class              #30            // java/lang/Exception
   #6 = Class              #31            // ExceptionTest
   #7 = Methodref          #6.#27         // ExceptionTest."<init>":()V
   #8 = Methodref          #6.#32         // ExceptionTest.inc:()LInstance;
   #9 = Fieldref           #33.#34        // java/lang/System.out:Ljava/io/PrintStream;
  #10 = Methodref          #35.#36        // java/io/PrintStream.println:(I)V
  #11 = Class              #37            // java/lang/Object
  #12 = Utf8               <init>
  #13 = Utf8               ()V
  #14 = Utf8               Code
  #15 = Utf8               LineNumberTable
  #16 = Utf8               inc
  #17 = Utf8               ()LInstance;
  #18 = Utf8               StackMapTable
  #19 = Class              #31            // ExceptionTest
  #20 = Class              #28            // Instance
  #21 = Class              #30            // java/lang/Exception
  #22 = Class              #38            // java/lang/Throwable
  #23 = Utf8               main
  #24 = Utf8               ([Ljava/lang/String;)V
  #25 = Utf8               SourceFile
  #26 = Utf8               ExceptionTest.java
  #27 = NameAndType        #12:#13        // "<init>":()V
  #28 = Utf8               Instance
  #29 = NameAndType        #39:#40        // flag:I
  #30 = Utf8               java/lang/Exception
  #31 = Utf8               ExceptionTest
  #32 = NameAndType        #16:#17        // inc:()LInstance;
  #33 = Class              #41            // java/lang/System
  #34 = NameAndType        #42:#43        // out:Ljava/io/PrintStream;
  #35 = Class              #44            // java/io/PrintStream
  #36 = NameAndType        #45:#46        // println:(I)V
  #37 = Utf8               java/lang/Object
  #38 = Utf8               java/lang/Throwable
  #39 = Utf8               flag
  #40 = Utf8               I
  #41 = Utf8               java/lang/System
  #42 = Utf8               out
  #43 = Utf8               Ljava/io/PrintStream;
  #44 = Utf8               java/io/PrintStream
  #45 = Utf8               println
  #46 = Utf8               (I)V
{
  public ExceptionTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 5: 0

  public Instance inc();
    descriptor: ()LInstance;
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=5, args_size=1
         0: new           #2                  // class Instance
         3: dup
         4: invokespecial #3                  // Method Instance."<init>":()V
         7: astore_1						  /*将调用<init>方法后返回的引用存到Slot 1*/
         8: aload_1
         9: iconst_1
        10: putfield      #4                  //将1赋值给instance.flag
        13: aload_1
        14: astore_2						  /*⭐就算后面的代码没有用到复制后的槽液依然将Slot 1的instance引用拷贝一份到Slot 2,证实了复制条件的猜测*/
        							          /*此时局部变量表情况Slot1:instance对象;Slot2:instance对象*/
        15: aload_1
        16: iconst_3						  /*finally块中声明的3*/
        17: putfield      #4                  //使用Slot1的instance引用,将3赋值给instance.flag
        20: aload_1							  /*⭐重点注意:此时返回的则是finally修改过的slot1,与上面finally块中没有return作比较*/
        21: areturn							  /*返回Slot 1的instance引用*/
        
        22: astore_2
        23: aload_1
        24: iconst_2
        25: putfield      #4                  // Field Instance.flag:I
        28: aload_1
        29: astore_3					     
        30: aload_1
        31: iconst_3
        32: putfield      #4                  // Field Instance.flag:I
        35: aload_1							   /*⭐重点注意:此时返回的同样是finally修改过的slot1,与上面finally块中没有return作比较*/
        36: areturn
        
        37: astore        4					  /*出现非Exception异常时会来到这里*/
        39: aload_1
        40: iconst_3
        41: putfield      #4                  // Field Instance.flag:I
        44: aload_1
        45: areturn							  /*⭐⭐这里依然是return,这会导致try块中如果出现不是Exception或其子类的异常时,这个异常会被吞掉,不会抛出到调用的栈帧;但finally块中出现异常时却仍然能报错*/
      Exception table:
         from    to  target type
             8    15    22   Class java/lang/Exception
             8    15    37   any
            22    30    37   any
            37    39    37   any
    
}

```

