# Chapter 1. Java基础
Java是一门非常注重类与对象的语言，它认为**所有的代码都应该在一个类之下，包括main**。
同为面向对象的语言，Java在许多地方与cpp有相似之处，本章采用在这两种语言之间建立映射的方式，帮助快速理解和上手Java，当然也会在后面的DS课正式开始后查漏补缺

## 1.1 hello world
下面是一个简单的输出字符串*hello world*的java程序：
```
public class HelloWorld {
	public static void main(String[] args) {
		System.out,println("hello world");
		}
}
```
可以看见整个程序使用了类似cpp中类的定义所使用的访问修饰符`public`和关键字`class`，说明**它是被以一个类的形式组织的**
在`main`方法的参数行中，我们看见了`String[] args`。和cpp一样，**`main`方法是整个程序的主方法入口：所有的Java程序都从`main`方法开始执行**。而`String[] args`是字符串类，类似于cpp中的**命令行参数**，用于在命令行中调用时为程序传递参数
![[Pasted image 20240914215739.png]]

Java程序文件保存时的扩展名为`.java`。因为整个程序被以类的形式组织，所以**保存`.java`文件时，文件名必须与类名相同**，也就是该hello world程序应被保存为`HelloWorld.java`
在我们安装好Java编译器，并将该程序保存为`HelloWorld.java`后，可以在shell中输入如下指令运行程序：
```
javac HelloWorld.java
	java HelloWorld
```
若有编码问题，则只需为`javac`指令添加一个选项`-encoding`设置`utf-8`来编译：
```
javac -encoding UTF-8 HelloWorld.java
java HelloWorld
```

这个流程非常像cpp使用`g++`编译器生成中间目标文件和可执行程序的过程。事实也确实如此，Java和编译型语言的区别可以用下图说明：
![[Pasted image 20240914220252.png]]

![[Pasted image 20240916214235.png]]

## 1.2 Java对象和类
Java作为一种面向对象的语言，它的基本概念与语法如下：
1. 类Class：定义对象的蓝图，包括其属性与方法，使用`class`声明
```
public class Car { ... }
```

2. 对象Object：类的实例，具有状态和行为
```
Car myCar = new Car();
```

3. 继承：一个类可以继承另一个类的属性和方法，使用关键字`extends`声明
```
public class Dog extends Animal { ... }
```

4. 封装：将对象的状态，即字段，私有化，只能通过public的方法进行间接访问
```
private String name;
public String getName() { return name; }
\\使用getName()安全地获得私有成员
```

5. 抽象：**针对类**。不提供具体实现，只提供抽象类和接口的定义而不实现，使用关键字`abstract`声明（一个抽象对应一个`abstract`，**类的方法如果是抽象，也要单独声明**），只能单继承（**一个类只能继承自一个抽象类**）
```
public abstract class Shape { abstract void draw(); }
```

6. 接口：**针对方法的“抽象”**。定义类**必须实现的方法（只是定义，没有实现）**，使用关键字`interface`，**大括号内为该接口所定义的所有方法的声明**。它支持多重继承（一个接口可以继承多个其他接口，**一个类若实现了该接口的方法，就要实现所有被继承的接口的方法**），类使用`implements 接口名`（implements 实现）的方式声明字节采用该接口，并在类中实现。（后面说）
```
public interface Animal { void eat(); }
```

7. 方法：定义类的行为，是**包含在类内的函数**
```
public class MathUtils {
public int add(int a, int b) { return a+b; }
public double add(double a, double b){ return a+b; }
}
```
**方法可以被重载，同名但参数不同**

一个类可以包含如下类型的变量：
- 局部变量：在方法、构造方法或语句块中定义的变量，**其声明和初始化都在方法中，随方法返回而被销毁**
- 成员变量：在类中定义的变量，**位于方法体之外**，它们在创建对象的时候被实例化，可以被类的方法、构造方法以及特定类的语句块访问
- 类变量：相当于cpp的**静态成员变量**，在类中定义，也位于方法体之外，但是**数据对于所有实例保持一致**，需要声明为`static`，**存储在方法区（共享数据区）的静态区，可以在没有对象时直接使用类名调用，因此没有`this`（指向对象自己的指针）和`super`（指向父类对象的指针）等方法进行自己的指代（可能不存在对象）**

想象人体和器官的关系，若我们的类需要有一个其他类的成员来帮助实现其功能，常见的做法是定义一个其他类的对象作为成员：
```
public class Person {
    private int blood;
    private Heart heart;
}

public class Heart {
    private int blood;
    public void test() {
        System.out.println(blood);
    }
}
```
但我们其实也可以**在类内直接定义另一个类**，相比起来这**会让代码结构变得很奇怪**：
```
//外部类
class Out {
    private int age = 12;
     
    //内部类
    class In {
        public void print() {
            System.out.println(age);
        }
    }
}
 
public class Demo {
    public static void main(String[] args) {
        Out.In in = new Out().new In();
        in.print();
        //这里in的得出可以换成下面的直观语句：
        /*
        Out out = new Out();
        Out.In in = out.new In();
        in.print();
        */
    }
}
```
但好处在于：**在外部类的内部定义内部类，这就可以让内部类随意地使用外部类的成员变量，无需先生成外部类的对象再进行访问，会更方便一些**

### 1.2.1 类的构造方法
类似于cpp的构造函数，如果没有显式地为类提供构造方法，编译器会为其生成一个。
**构造方法的名称必须与类同名，一个类可以有多个构造方法**，创建对象时至少需要调用一个
```
public class Puppy {  \\类 
	public Puppy() {  \\构造方法
	}
}
```
和cpp一样，**一旦自己定义了一个拷贝/移动/赋值构造方法，那就不会自动生成其他的构造方法了，想用得自己显式定义**

### 1.2.2 类对象的创建
Java中，**使用关键字`new`来创建一个对象，在后面调用类的构造方法来创建**
```
public Puppy {
	public Puppy(String name){
	System.out.println(“创建了小狗：”+name); }
}  //类及其构造方法

public static void main(String[] args) {  //主方法
	Puppy myPuppy = new Puppy("tommy");  //调用Puppy类同名的构造方法
}
```

### 1.2.3 访问实例变量与调用成员方法
可以通过已创建的对象来访问其公开的成员变量和成员方法，和cpp类似，使用点运算符`.`即可：
```
public class Puppy { 
	private int age;
	private String name; // 构造器 
	public Puppy(String name) { this.name = name; System.out.println("小狗的名字是 : " + name); }
	public int getAge() { return age; }  //获取age成员的方法
	public void setAge(int age) { this.age = age; }  //修改age成员的方法
	}
	

// 主方法 
public static void main(String[] args) { // 创建对象 
Puppy myPuppy = new Puppy("Tommy"); 
// 通过方法来设定 age 
myPuppy.setAge(2); 
// 调用另一个方法获取 
age int age = myPuppy.getAge();
}
```

### 1.2.4 源文件与包的结构
Java中，源文件指的是包含程序源代码的文件，通常具有`.java`的扩展名。
当在一个源文件中定义多个类时，要注意以下规则：
- **一个源文件中只能有一个`public`类**，`public`接口和非`public`类可以有多个。其他非`public`类只能被文件内访问，不提供公共接口
- **源文件的名称应与`public`类的类名一致**

Java中，**可以使用`package`语句来指定类或接口所属的包，应该放在源文件的==首行==指明下面的类定义在该包中**：
```
package com.example.utils;
public class MyUtility {
//类的内容
}
```
这类似于cpp的`namespace`，用于防止重名冲突。Java编译器在**编译时，根据`package aaa.bbb.ccc`指定的信息将生成的`.class`文件生成到目录`./aaa/bbb/ccc/`下**，用`.`分隔层级

**注意区分“包”和“源文件”：一个源文件只能有一个`public`类，多个源文件可以同属于一个包，因此一个包内可以有多个`public`类**

Java中，给出一个完整的限定名（包名+类名），那么编译器就能很容易定位到源代码或者类。使用`import`语句提供导入的路径。**如果源文件有`package`语句，那么`import`语句应该放在`package`语句和类的定义之间；没有`package`则放在最前面**
多个源文件可以声明至同一个包使得包有多个`public`类，单独使用包内某个类：
```
import com.example.utils.MyUtility;
```
也支持使用通配符`*`，将**一个包里的所有`public`类**引入：
```
import com.example.utils.*;
```

Java程序是从一个public类的main函数（线程）开始执行的，规定一个编译单元（.java文件）只能有一个`public`类，是因为**每个编译单元只能有至多一个==公共接口==**，便于类装载器的工作。当然编译单元也可以没有公共接口
**没有公共接口的编译单元（文件）用于：**
- **封装内部实现细节：将公共接口提供的内部细节封装地实现起来，仅对包内可见**。
- **提供包内接口：可以被同一个包内的其他类和接口使用的接口，但包外无法访问它们**
- **提供仅包内可见的辅助类、枚举、注解等内容，用于辅助`public`的实现**

## 1.3 基本数据类型
变量就是在内存中申请空间来存储值。
$$ Java两大基本数据类型 ：\begin {cases}
内置数据类型\\
引用数据类型
\end{cases} $$
### 1.3.1 内置数据类型
Java语言提供了八种基本类型：**六种数字类型（四个整数型，两个浮点型），一种字符类型，还有一种布尔型**。我们只将与cpp有不同的类型单独介绍，其他的都和cpp一样
六种数字类型：`byte`、`short`、`int`、`long`、`float`、`double`
其中`byte`是**8位有符号的二进制补码整数**，其作用就是在大型数组中代替32位的`int`，节省空间

