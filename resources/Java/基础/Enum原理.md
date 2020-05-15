# Enum原理

首先来看看Enum的几种用法：

* ```java
  public class EnumDemo {
  
      public static void main(String[] args){
          //直接引用
          Day day =Day.MONDAY;
      }
  
  }
  //定义枚举类型
  enum Day {
      MONDAY, TUESDAY, WEDNESDAY,
      THURSDAY, FRIDAY, SATURDAY, SUNDAY
  }
  ```

*  

  ```java
  public enum Day2 {
      MONDAY("星期一"),
      TUESDAY("星期二"),
      WEDNESDAY("星期三"),
      THURSDAY("星期四"),
      FRIDAY("星期五"),
      SATURDAY("星期六"),
      SUNDAY("星期日");//记住要用分号结束
  
      private String desc;//中文描述
  
      /**
       * 私有构造,防止被外部调用
       * @param desc
       */
      private Day2(String desc){
          this.desc=desc;
      }
  
      /**
       * 定义方法,返回描述,跟常规类的定义没区别
       * @return
       */
      public String getDesc(){
          return desc;
      }
  
  }
  ```

* ```java
  public enum EnumDemo3 {
  
      FIRST{
          @Override
          public String getInfo() {
              return "FIRST TIME";
          }
      },
      SECOND{
          @Override
          public String getInfo() {
              return "SECOND TIME";
          }
      }
  
      ;
  
      /**
       * 定义抽象方法
       * @return
       */
      public abstract String getInfo();
  
  }
  ```

### 一、原理

枚举类型其实是类的一部分,在前端编译时会将枚举类型解语法糖变成一个类。下面就先以第一个用法作为例子来讲解

#### Ⅰ、例子①

在解语法糖之后，EnumDemo的类结构会变成这样

```java
final class Day extends Enum
{
    
    //编译器为我们添加的静态的values()方法
    public static Day[] values()
    {
        return (Day[])$VALUES.clone();
    }
    //编译器为我们添加的静态的valueOf()方法，注意间接调用了Enum也类的valueOf方法
    public static Day valueOf(String s)
    {
        return (Day)Enum.valueOf(com/zejian/enumdemo/Day, s);
    }
    //私有构造函数
    private Day(String s, int i)
    {
        super(s, i);
    }
     //前面定义的7种枚举实例
    public static final Day MONDAY;
    public static final Day TUESDAY;
    public static final Day WEDNESDAY;
    public static final Day THURSDAY;
    public static final Day FRIDAY;
    public static final Day SATURDAY;
    public static final Day SUNDAY;
    private static final Day $VALUES[];

    
    static 
    {    
        //实例化枚举实例
        MONDAY = new Day("MONDAY", 0);
        TUESDAY = new Day("TUESDAY", 1);
        WEDNESDAY = new Day("WEDNESDAY", 2);
        THURSDAY = new Day("THURSDAY", 3);
        FRIDAY = new Day("FRIDAY", 4);
        SATURDAY = new Day("SATURDAY", 5);
        SUNDAY = new Day("SUNDAY", 6);
        $VALUES = (new Day[] {
            MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
        });
    }
}
```

该类被`final`修饰,是不可被继承的,并且该类是一个继承于`Enum`的抽象类(虽然没有`abstract`修饰)。在枚举类型中声明的枚举实例实际上就是些以枚举类`Day`为类型的静态常量。

其中要注意的点是枚举类的构造方法，这是一个二参构造器，并且用`private`修饰(因为只有静态块才允许构造该枚举对象,这解出来的语法糖只能由虚拟机自己使用),第一个参数代表枚举实例的变量名，第二个参数则代表的是枚举实例在代码中声明的顺序，构造方法体中则会直接调用父类的构造器将两个参数交由`Enum`类去管理,而管理的目的就是为了能让用户使用下列这些`Enum`抽象类方法。

| 返回类型    | 方法名称                                      | 方法说明                                                     |
| ----------- | --------------------------------------------- | ------------------------------------------------------------ |
| `int`       | `compareTo(E o)`                              | 比较此枚举与指定对象的顺序                                   |
| `boolean`   | `equals(Object other)`                        | 当指定对象等于此枚举常量时，返回 true。                      |
| `Class`     | `getDeclaringClass()`                         | 返回与此枚举常量的枚举类型相对应的 Class 对象                |
| `String`    | `name()`                                      | 返回此枚举常量的名称，在其枚举声明中对其进行声明             |
| `int`       | `ordinal()`                                   | 返回枚举常量的序数（它在枚举声明中的位置，其中初始常量序数为零） |
| `String`    | `toString()`                                  | 返回枚举常量的名称，它包含在声明中                           |
| `static> T` | `static valueOf(Class enumType, String name)` | 返回带指定名称的指定枚举类型的枚举常量。                     |

#### Ⅱ、例子②

例子②解语法糖之后的类结构有些不同，主要体现在构造函数中

```java
final class Day extends Enum
{
    //编译器为我们添加的静态的values()方法
    public static Day[] values()
    {
        return (Day[])$VALUES.clone();
    }
    //编译器为我们添加的静态的valueOf()方法，注意间接调用了Enum也类的valueOf方法
    public static Day valueOf(String s)
    {
        return (Day)Enum.valueOf(com/zejian/enumdemo/Day, s);
    }
    //私有构造函数
    private Day(String s, int i,String desc)
    {
        super(s, i);
        this.desc=desc;
    }
     //前面定义的7种枚举实例
    public static final Day MONDAY;
    public static final Day TUESDAY;
    public static final Day WEDNESDAY;
    public static final Day THURSDAY;
    public static final Day FRIDAY;
    public static final Day SATURDAY;
    public static final Day SUNDAY;
    private static final Day $VALUES[];

    //中文描述
    private Stirng desc;
    
    static 
    {    
        //实例化枚举实例
        MONDAY = new Day("MONDAY", 0,"星期一");
        TUESDAY = new Day("TUESDAY", 1,"星期二");
        WEDNESDAY = new Day("WEDNESDAY", 2,"星期三");
        THURSDAY = new Day("THURSDAY", 3,"星期四");
        FRIDAY = new Day("FRIDAY", 4,"星期五");
        SATURDAY = new Day("SATURDAY", 5,"星期六");
        SUNDAY = new Day("SUNDAY", 6,"星期天");
        $VALUES = (new Day[] {
            MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
        });
    }
}
```

可以看到,解语法糖后构造函数变成了三个参数,其中有一个参数就是用来给`desc`赋值.

#### Ⅲ、例子③

由于枚举类型解语法糖后变成了一个抽象类，所以这种在枚举实例中声明重写方法的举动其实就相当于匿名内部类重写方法而已

```java
  static 
    {    
        //实例化枚举实例
        MONDAY = new Day("MONDAY", 0)
        {
         @Override
        public String getInfo() {
            return "FIRST TIME";
        }
        };
  }
```