布尔型`boolean`：和cpp的`bool`一样，1位布尔值，默认为`false`
字符类型`char`：**是一个单一的16位Unicode字符，而非ASCII字符，这使得它可以存储任何字符（其实和cpp的char一样）**

在Java中，内置数据类型的各项数值可以根据以下方法得出：
```
type.SIZE  //二进制位数
type.MAX_VALUE  type.MIN_VALUE  //最大值和最小值
```
两个浮点数的最大值和最小值以科学计数法得出：$aEb = a*10^b$
**小数数据默认是`double`类型，除非在数字后面加上字符`f`或`F`

### 1.3.2 引用类型
在计算机存储时，我们的数据必须被编码为一堆二进制位，当我们声明一个变量时，就会分配一个空间，用于存储该变量的值的二进制位（未赋值时也有可能存储某些之前留下的东西，但无法访问它们）。在Java中，**任何不是八种基本类型的变量都是引用类型的，相当于cpp的指针**，这是因为无法确定用固定位数来存储任何自定义类型变量，只能**存下一段地址用于指示其存储位置**，即使用了`new`关键字调用构造方法分配了一段内存的地方，因此在Java中两个这样的变量使用=时其实只是指向对象（地址）的转接
Java中**引用类型的变量非常类似于cpp的指针，==指向一个对象的变量就是引用变量==，在声明时被指定为一个特定类型，类型声明后不可更改**
```
Site site = new Site("Runoob");  site是一个指向Site("Runoob")构造出来的对象的引用变量
```
这很类似于cpp中通过`new`为一个无名对象分配内存，然后返回一个指向该对象的指针的过程
**一个引用变量可以用来引用任何与之兼容的类型**，相当于cpp中将指针指向对象改变

### 1.3.3 Java常量
常量是在运行过程中不可被修改的量，在Java中使用关键字`final`声明：
```
final double PI = 3.1415927;
```
命名时通常用全大写进行区分

### 1.3.4 强制类型转换
Java的强制类型转换语法和cpp类似，格式为：
```
(type)value
```
例如
```
int i = 1;
byte b = (byte)i;
``` 
自动类型转换和字面值的默认值和cpp几乎没有差别，只是注意定义float类型时必须在数字后面跟上F或f

## 1.4 Java变量
Java变量是否会被赋予默认值也和cpp一样有一定的规则：
- 在类中方法、构造函数或块之外声明的**实例变量**在没有明确初始化时，**会被赋予该类型变量的默认值**
- 在方法、构造函数或块之内声明的**局部变量**则需要明确初始化，**不会被赋予默认值，未初始化则导致编译错误**
- 类中用`static`关键字声明的**静态变量**在类加载时会被默认初始化，**只初始化一次**

Java的**参数变量指的是在方法或构造函数中声明的变量，用于接收传递给方法或构造函数的值，它们只在方法或构造函数被调用时存在，并且只能在方法和构造函数内部使用**
和cpp类似，调用方法时我们需要为参数变量传值，可以有**值传递和引用传递**两种方式
值传递就是传递实际参数值的副本，不影响原始值
引用传递传递的是实际参数的引用（内存地址），会影响原始值
Java引用传递的例子如下：
```
MyObject obj = new MyObject(10);
modifyObject(obj);
```


**成员变量名在类内可以直接使用，在类外必须`类名.成员变量名`这样地给出完整名称**

## 1.5 Java修饰符
### 1.5.1 访问修饰符
Java中，可以使用访问控制符来保护对类、变量、方法和构造方法的访问。Java 支持 4 种不同的访问权限。
- **default** (即**默认**，什么也不写）: **在同一包内可见**，不使用任何修饰符。使用对象：类、接口、变量、方法。
    
- **private** : **在同一类内可见**。使用对象：变量、方法。 **注意：不能修饰类（外部类）**
    
- **public** : **对所有类可见**。使用对象：类、接口、变量、方法
    
- **protected** : **对==同一包内==的类和所有子类可见**。使用对象：变量、方法。 **注意：不能修饰类（外部类）**。

![[Pasted image 20240915170041.png]]

**私有访问修饰符private：所有被声明为private的方法、变量和构造方法只允许被所属类访问，==类和接口不能被声明为private==**

**受保护的访问修饰符protected：需要从两种情况分析**
- **子类与基类在同一包中：此时被声明为protected的变量、方法、构造器可以被同一个包中的任何其他类访问**
- **子类与基类在不同包中：子类实例可以访问所有从基类继承的protected方法，但不能访问基类==实例==的protected方法**

protected 可以修饰数据成员，构造方法，方法成员，**不能修饰类（内部类除外）**
**接口及接口的成员变量和成员方法不能声明为 protected**
下面是继承的访问控制原则：
- 父类中声明为 public 的方法在子类中也**必须**为 public。
    
- 父类中声明为 protected 的方法在子类中要么声明为 protected，要么声明为 public，不能声明为 private。
    
- 父类中声明为 private 的方法，不能够被子类继承。

### 1.5.2 非访问修饰符
`final`修饰符可以修饰类：
```
public final class Test {
//......
}
```
**final 类不能被继承**，没有类能够继承 final 类的任何特性。
**一个类不能同时被 abstract 和 final 修饰**。如果一个类包含抽象方法，那么该类一定要声明为抽象类，否则将出现编译错误。
抽象类可以包含抽象方法和非抽象方法。

抽象方法即cpp中的纯虚函数，不能被声明为`final`或`static`，因为其具体实现需要由子类提供
类内的定义方法：**直接使用`abstract`关键字，以分号`;`结尾**
```
public abstract sample();
```

下面则是一些与线程有关的非访问修饰符：
`synchornized`关键字声明的**方法**在同一时间内**只能被一个线程访问**，可以和四个访问修饰符同时应用：
```
public synchronized void showDetails(){ ....... }
```
`transient`修饰符**用于标记类中的字段，使得这些字段在对象序列化时被忽略。**
**序列化就是将对象的状态转换为一种可以存储或传输的格式的过程**，使得数据可以保存到磁盘、传输到网络等，反序列化就是将这种格式转换回原始对象
该修饰符包含在定义变量的语句中，用来预处理类和变量的数据类型

`volatile`修饰符修饰的**成员变量**在每次被线程访问时，都强制从**共享内存**中重新读取该成员变量的值。当成员变量发生变化时，则会强制线程将变化值写回到共享内存，从而**使得不同线程看到的该成员变量值相同，一旦一个线程使其发生变化，所有线程的该成员变量值都会变化**
```
public class MyRunnable implements Runnable { private volatile boolean active; ...}
```

### 1.5.2 Java运算符
Java的运算符和cpp几乎完全一样，也有三元运算符? :
它对向右移位运算符做了细分：`>>`右移位的空位按最高位的数填充；`>>>`则全部用0填充
除此之外还有很多赋值运算符：
![[Pasted image 20240915190034.png]]

Java提供了`instanceof`运算符来**检查对象实例是否是一个特定类型（类类型或接口类型）**，如果运算符**左侧变量所指的对象**是**右侧类或接口的一个对象**，结果为真。
```
object instanceof class
```

## 1.6 循环和条件控制
和cpp一模一样，包括范围for循环，不过java叫做增强for循环。直接用就行：
```
for(声明语句 : 表达式) { 
//代码句子
}
```
声明语句声明的局部变量类型必须和数组元素的类型匹配，作用域限定在循环语句块，值与遍历时数组元素的值相同。表达式是当前遍历的数组名，或者返回数组的方法

## 1.7 Number & Math类
编程中，我们有时需要的是基本数据类型的对象，而非基本数据类型本身。因此Java语言为每个内置数据类型提供了**包装类**，用于将这些类型封装为对象：
![[Pasted image 20240915194044.png]]
**所有的包装类都是抽象类`Number`的子类**，Number 类属于 java.lang 包。
这种**由编译器特别支持的包装称为装箱**，所以当内置数据类型被当作对象使用的时候，编译器会把内置类型装箱为包装类。相似的，编译器也可以把一个对象**拆箱为内置类型**

**Java Math类包含了用于执行基本数学运算的属性和方法，如初等指数、对数、平方根和三角函数。**

Math 的方法都被定义为 static 形式，通过 Math 类可以在主函数中直接调用。下面是常用的一些方法：
下面的表中列出的是 Number & Math 类常用的一些方法：

|序号|方法与描述|
|---|---|
|1|[xxxValue()](https://www.runoob.com/java/number-xxxvalue.html)  <br>将 Number 对象转换为xxx数据类型的值并返回。|
|2|[compareTo()](https://www.runoob.com/java/number-compareto.html)  <br>将number对象与参数比较。|
|3|[equals()](https://www.runoob.com/java/number-equals.html)  <br>判断number对象是否与参数相等。|
|4|[valueOf()](https://www.runoob.com/java/number-valueof.html)  <br>返回一个 Number 对象指定的内置数据类型|
|5|[toString()](https://www.runoob.com/java/number-tostring.html)  <br>以字符串形式返回值。|
|6|[parseInt()](https://www.runoob.com/java/number-parseInt.html)  <br>将字符串解析为int类型。|
|7|[abs()](https://www.runoob.com/java/number-abs.html)  <br>返回参数的绝对值。|
|8|[ceil()](https://www.runoob.com/java/number-ceil.html)  <br>返回大于等于( >= )给定参数的的最小整数，类型为双精度浮点型。|
|9|[floor()](https://www.runoob.com/java/number-floor.html)  <br>返回小于等于（<=）给定参数的最大整数 。|
|10|[rint()](https://www.runoob.com/java/number-rint.html)  <br>返回与参数最接近的整数。返回类型为double。|
|11|[round()](https://www.runoob.com/java/number-round.html)  <br>它表示**四舍五入**，算法为 Math.floor(x+0.5)，即将原来的数字加上 0.5 后再向下取整，所以，Math.round(11.5) 的结果为12，Math.round(-11.5) 的结果为-11。|
|12|[min()](https://www.runoob.com/java/number-min.html)  <br>返回两个参数中的最小值。|
|13|[max()](https://www.runoob.com/java/number-max.html)  <br>返回两个参数中的最大值。|
|14|[exp()](https://www.runoob.com/java/number-exp.html)  <br>返回自然数底数e的参数次方。|
|15|[log()](https://www.runoob.com/java/number-log.html)  <br>返回参数的自然数底数的对数值。|
|16|[pow()](https://www.runoob.com/java/number-pow.html)  <br>返回第一个参数的第二个参数次方。|
|17|[sqrt()](https://www.runoob.com/java/number-sqrt.html)  <br>求参数的算术平方根。|
|18|[sin()](https://www.runoob.com/java/number-sin.html)  <br>求指定double类型参数的正弦值。|
|19|[cos()](https://www.runoob.com/java/number-cos.html)  <br>求指定double类型参数的余弦值。|
|20|[tan()](https://www.runoob.com/java/number-tan.html)  <br>求指定double类型参数的正切值。|
|21|[asin()](https://www.runoob.com/java/number-asin.html)  <br>求指定double类型参数的反正弦值。|
|22|[acos()](https://www.runoob.com/java/number-acos.html)  <br>求指定double类型参数的反余弦值。|
|23|[atan()](https://www.runoob.com/java/number-atan.html)  <br>求指定double类型参数的反正切值。|
|24|[atan2()](https://www.runoob.com/java/number-atan2.html)  <br>将笛卡尔坐标转换为极坐标，并返回极坐标的角度值。|
|25|[toDegrees()](https://www.runoob.com/java/number-todegrees.html)  <br>将参数转化为角度。|
|26|[toRadians()](https://www.runoob.com/java/number-toradians.html)  <br>将角度转换为弧度。|
|27|[random()](https://www.runoob.com/java/number-random.html)  <br>返回一个随机数。|<br>它表示**四舍五入**，算法为 Math.floor(x+0.5)，即将原来的数字加上 0.5 后再向下取整，所以，Math.round(11.5) 的结果为12，Math.round(-11.5) 的结果为-11。|
|12|[min()](https://www.runoob.com/java/number-min.html)  <br>返回两个参数中的最小值。|
|13|[max()](https://www.runoob.com/java/number-max.html)  <br>返回两个参数中的最大值。|
|14|[exp()](https://www.runoob.com/java/number-exp.html)  <br>返回自然数底数e的参数次方。|
|15|[log()](https://www.runoob.com/java/number-log.html)  <br>返回参数的自然数底数的对数值。|
|16|[pow()](https://www.runoob.com/java/number-pow.html)  <br>返回第一个参数的第二个参数次方。|
|17|[sqrt()](https://www.runoob.com/java/number-sqrt.html)  <br>求参数的算术平方根。|
|18|[sin()](https://www.runoob.com/java/number-sin.html)  <br>求指定double类型参数的正弦值。|
|19|[cos()](https://www.runoob.com/java/number-cos.html)  <br>求指定double类型参数的余弦值。|
|20|[tan()](https://www.runoob.com/java/number-tan.html)  <br>求指定double类型参数的正切值。|
|21|[asin()](https://www.runoob.com/java/number-asin.html)  <br>求指定double类型参数的反正弦值。|
|22|[acos()](https://www.runoob.com/java/number-acos.html)  <br>求指定double类型参数的反余弦值。|
|23|[atan()](https://www.runoob.com/java/number-atan.html)  <br>求指定double类型参数的反正切值。|
|24|[atan2()](https://www.runoob.com/java/number-atan2.html)  <br>将笛卡尔坐标转换为极坐标，并返回极坐标的角度值。|
|25|[toDegrees()](https://www.runoob.com/java/number-todegrees.html)  <br>将参数转化为角度。|
|26|[toRadians()](https://www.runoob.com/java/number-toradians.html)  <br>将角度转换为弧度。|
|27|[random()](https://www.runoob.com/java/number-random.html)  <br>返回一个随机数。|

## 1.8 Character类
和Number一样，java为基本类型char提供了包装类Character。Character 类在对象中包装一个基本类型 **char** 的值。Character类提供了一系列方法来操纵字符。
```
Character ch = new Character('a');
```
下面是Character类的方法：

| 序号  | 方法与描述                                                                                    |
| --- | ---------------------------------------------------------------------------------------- |
| 1   | [isLetter()](https://www.runoob.com/java/character-isletter.html)  <br>是否是一个字母           |
| 2   | [isDigit()](https://www.runoob.com/java/character-isdigit.html)  <br>是否是一个数字字符           |
| 3   | [isWhitespace()](https://www.runoob.com/java/character-iswhitespace.html)  <br>是否是一个空白字符 |
| 4   | [isUpperCase()](https://www.runoob.com/java/character-isuppercase.html)  <br>是否是大写字母     |
| 5   | [isLowerCase()](https://www.runoob.com/java/character-islowercase.html)  <br>是否是小写字母     |
| 6   | [toUpperCase()](https://www.runoob.com/java/character-touppercase.html)  <br>指定字母的大写形式   |
| 7   | [toLowerCase](https://www.runoob.com/java/character-tolowercase.html)()  <br>指定字母的小写形式   |
| 8   | [toString](https://www.runoob.com/java/character-tostring.html)()                        |

## 1.9 String类
java中，字符串隶属于String类，它提供了创建和操作字符串的各种方法
可以直接创建字符串，也可以使用类的构造方法来创建：
```
String str = "Runoob";
String str2=new String("Runoob");
```
区别在于**String 创建的字符串存储在公共池中，而 new 创建的字符串对象在堆上** 
String有11中构造方法，其中可以向构造方法提供一个字符数组来构造：
```
char[] helloArray = { 'r', 'u', 'n', 'o', 'o', 'b'}; String helloString = new String(helloArray);
```
注意的点是**String类是不可改变的，创建后值就不能变了**
值得注意的是下面这种情况：
```
String s = "Google";
System.out.println("s = " + s);

s = "Runoob";
System.out.println("s = " + s);

```
它的输出为：
```
Google
Runoob
```
**这并不代表字符串被改变了，实际上`String s`本质上是一个==String对象的引用==。这里只是改变了引用的对象，原来的String对象`Google`本质上还存在于内存中未被销毁**

下面是一些常用的String类访问器方法（获取有关对象的信息的方法）：
返回字符串对象包含的字符数方法`length()`
```
String site = "www.runoob.com"; int len = site.length();
```

连接字符串的方法`concat()`或直接`+`：
```
"Hello,".concat("Runoob!");
"Hello," + " runoob" + "!"

String string1 = "菜鸟教程网址："; 
System.out.println("1、" + string1 + "www.runoob.com");
```

输出格式化字符串的方法`printf()`：
```
System.out.printf("浮点型变量的值为 " + "%f, 整型变量的值为 " + " %d, 字符串变量的值为 " + "is %s", floatVar, intVar, stringVar);
```
创建格式化字符串的方法`format()`：
```
String fs; fs = String.format("浮点型变量的值为 " + "%f, 整型变量的值为 " + " %d, 字符串变量的值为 " + " %s", floatVar, intVar, stringVar);
```

可以对整数、浮点数和字符进行格式化：
**1.对整数进行格式化：%\[index$]\[标识]\[最小宽度]转换方式** 的形式
特殊的格式常以 **%index$** 开头，index**从1开始取值**，表示将**第index个参数**拿进来进行格式化，\[最小宽度]的含义也很好理解，就是**最终该整数转化的字符串最少包含多少位数字**。剩下2个部分的含义：

标识：

-  '-' 在最小宽度内**左对齐**，不可以与"用0填充"同时使用
-  '#' 只适用于8进制和16进制，**8进制时在结果前面增加一个0，16进制时在结果前面增加0x**
-  '+' 结果总是**包括一个符号(一般情况下只适用于10进制，若对象为BigInteger才可以用于8进制和16进制)**
-  ' ' **正值前加空格，负值前加负号**(一般情况下只适用于10进制，若对象为BigInteger才可以用于8进制和16进制)
-  '0' 结果将**用零来填充**
-  ',' 只适用于10进制，每3位数字之间用"，"分隔
-  '(' 若参数是负数，则结果中不添加负号而是用圆括号把数字括起来(同'+'具有同样的限制)

转换方式：

d 十进制   o 八进制   x或X 十六进制
```
System.out.println(String.format("%1$,09d", -3123));
System.out.println(String.format("%1$9d", -31));
System.out.println(String.format("%1$-9d", -31));
System.out.println(String.format("%1$(9d", -31));
System.out.println(String.format("%1$#9x", 5689));
//结果为：
//-0003,123
// -31
//-31
// (31)
// 0x1639　
```

**2.对浮点数进行格式化：%\[index$]\[标识]\[最少宽度]\[.精度]转换方式** 的形式
精度控制的是**小数点后面的位数**
标识：

- '-' 在最小宽度内左对齐，不可以与"用0填充"同时使用
- '+' 结果总是包括一个符号
- ' ' 正值前加空格，负值前加负号
- '0' 结果将用零来填充
- ',' 每3位数字之间用"，"分隔(只适用于fgG的转换)
- '(' 若参数是负数，则结果中不添加负号而是用圆括号把数字括起来(只适用于eEfgG的转换)

转换方式：

- 'e', 'E' -- 结果被格式化为用计算机科学记数法表示的十进制数
- 'f' -- 结果被格式化为十进制普通表示方式
- 'g', 'G' -- 根据具体情况，自动选择用普通表示方式还是科学计数法方式
- 'a', 'A' -- 结果被格式化为带有效位数和指数的十六进制浮点数

**3.对字符进行格式化：**

对字符进行格式化是非常简单的，c表示字符，标识中'-'表示左对齐，其他就没什么了。

## 1.10 StringBuffer 和 StringBuilder 类
当**对字符串进行修改**的时候，需要使用 StringBuffer 和 StringBuilder 类。

和 String 类不同的是，**StringBuffer 和 StringBuilder 类的对象能够被多次的修改，并且不产生新的未使用对象**。也就是对对象本身进行操作而不产生新的对象
StringBuffer是**线程安全的**，支持**同步访问**；而StringBuilder则不支持，但也因此比StringBuffer**要更快**，多数情况下建议采用StringBuilder
然而**在应用程序要求线程安全的情况下，则必须使用 StringBuffer 类**。(但其实大部分需要线程安全的情况下也没必要选择StringBuffer，因为需要的不仅仅是线程安全，还需要锁，所以尽量还是用StringBuilder)

以下是 StringBuilder 类支持的主要方法：

| 序号  | 方法描述                                                                           |
| --- | ------------------------------------------------------------------------------ |
| 1   | public StringBuffer append(String s)  <br>将指定的字符串**追加**到此字符序列。                 |
| 2   | public StringBuffer reverse()  <br> 将此字符序列用其**反转形式**取代。                        |
| 3   | public delete(int start, int end)  <br>移除此序列的子字符串中的字符。                         |
| 4   | public insert(int offset, int i)  <br>**将 `int` 参数的字符串表示形式插入此序列中**。            |
| 5   | insert(int offset, String str)  <br>**将 `str` 参数的字符串插入此序列中**。                  |
| 6   | replace(int start, int end, String str)  <br>使用给定 `String` 中的字符替换此序列的子字符串中的字符。 |

以下列表列出了 StringBuffer 类的其他常用方法：

| 序号  | 方法描述                                                                                           |
| --- | ---------------------------------------------------------------------------------------------- |
| 1   | int capacity()  <br>返回当前容量。                                                                    |
| 2   | char charAt(int index)  <br>返回此序列中指定索引处的 `char` 值。                                             |
| 3   | void ensureCapacity(int minimumCapacity)  <br>确保容量至少等于指定的最小值。                                  |
| 4   | void getChars(int srcBegin, int srcEnd, char[] dst, int dstBegin)  <br>将字符从此序列复制到目标字符数组 `dst`。 |
| 5   | int indexOf(String str)  <br>返回第一次出现的指定子字符串在该字符串中的索引。                                          |
| 6   | int indexOf(String str, int fromIndex)  <br>从指定的索引处开始，返回第一次出现的指定子字符串在该字符串中的索引。                 |
| 7   | int lastIndexOf(String str)  <br>返回最右边出现的指定子字符串在此字符串中的索引。                                      |
| 8   | int lastIndexOf(String str, int fromIndex)  <br>返回 String 对象中子字符串最后出现的位置。                      |
| 9   | int length()  <br> 返回长度（字符数）。                                                                  |
| 10  | void setCharAt(int index, char ch)  <br>将给定索引处的字符设置为 `ch`。                                     |
| 11  | void setLength(int newLength)  <br>设置字符序列的长度。                                                  |
| 12  | CharSequence subSequence(int start, int end)  <br>返回一个新的字符序列，该字符序列是此序列的子序列。                    |
| 13  | String substring(int start)  <br>返回一个新的 `String`，它包含此字符序列当前所包含的字符子序列。                          |
| 14  | String substring(int start, int end)  <br>返回一个新的 `String`，它包含此序列当前所包含的字符子序列。                   |
| 15  | String toString()  <br>**返回此序列中数据的字符串表示形式**。                                                   |

## 1.11 数组
Java语言提供的数组是用来**存储固定大小的同类型元素的**
数组的**声明方法**有两种：
```
dataType[] arrayRefVar;  //首选方法
或者
dataType arrayRefVar[];  //效果相同，目的是让cpp程序员快速上手
```

**数组的创建则需要使用`new`操作符：**
```
arrayRefVar = new dataType[arraySize];
```
创建的流程是：1.使用`dataType[arraySize]`创建了一个数组； 2.把新创建的数组引用赋值给变量`arrayRefVar`
也可以**枚举地声明+创建数组：**
```
dataType[] arrayRefVar = {value0, value1, ..., valuek};
```

**数组变量的声明和创建可以在一个语句中完成：**
```
dataType[] arrayRefVar = new dataType[arraySize];
```

**和cpp的范围for循环一样，java中常用加强型循环来遍历数组，不需要下标操作：**
```
for(type element: array)
{
    System.out.println(element);
}
```
`type element`是定义的一个与数组元素类型相同的循环变量

类似于cpp，**Java中的多维数组也可以视为数组的数组，n维数组的每个元素都是n-1维数组，从左往右数组维度下降**
创建方法：
1. 直接为每一维分配空间，格式如下：

```
type[][] typeName = new type[typeLength1][typeLength2];
```

type 可以为基本数据类型和复合数据类型，typeLength1 和 typeLength2 必须为正整数，**typeLength1 为行数，typeLength2 为列数**。

2. 从最高维开始，分别为每一维分配空间，例如：

```
String[][] s = new String[2][];
s[0] = new String[2];
s[1] = new String[3];
s[0][0] = new String("Good"); 
s[0][1] = new String("Luck"); 
s[1][0] = new String("to"); 
s[1][1] = new String("you");
s[1][2] = new String("!");
```

### Arrays类
包`java.util.Arrays`的类可以更方便地操作数组，**它提供的所有方法都是静态的**：
具有以下功能：

- 给数组赋值：通过 fill 方法。
- 对数组排序：通过 sort 方法,按升序。
- 比较数组：通过 equals 方法比较数组中元素值是否相等。
- 查找数组元素：通过 binarySearch 方法能对排序好的数组进行二分查找法操作。

| 序号  | 方法和说明                                                                                                                                                                                                                |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **public static int binarySearch(Object[] a, Object key)**  <br>用二分查找算法在给定数组中搜索给定值的对象(Byte,Int,double等)。数组在调用前必须排序好的。如果查找值包含在数组中，则返回搜索键的索引；否则返回 (-(_插入点_) - 1)。                                                      |
| 2   | **public static boolean equals(long[] a, long[] a2)**  <br>如果两个指定的 long 型数组彼此_相等_，则返回 true。如果两个数组包含相同数量的元素，并且两个数组中的所有相应元素对都是相等的，则认为这两个数组是相等的。换句话说，如果两个数组以相同顺序包含相同的元素，则两个数组是相等的。同样的方法适用于所有的其他基本数据类型（Byte，short，Int等）。 |
| 3   | **public static void fill(int[] a, int val)**  <br>将指定的 int 值分配给指定 int 型数组指定范围中的每个元素。同样的方法适用于所有的其他基本数据类型（Byte，short，Int等）。                                                                                           |
| 4   | **public static void sort(Object[] a)**  <br>对指定对象数组根据其元素的自然顺序进行升序排列。同样的方法适用于所有的其他基本数据类型（Byte，short，Int等）。                                                                                                           |

## 1.12 正则表达式
正则表达式定义了字符串的模式，可用于搜索、编辑或处理文本。
Java提供`java.util.regex`包，包含`Pattern`和`Matcher`类来处理正则表达式的匹配操作
其实我们使用的字符串就是一种正则表达式的匹配，`hello world`就匹配字符串"hello world"，但更重要的用法是特定条件下的正则表达式匹配
ava.util.regex 包主要包括以下三个类：

- **Pattern 类：**
    
    pattern 对象是一个**正则表达式的编译表示**。Pattern 类**没有公共构造方法**。要创建一个 Pattern 对象，你必须**首先调用其公共静态编译方法**，它返回一个 Pattern 对象。**该方法接受一个正则表达式作为它的第一个参数**。
    
- **Matcher 类：**
    
    Matcher 对象是**对输入字符串进行解释和匹配操作的引擎**。与Pattern 类一样，Matcher **也没有公共构造方法**。你需要**调用 Pattern 对象的 matcher 方法来获得一个 Matcher 对象**。
    
- **PatternSyntaxException：**
    
    PatternSyntaxException 是一个非强制异常类，它表示一个正则表达式模式中的语法错误。


**正则表达式的语法：**
在其他语言中，\\ 表示：**我想要在正则表达式中插入一个普通的（字面上的）反斜杠，请不要给它任何特殊的意义。**

在 Java 中，\\ 表示：**我要插入一个正则表达式的反斜线，所以其后的字符具有特殊的意义**
因此，**Java中需要两个反斜线`\\`才能起到其他语言一个字面意义上的斜杠的作用**

下面是

| 字符            | 说明                                                                                                                                                                                                                        |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| \             | 将下一字符标记为特殊字符、文本、反向引用或八进制转义符。例如， n匹配字符 n。\n 匹配换行符。序列 \\\\ 匹配 \\ ，\\( 匹配 (。                                                                                                                                                 |
| ^             | 匹配输入字符串开始的位置。如果设置了 **RegExp** 对象的 **Multiline** 属性，^ 还会与"\n"或"\r"之后的位置匹配。                                                                                                                                                 |
| $             | 匹配输入字符串结尾的位置。如果设置了 **RegExp** 对象的 **Multiline** 属性，$ 还会与"\n"或"\r"之前的位置匹配。                                                                                                                                                 |
| *             | 零次或多次匹配前面的字符或子表达式。例如，zo* 匹配"z"和"zoo"。\* 等效于 {0,}。                                                                                                                                                                         |
| +             | 一次或多次匹配前面的字符或子表达式。例如，"zo+"与"zo"和"zoo"匹配，但与"z"不匹配。+ 等效于 {1,}。                                                                                                                                                              |
| ?             | 零次或一次匹配前面的字符或子表达式。例如，"do(es)?"匹配"do"或"does"中的"do"。? 等效于 {0,1}。                                                                                                                                                            |
| {_n_}         | _n_ 是非负整数。正好匹配 _n_ 次。例如，"o{2}"与"Bob"中的"o"不匹配，但与"food"中的两个"o"匹配。                                                                                                                                                           |
| {_n_,}        | _n_ 是非负整数。至少匹配 _n_ 次。例如，"o{2,}"不匹配"Bob"中的"o"，而匹配"foooood"中的所有 o。"o{1,}"等效于"o+"。"o{0,}"等效于"o*"。                                                                                                                            |
| {_n_,_m_}     | _m_ 和 _n_ 是非负整数，其中 _n_ <= _m_。匹配至少 _n_ 次，至多 _m_ 次。例如，"o{1,3}"匹配"fooooood"中的头三个 o。'o{0,1}' 等效于 'o?'。注意：您不能将空格插入逗号和数字之间。                                                                                                    |
| ?             | 当此字符紧随任何其他限定符（*、+、?、{_n_}、{_n_,}、{_n_,_m_}）之后时，匹配模式是"非贪心的"。"非贪心的"模式匹配搜索到的、尽可能短的字符串，而默认的"贪心的"模式匹配搜索到的、尽可能长的字符串。例如，在字符串"oooo"中，"o+?"只匹配单个"o"，而"o+"匹配所有"o"。                                                                  |
| .             | 匹配除"\r\n"之外的任何单个字符。若要匹配包括"\r\n"在内的任意字符，请使用诸如"\[\s\S]"之类的模式。                                                                                                                                                               |
| (_pattern_)   | 匹配 _pattern_ 并捕获该匹配的子表达式。可以使用 **$0…$9** 属性从结果"匹配"集合中检索捕获的匹配。若要匹配括号字符 ( )，请使用"\("或者"\)"。                                                                                                                                   |
| (?:_pattern_) | 匹配 _pattern_ 但不捕获该匹配的子表达式，即它是一个非捕获匹配，不存储供以后使用的匹配。这对于用"or"字符 (\|) 组合模式部件的情况很有用。例如，'industr(?:y\|ies) 是比 'industry\|industries' 更经济的表达式。                                                                                    |
| (?=_pattern_) | 执行正向预测先行搜索的子表达式，该表达式匹配处于匹配 _pattern_ 的字符串的起始点的字符串。它是一个非捕获匹配，即不能捕获供以后使用的匹配。例如，'Windows (?=95\|98\|NT\|2000)' 匹配"Windows 2000"中的"Windows"，但不匹配"Windows 3.1"中的"Windows"。预测先行不占用字符，即发生匹配后，下一匹配的搜索紧随上一匹配之后，而不是在组成预测先行的字符后。     |
| (?!_pattern_) | 执行反向预测先行搜索的子表达式，该表达式匹配不处于匹配 _pattern_ 的字符串的起始点的搜索字符串。它是一个非捕获匹配，即不能捕获供以后使用的匹配。例如，'Windows (?!95\|98\|NT\|2000)' 匹配"Windows 3.1"中的 "Windows"，但不匹配"Windows 2000"中的"Windows"。预测先行不占用字符，即发生匹配后，下一匹配的搜索紧随上一匹配之后，而不是在组成预测先行的字符后。 |
| _x_\|_y_      | 匹配 _x_ 或 _y_。例如，'z\|food' 匹配"z"或"food"。'(z\|f)ood' 匹配"zood"或"food"。                                                                                                                                                       |
| [_xyz_]       | 字符集。匹配包含的任一字符。例如，"[abc]"匹配"plain"中的"a"。                                                                                                                                                                                   |
| [^_xyz_]      | 反向字符集。匹配未包含的任何字符。例如，"[^abc]"匹配"plain"中"p"，"l"，"i"，"n"。                                                                                                                                                                    |
| [_a-z_]       | 字符范围。匹配指定范围内的任何字符。例如，"[a-z]"匹配"a"到"z"范围内的任何小写字母。                                                                                                                                                                          |
| [^_a-z_]      | 反向范围字符。匹配不在指定的范围内的任何字符。例如，"[^a-z]"匹配任何不在"a"到"z"范围内的任何字符。                                                                                                                                                                  |
| \b            | 匹配一个字边界，即字与空格间的位置。例如，"er\b"匹配"never"中的"er"，但不匹配"verb"中的"er"。                                                                                                                                                              |
| \B            | 非字边界匹配。"er\B"匹配"verb"中的"er"，但不匹配"never"中的"er"。                                                                                                                                                                            |
| \c_x_         | 匹配 _x_ 指示的控制字符。例如，\cM 匹配 Control-M 或回车符。_x_ 的值必须在 A-Z 或 a-z 之间。如果不是这样，则假定 c 就是"c"字符本身。                                                                                                                                    |
| \d            | 数字字符匹配。等效于 [0-9]。                                                                                                                                                                                                         |
| \D            | 非数字字符匹配。等效于 [^0-9]。                                                                                                                                                                                                       |
| \f            | 换页符匹配。等效于 \x0c 和 \cL。                                                                                                                                                                                                     |
| \n            | 换行符匹配。等效于 \x0a 和 \cJ。                                                                                                                                                                                                     |
| \r            | 匹配一个回车符。等效于 \x0d 和 \cM。                                                                                                                                                                                                   |
| \s            | 匹配任何空白字符，包括空格、制表符、换页符等。与 [ \f\n\r\t\v] 等效。                                                                                                                                                                                |
| \S            | 匹配任何非空白字符。与 [^ \f\n\r\t\v] 等效。                                                                                                                                                                                            |
| \t            | 制表符匹配。与 \x09 和 \cI 等效。                                                                                                                                                                                                    |
| \v            | 垂直制表符匹配。与 \x0b 和 \cK 等效。                                                                                                                                                                                                  |
| \w            | 匹配任何字类字符，包括下划线。与"[A-Za-z0-9_]"等效。                                                                                                                                                                                         |
| \W            | 与任何非单词字符匹配。与"[^A-Za-z0-9_]"等效。                                                                                                                                                                                            |
| \x_n_         | 匹配 _n_，此处的 _n_ 是一个十六进制转义码。十六进制转义码必须正好是两位数长。例如，"\x41"匹配"A"。"\x041"与"\x04"&"1"等效。允许在正则表达式中使用 ASCII 代码。                                                                                                                      |
| \_num_        | 匹配 _num_，此处的 _num_ 是一个正整数。到捕获匹配的反向引用。例如，"(.)\1"匹配两个连续的相同字符。                                                                                                                                                               |
| \_n_          | 标识一个八进制转义码或反向引用。如果 \_n_ 前面至少有 _n_ 个捕获子表达式，那么 _n_ 是反向引用。否则，如果 _n_ 是八进制数 (0-7)，那么 _n_ 是八进制转义码。                                                                                                                              |
| \_nm_         | 标识一个八进制转义码或反向引用。如果 \_nm_ 前面至少有 _nm_ 个捕获子表达式，那么 _nm_ 是反向引用。如果 \_nm_ 前面至少有 _n_ 个捕获，则 _n_ 是反向引用，后面跟有字符 _m_。如果两种前面的情况都不存在，则 \_nm_ 匹配八进制值 _nm_，其中 _n_ 和 _m_ 是八进制数字 (0-7)。                                                      |
| \nml          | 当 _n_ 是八进制数 (0-3)，_m_ 和 _l_ 是八进制数 (0-7) 时，匹配八进制转义码 _nml_。                                                                                                                                                                 |
| \u_n_         | 匹配 _n_，其中 _n_ 是以四位十六进制数表示的 Unicode 字符。例如，\u00A9 匹配版权符号 (©)。                                                                                                                                                               |
### 1.12太烦了，之后再看

## 1.13 方法
我们之前常用的`System.out.println()`的含义是：
- println() 是一个方法。
- System 是系统类。
- out 是标准输出对象。
也就是说**调用系统类`System`的`out`对象中的方法`println()`

Java方法是语句的集合，它们在一起执行一个功能。
- 方法是解决一类问题的步骤的有序组合
- 方法**包含于类或对象中**
- 方法在程序中被创建，在其他地方被引用

方法的定义需要以下几个要素：
```
修饰符 返回值类型 方法名（参数类型 参数名）{
	方法体
	return 返回值;
}
```

- **修饰符：** 修饰符，这是**可选的**，**告诉编译器如何调用该方法**。定义了该方法的**访问类型**。**修饰符可以有访问修饰符和非访问修饰符多个**
- **返回值类型 ：** 方法可能会返回值。returnValueType 是方法返回值的数据类型。有些方法执行所需的操作，但没有返回值。在这种情况下，returnValueType 是关键字**void**。
- **方法名：** 是方法的实际名称。方法名和参数表共同构成方法签名。
- **参数类型：** 参数像是一个占位符。当方法被调用时，**传递值给参数。这个值被称为实参或变量**。参数列表是指方法的参数类型、顺序和参数的个数。参数是可选的，方法可以不包含任何参数。
- **方法体：** 方法体包含具体的语句，定义该方法的功能。

特别地，像下面这样的静态方法：
```
static void staticCounter() {
...
}
或
public static void main(String[] args) {
MyClass t1 = new MyClass( 10 ); MyClass t2 = new MyClass( 20 ); System.out.println(t1.x + " " + t2.x); 
}
```
**属于类而不是某个实例，因此可以在不创建类的实例对象的情况下调用这样的方法（直接通过`类名.方法名`来调用），这些方法的实现与具体的对象没有关系，==无参数情况下只能访问类的`static`成员，不能使用`this`或`super`==**
如果希望在静态方法中访问实例成员，那就需要为其传递一个该类型对象的引用的参数或者在方法内`new`一个该类型对象：
```
public class Example{
static int staticValue = 2;
private int instanceValue = 1;

static int setTwiceValue(Example example) {
	return example.instanceValue * staticValue;
}

static int setValue(int i) {
	Example example = new Example();
	return example.instanceValue * staticValue * i;
}
}
```
值传递和引用传递的操作和cpp差不多，甚至更方便
方法重载、参数作用域的规则也和cpp一样

**一旦我们定义了自己的构造方法，默认构造方法就会失效。Java没有cpp的\`=default\`  这样的语法来要求编译器生成默认构造方法，我们得自己写无参数构造方法（大多数情况还是需要有参数的构造方法的）**
## 1.14 命令行参数与可变参数与析构finalize()方法
### 命令行参数
**如果我们希望在命令行中运行一个可执行程序时向其传递一些信息，这可以通过向`main()`方法传递命令行参数实现，==命令行参数是在命令行中调用可执行程序时跟在程序名后面的那些参数==**
它们被存储在在main方法的`String[] args`数组中，按照命令行中靠近程序名字，按空格分开的顺序依次存储进数组的各个位置
下面是打印出所有命令行参数的例子：
```
public class CommandLine {
   public static void main(String[] args){ 
      for(int i=0; i<args.length; i++){
         System.out.println("args[" + i + "]: " + args[i]);
      }
   }
}
```
命令行中是这样的：
```
$ javac CommandLine.java 
$ java CommandLine this is a command line 200 -100
args[0]: this
args[1]: is
args[2]: a
args[3]: command
args[4]: line
args[5]: 200
args[6]: -100
```

### 可变参数
JDK 1.5 开始，Java支持传递同类型的可变参数给一个方法：**可变参数是一种允许方法接受可变数量的参数的机制**
其声明如下所示：
```
typename... parameterName
```
也就是**在指定参数类型后面加上省略号`...`
一个方法只能指定一个可变参数，并且它必须是方法浮点==最后一个参数==，在它之前的参数都是普通参数声明**
```
public static void printMax( double... numbers) { 
	if (numbers.length == 0) { 
		System.out.println("No argument passed"); 
		return; }
```
**传入的可变参数本质上是==该类型参数的一个数组==，即`typename[] parameterName`，我们直接将它当成数组来用即可**
**重载方面，比起具有可变参数的方法，会==优先匹配具有固定参数的方法**==

### finalize()方法
Java 允许定义这样的方法，它**在对象被垃圾收集器析构(回收)之前调用**，这个方法叫做 finalize( )，**它用来清除回收对象**。
例如，在一个需要打开文件的对象中编写finlize()方法来确保对象打开的文件在它被析构前被正确关闭。
**其实就是cpp的析构函数**
一般**会将其声明为`protected`来确保不会被类以外的代码调用**：
```
protected void finalize()
{
   // 在这里终结代码
}
```
**Java的内存回收比cpp简单，它可以由JVM字段完成，当然你也可以手动用这样的析构函数来完成**

## 1.15 Java的流、文件和IO
`java.io`包几乎包含了**所有操作输入输出所需要的类**。因此通常需要在使用它们时声明`import java.io.*`

一个流可以理解为一个数据序列，输入流表示从一个源中读取数据，输出流表示向一个目标写数据

这是描述Java的输入输出流的类层次图，用以提供一个简单的印象：
![[Pasted image 20240916161337.png]]
### 1.15.1 控制台输入输出
**控制台输入**：
**Java 的控制台输入由 System.in 完成。**
为了获得**绑定到控制台的字符流**，我们需要使用到BufferedReader类

Reader对象：是java的IO系统中的抽象类，用于读取==**字符流**==。**它是所有字符输入流的基类，提供了基本的读取字符、数组和行数据的方法，定义了操作字符流的基本接口**

**System.in是InputStream类的一个==引用==，它是字节流类**

**BufferedReader类**：
BufferedReader类是Java标准库用于高效读取文本数据的一个类，它**提供了一种缓冲机制来提高输入效率**，BufferedReader类的构造方法允许我们接受一个**Reader对象**，绑定到这个源上为其提供缓冲区来进行读取
可以通过下面的方式声明一个BufferedReader对象，并绑定到System.in上：
```
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
```
注意：System.in其实是一个字节流InputStream对象，这里InputStreamReader作为一个桥接类，将字节流转换为字符流

**从控制台读取字符**：
创建这样一个BufferedReader对象后，我们就可以使用它的方法来进行字符读取了：
从 BufferedReader 对象读取一个字符要使用 **read() 方法**，它的**原型**如下：
```
int read() throws IOException
```
后面的` throws IOException`是异常处理，抛出IOException进行报告
该方法**在每次调用时从输入流读取一个字符并把该字符作为==整数值==返回**
我们可以通过强制类型转换`(char) br.read()`来把整数值转换回字符
read()方法通常与while、do-while语句结合使用，以从控制台不断地读取单个字符。使用时直接`int i = br.read()`储存在变量中即可
下面是一个使用的例子，从控制台读取字符并且在读到'q'时结束读取：
```
//使用 BufferedReader 在控制台读取字符
 
import java.io.*;
 
public class BRRead {
    public static void main(String[] args) throws IOException {
        char c;
        // 使用 System.in 创建 BufferedReader
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        System.out.println("输入字符, 按下 'q' 键退出。");
        // 读取字符
        do {
            c = (char) br.read();
            System.out.println(c);
        } while (c != 'q');
    }
}
```


**从控制台读取字符串**：
BufferedReader类还提供了`readline()`方法来从控制台读取字符串，其原型为：
```
String readLine( ) throws IOException
```
和read()行为差不多，只是读取字符串，返回String，下面是一个使用例：
```
//使用 BufferedReader 在控制台读取字符串
import java.io.*;
 
public class BRReadLines {
    public static void main(String[] args) throws IOException {
        // 使用 System.in 创建 BufferedReader
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        String str;
        System.out.println("Enter lines of text.");
        System.out.println("Enter 'end' to quit.");
        do {
            str = br.readLine();
            System.out.println(str);
        } while (!str.equals("end"));
    }
}
```

控制台输入还可以通过`java Scanner`类获取，这会在1.16节讲到

**控制台输出：**
在此前已经介绍过，**控制台的输出由 print( ) 和 println() 完成**。这些方法都由**类 PrintStream** 定义，**System.out 是该类对象的一个==引用==**。

print( ) 和 println()的区别在于**println()会在输出内容时在末尾自动换行，print()则不会**
PrintStream 继承了 OutputStream类，并且实现了**方法 write()**。这样，**write() 也可以用来往控制台写操作**
其原型为：
```
void write(int byteval)
```
**该方法将 byteval 的==低八位字节==写到流中**。
因为char就正好占1字节8位，存储进int的低8位中，**所以writer()方法可以用于输出单个字符**。类似于cpp的put()

### 1.15.2 文件读写
下面将要讨论的两个重要的流是 **FileInputStream** 和 **FileOutputStream**。
**FileInputStream:**
该流用于从文件读取数据，它的对象可以用关键字`new` 来创建。属于InputStream类的对象
可以使用**字符串类型的文件名**来创建一个输入流对象来读取文件：
```
InputStream f = new FileInputStream("C:/java/hello");
```
也可以传入一个文件对象`File`来创建文件输入流，这需要先使用File的构造函数创建一个文件对象：
```
File f = new File("C:/java/hello");
InputStream in = new FileInputStream(f);
```

创建InputStream对象后，我们就可以使用下面的方法来进行文件读取了，下面是这些方法的原型：

| **序号** | **方法及描述**                                                                                                                         |
| ------ | --------------------------------------------------------------------------------------------------------------------------------- |
| 1      | **public void close() throws IOException{}**  <br>**关闭此文件输入流并释放与此流有关的所有系统资源**。抛出IOException异常。                                    |
| 2      | **protected void finalize()throws IOException {}**  <br>这个方法**清除与该文件的连接**。确保在不再引用文件输入流时调用其 close 方法。抛出IOException异常。              |
| 3      | **public int read(int r)throws IOException{}**  <br>这个方法从 InputStream 对象**读取指定字节的数据**。**返回为整数值**。**返回下一字节数据，如果已经到结尾则返回-1**。可以没有参数 |
| 4      | **public int read(byte[] r) throws IOException{}**  <br>这个方法从输入流读取r.length长度的字节。返回读取的字节数。如果是文件结尾则返回-1。                            |
| 5      | **public int available() throws IOException{}**  <br>返回下一次对此输入流调用的方法可以不受阻塞地从此输入流读取的字节数。返回一个整数值。                                   |

**FileOutputStream:**
该类用来创建一个文件并向文件中写数据。如果该流在打开文件进行输出前，目标文件不存在，那么该流会创建该文件。
它有两种创建方法：
使用字符串类型文件名来创建：
```
OutputStream f = new FileOutputStream("C:/java/hello")
```

使用文件对象创建：
```
File f = new File("C:/java/hello");
OutputStream fOut = new FileOutputStream(f);
```

它的流操作有：

| **序号** | **方法及描述**                                                                                                            |
| ------ | -------------------------------------------------------------------------------------------------------------------- |
| 1      | **public void close() throws IOException{}**  <br>**关闭此文件输入流并释放与此流有关的所有系统资源**。抛出IOException异常。                       |
| 2      | **protected void finalize()throws IOException {}**  <br>这个方法**清除与该文件的连接**。确保在不再引用文件输入流时调用其 close 方法。抛出IOException异常。 |
| 3      | **public void write(int w)throws IOException{}**  <br>这个方法把**指定的字节写到输出流**中。                                          |
| 4      | **public void write(byte[] w)**  <br>**把指定数组中w.length长度的字节写到OutputStream中。**                                         |

# Chapter 2. 课程内的Java补充
Java在编程方面的优势：更容易捕捉到错误，便于debug；几乎不会出现类型方面的错误；代码的可读性更强，运行的效率更高，**不需要做开销昂贵的运行时检查**
劣势在于代码冗长，同时数量巨大的包和类型使得它的代码不那么泛用，为此有一个名为generics的方法进行应对
关于Java的编译方式:
为什么需要让javac编译器生成一次`.class`文件？一方面是`.calss`以及经过了类型检查，这样的分布式代码更加安全；同时`.class`文件更容易让代码执行，而且还不必担心被人反向推出原码，很安全。

## 2.1 交互式的调试工具
在本门课程中，程序往往会很大，这时通过一个个设置断点，输出结果就会非常困难，因此我们会来尝试使用IntelliJ's的交互式调试工具

# Lecture 3. 列表.1
Java中的数组是静态数组，其大小是固定的，而如果我们需要一些**动态大小的数组**，这就是我们接下来要构建的**列表**的工作了
列表类需要有两个数据成员：一个称为`first`的int成员和一个列表的引用成员
```
package lec3_list1:
public class Intlist {
	public int first;
	public Intlist rest;
}
```
正如其名，**`first`成员是列表的第一个元素，而`rest`成员是列表除第一个元素以外的其他元素组成的数组**
那么我们就可以通过类似这样的操作来不断地申请空间：
```
public class IntList { 
public int first; 
public IntList rest;
public static void main(String[] args){
	IntList L = new IntList();
	L.first = 5;
	L.rest = new IntList();

	L.rest.first = 10;
	L.rest.rest = new IntList();
	L.rest.rest.first = 15;
	L.rest.rest.rest = null;
```
这就形成了这样的一个结构：
![[Pasted image 20240917195846.png]]
把这种思想写到构造方法中：
```
public class IntList { 
public int first; 
public IntList rest;
public IntList(int f, IntList r) {
	first = f;
	rest = r;
}

//倒着从最后一节开始构造一共三节的列表
public static void main(String[] args) {
	IntList L = new IntList(15, null);
	//创建第三节列表

	L = new IntList(10, L);
	//创建第二节列表
	L = new IntList(5, L);
	//第一节...
}
}
```
这就是最基础的列表了，但是如果我们希望在其基础上进行索引、项的增删、列表的大小等功能，那就需要添加更多方法：
```
public class IntList { 
public int first; 
public IntList rest;
public IntList(int f, IntList r) {
	first = f;
	rest = r;
}

//递归返回列表的大小
public int size() {
	if (rest == null){
		return 1; }
	retrun 1 + this.rest.size();  //递归调用。基准情况就是rest == null
}

//递归返回列表的第i项元素
public int get(int i) {
	if (i == 0) {
		return first; }
	return rest.get(i-1);   //相当于每次把i--，然后在被减为0的时候到达基准情况，也就是索引为i的那个rest的first成员
}
}
```

# Lecture 4. 列表.2：进化单链表
上一讲中的列表是**裸递归实现**的，在Java中人们通常不会使用这样的列表类，不符合使用习惯，因此我们需要对其进行一些改进，首先是改名：
```
package lec3_lists1;

public class IntNode { 
public int item; 
public IntNode next;
public IntNode(int i, IntNode n) {
	item = i;
	next = n;
}
```
这就是我们的一个**列表结点**的结构了，下面我们在**同一个包内**写一个**列表结构**：
```
package lec3_lists1;

public class SLList {
	public IntNode first;  //建造第一个结点

//用一个值x，创建一个新列表
	public SLList(int x){
		first = new IntNode(x, null); 
		}
}
```
这样做的意义在于**将构造的递归细节放到了单链表的构造函数中进行隐藏，不必让我们在使用时还需要去思考递归的问题**
这就是本讲需要做到的工作，接下来我们需要把其他的方法也使用这样的**包装**：
```
package lec3_lists1;

public class SLList {
	private IntNode first;  //第一个结点，避免用户访问

//用一个值x，创建一个新列表
	public SLList(int x){
		first = new IntNode(x, null); 
		}

//把值x加入到链表的最开头（first）
	public void addFirst(int x){
		first = new IntNode(x, first); //直接新建一个结点，让其的next指针指向当前的first，再把当前的first转交给该结点
		}

//获取首结点存储的值
	public int getFirst() {
		return first.item;  }
}
```
另一件事是我们希望避免用户触碰到数据结构较为底层的一些成员，从而扰乱我们的结构，因此我们把first成员设置为`private`避免访问
实际上我们的`private`并不能完全保证成员的安全，Java的`reflaction`库就提供了访问私有成员的方法，它的更多的作用在于提醒用户“如果你无法保证自己的行为是安全的，最好别试图乱动它”

我们可以发现列表结点类并不会被独立使用，那么我们为了简化操作和引入，完全可以将结点类写为单链表类的一个**嵌套类并且把它设置为私有以防用户自己申请和修改一个结点类**：
```
package lec3_lists1;

private class SLList {
	public class IntNode { 
		public int item; 
		public IntNode next;
		public IntNode(int i, IntNode n) {
			item = i;
			next = n;
}
	private IntNode first;  //第一个结点，避免用户访问

//用一个值x，创建一个新列表
	public SLList(int x){
		first = new IntNode(x, null); 
		}

//把值x加入到链表的最开头（first）
	public void addFirst(int x){
		first = new IntNode(x, first); //直接新建一个结点，让其的next指针指向当前的first，再把当前的first转交给该结点
		}

//获取首结点存储的值
	public int getFirst() {
		return first.item; 
		 }
}
```
和之前说的那样，嵌套类虽然有时候会破坏代码的结构性，但是对于类的组织来说比较清晰，视情况可以适当使用。
下面我们来为单链表类尾后插入结点。这需要一个搜索来到达链表的尾部：
```
public void addLast(int x) {
	IntNode p = first;
	//搜索结点，直到找到链表的最后
	while (p.next != null) {
		p = p.next; }
	p = new IntNode(x, null); 
	//在尾后新建结点
}
```

而由于我们将结点和链表分为了两个类，我们的`size()`方法似乎无法单纯地使用递归求出，因为两个类的成员不一样，无法在链表类中调用`next`
为了保持我们递归的良好传统，一个方法是**为链表类加入一个private的`size(IntNode p)`方法，它接受一个结点类的对象，返回从这个结点开始的所有往后结点的个数**，它根据结点类的对象的`next`成员进行递归：
```
private int size(IntNode p) {
	if (p.next == null)
		return 1;
	return 1 + size(p.next);
}
```
这样我们在公共接口中就可以使用自己的`first`结点成员进行递归计算了：
```
public int size() {
	return size(first);
}
```
因为private的`size`方法有参数而外层没有，这形成了重载

虽说保持了递归的传统，但是很显然，因为**私有的size方法一直在扫描链表（一个个结点的next），它的时间复杂度是线性的，过慢了**，所以我们需要找到优化的方案：
我们如果在链表的其他操作中维护一个保存大小的成员，而不是每一次都从头进行扫描，这样就可以根据链表的更新实时记录其大小，不必从头开始进行扫描了：
```
package lec3_lists1;

private class SLList {
	public class IntNode { 
		public int item; 
		public IntNode next;
		public IntNode(int i, IntNode n) {
			item = i;
			next = n;
}
	private IntNode first;  //第一个结点，避免用户访问
	private int size;  //维护一个链表大小成员

//用一个值x，创建一个新列表
	public SLList(int x){
		first = new IntNode(x, null); 
		size = 1;
		}

//把值x加入到链表的最开头（first）
	public void addFirst(int x){
		first = new IntNode(x, first); //直接新建一个结点，让其的next指针指向当前的first，再把当前的first转交给该结点
		size += 1;
		}

//获取首结点存储的值
	public int getFirst() {
		return first.item; 
		 }

	public void addLast(int x) {
	IntNode p = first;
	//搜索结点，直到找到链表的最后
	while (p.next != null) {
		p = p.next; }
	p = new IntNode(x, null); 
	//在尾后新建结点
	size += 1;
	}

	public int size() {
	return size;  //直接返回大小成员就好
	}

}
```
经典的空间换时间思想，让我们的方法快了很多。
这也是包装后的链表比起裸递归结点的好处了：有个很好的地方来存放这些维护数据的成员

还有一个问题是，当我们有一个空链表时，如果我们希望在尾后添加一个结点，按照上面的代码，程序会崩溃，这是因为在`addLast`方法中，我们使用的`IntNode p = first;`这句话的**p在空链表中为一个空指针**，我们无法对它使用while语句中的`p.next == null`的检查，因为你**无法对一个空指针使用点运算符**
所以我们需要略微地修改一下代码，特殊判定一下`first == null`的情况：
```
public void addLast(int x) {
	IntNode p = first;

	if (first == null) {
		p = new IntNode(x, null);
		return;
	}
	//搜索结点，直到找到链表的最后
	while (p.next != null) {
		p = p.next; }
	p = new IntNode(x, null); 
	//在尾后新建结点
	size += 1;
	}

	public int size() {
	return size;  //直接返回大小成员就好
	}
```
这还是太长了，太长的代码不利于维护，尤其是有大量的特殊判断时，如果数据结构很复杂，那就更加麻烦了。
因此我们不妨从构造函数开始，使得**构造一个空链表实际上是构造一个没有值的“哨兵结点”（也就是cpp说的“头指针”）**，然后如果是新建第一个结点，它实际上**被哨兵结点的next成员指向**：
```
package lec3_lists1;

private class SLList {
	public class IntNode { 
		public int item; 
		public IntNode next;
		public IntNode(int i, IntNode n) {
			item = i;
			next = n;
}
	private IntNode sentinel;  //哨兵结点。first结点是sentinel.next
	private int size;  //维护一个链表大小成员

//创建一个空列表。这里通过哨兵结点提供更安全的策略
	public SLList(int x){
		sentinel = new IntNode(0, null);  //第一个参数这个值是多少都无所谓，因为哨兵结点的值没有用
		size = 0;
		}

	//用一个值x，创建一个新列表。第一个结点在哨兵结点后面
	public SLList(int x){
		sentinel.next = new IntNode(x, null);  //这个值是多少
		size = 1;
		}

//把值x加入到链表的最开头（first）
	public void addFirst(int x){
		sentinel.next = new IntNode(x, first); //直接新建一个结点，让其的next指针指向当前的first，再把当前的first转交给该结点。first是哨兵结点的next指向对象
		size += 1;
		}

//获取首结点存储的值
	public int getFirst() {
		return sentinel.next.item; 
		 }

	public void addLast(int x) {
	IntNode p = sentinel; //把p改成了sentinel，这样可以进行next成员的访问，而如果是空列表，next成员就是空，我们就可以直接结束while循环了！
	
	//搜索结点，直到找到链表的最后
	while (p.next != null) {
		p = p.next; }
	p = new IntNode(x, null); 
	//在尾后新建结点
	size += 1;
	}

	public int size() {
	return size;  //直接返回大小成员就好
	}

}
```
这很清晰地展现了头指针的作用，`sentinel.next`指向第一个结点是恒成立的，这称为**不变式**，它们可以用于推理代码：**假设所想的情况都是成立的，提供一层安全保护**

# Lecture 5. 列表.3：进化双链表
## 5.1 双链表的思想与泛型编程
不难发现，在上一讲实现的单链表尾部添加结点太慢了，它需要扫描过整个列表才能添加结点。我们希望**我们可以无需扫描就能到达链表的结尾**
我们希望对尾部进行结点的添加、删除、获取能更快一些，**若仅仅添加一个“尾后指针”来仿照头指针的工作**，思考一下，如果**删掉了尾部的结点**，我们需要将**尾后指针指向的对象变为倒数第二个结点**，这又需要一次扫描列表来获取，显然是不足以让它变快的。
因此，我们不妨构想一个链表：**它的每个结点不仅有指向后一结点（后继）的指针，还有指向前一结点（前驱）的指针，配合上尾部指针，这就是一个==双链表==了**，这样我们的工作就会轻松不少

但是要思考一下，如果有尾部指针，它有时会指向没有值的哨兵结点，有时又会指向一个实值结点，这又需要进行**特殊判定**，我们希望把它统一到一个和谐的形态，这显然可以**在尾后增加一个对称的哨兵结点**来解决
但是还有另一种解决方法可能更好：我们还是**只设一个哨兵结点**，然后来建立一个**循环链表：如果只有一个哨兵，令哨兵结点的后继就是哨兵自己，哨兵结点的前驱也是自己**，然后在链表建立的过程中，**让哨兵结点的前驱指向链表的尾结点，后继指向链表的第一个结点**，我们会发现这样完美地解决了前面的问题

最后一个问题是我们的链表目前为止只能全部存储一个基本类型的值int，如果**我们希望它能够存储另一种类的数值，例如String**，我们的Java为其提供了**泛型编程**的方法，和cpp的泛型编程是一个道理的
Java泛型编程（模板）的语法如下：
```
public class SLList<Pineapple> {  //这里的Pineapple名字可以随便选，只是一个类型的占位符，用尖括号括起来
	private class IntNode {
		public Pineapple item;  //把需要泛型编程的地方类型改成占位符
		public IntNode next;
	}
	......省略
}

使用时，我们直接在类名后面的尖括号加上我们希望创建的类名。如果是内置类型，需要使用包装类：
SLList<Integer> L2 = new SLList<Integer>();
```
不过这样还是不支持放置各种各样的类型元素，这方面的内容在以后可能会讲到

## 5.2 调试
我们有时无法100%确定自己的代码是否完全正确，此时就应该编写一些测试来调试我们的代码。
我们的测试可以写在写代码之前，也就是先写好测试再进行其他代码的编写，这个好处就是在代码没改正之前测试的结果一直无法通过

### 5.2.1 使用Google Truth库
Google Truth库的语法很像自然的语言，例如当我们已知某个函数的期望输出，可以通过下面的方法来进行是否满足期望输出的测试：
```
	public void testVariance() {
		double[] input = {10, 20, 30, 40};
		double expected = 125.0;
		double actual = Variance.variance(input);
		
		assertThat(actual).isEqualto(expected);  //测试实际输出是否等于期望  
		}
```
但是写好了测试之后，如果我们希望**能够不写一个主方法就能运行程序来进行测试**，可以使用下面的语法：
在写好的测试函数上方使用`@Test`，这样就会在该行的下一行旁边增加一个运行符号
![[Pasted image 20240923205721.png]]
该运行符号只适用于一个函数，用于单独运行该测试函数，或是在一起运行时单独指示该函数的测试结果，所以**每一个测试函数的前面都应该带上一个`@Test`**
当有多个这样的测试函数时，运行整个程序之后便会得到类似如下的报告：
![[Pasted image 20240923210859.png]]
意味着测试函数的通过与否
这样的测试函数完全可以写在我们写好希望对其进行测试的函数之前，写好测试函数后，函数的编写变成了“让所有失败的测试变成绿灯”的“小游戏“

而且，对对应的测试条目右键，选择”debug"选项：
![[Pasted image 20240923211722.png]]
就可以单独debug出该测试中出问题的行数及错误，更方便地进行纠错

# Lecture.6 列表.4：使用任意访问的数组
链表结构只支持线性访问，因此对感兴趣的项进行搜索只能遍历，这大大影响了程序的性能。我们希望摆脱链表结点之间的连接方式，使得搜索的过程更快
我们使用数组就可以实现随机访问，很快就能进行给定索引的存储值访问，这与硬件中位的存储模式有关，这里暂不讨论
那么我们可以**使用固定长度的数组结构来构建一个可动态延伸的列表结构**，这和单链表有些类似，只是底层实现不同

在链表类内部，我们**使用一个整型数组的成员来进行元素的存储，而不是使用一个结点类成员**：
```
public class AList {
	private int[] items;
	int size;

	public AList() {
	items = new int[100];
	size = 0;
	}
}
```
获取元素和向数组内增加元素的操作直接使用数组操作即可：
![[Pasted image 20240923215045.png]]
这使用到了很多恒等的不变量，例如最后一个元素的索引为`size-1`等
更有趣的是从列表中取出元素和删除的操作。
从列表的尾部取出一个元素，首先size必须减少。如果我们希望”真的将该元素从列表中删去“，这得把它后面的所有空着的元素格向前移一格，然后新分配一个空间，太过麻烦了
实际上，我们使用时并不关心一个元素是否被真的删去了，我们关心的是使用者的行为是否和我们期望的一样，而且性能是否良好
因此我们不妨直接让size减一，这样尾部的元素就不在我们可以访问的0~size-1的范围内了，再下次往链表尾部添加元素的时候会将该位置上的值覆盖，size加回1

而当我们的底层数组用尽之后，我们希望无限扩展的数组列表可以分配新的空间来储存更多的新的值，这时我们得新申请一个更大的数组，把原来的数组元素复制到对应元素，就变成了一个更大的数组，复制元素部分可以简单地调用`System.arraycopy`方法，最后让items指向新的数组。原来的数组会被自动销毁
```
private void resize(int capacity) {
	int[] a = new int[capacity];
	System.arraycopy(items, 0, a, 0, size);
	items = a;
}
```
一个个递增地申请更大的数组实在是太麻烦了，我们可以在每次需要申请更大的数组时，申请是原数组两倍大小的更大的数组，这样就减轻了多次申请带来的效率减缓
而当我们的数组被扩大到很大的地步，如果我们希望删除其中的大部分，只保留一小部分，此时比起将size减去很大的数来浪费空间，我们不如申请一个更小的数组，把保留的一小部分放入新申请的数组中

Java要求我们在泛型编程时，**不能使用类型占位符来申请一个该占位符类型的数组**，此时需要使用一些魔法：将数组申请时的类型改为`Object`，然后在前面加上`(占位符[])`，例如：
```
items = (Pineapple[]) new Object[100];
```
虽然这还是会报警告，但我们可以这么申请了

# Lecture 7. 继承.1
