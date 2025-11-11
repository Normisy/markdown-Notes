# 编写makefile
## Chapter 1. 简单介绍makefile
在计算机组成原理中，我们知道一个**可执行代码文件p**的生成流程是：
-   
    首先，**==C预处理器==扩展源代码，插入所有用==#include==命令指定的文件，并扩展所有用==#define==声明的宏**
    
- 其次，**==编译器==产生两个源文件p1.c和p2.c的==汇编代码==p1.s和p2.s**
    
- 然后，**==汇编器==将汇编代码转换成==二进制目标代码文件==p1.o和p2.o**。  
    （**==目标代码==** 是机器代码的一种形式，它包含所有指令的二进制表示，但还没有填入 **==全局值的地址==**）
    
- 最后，**==链接器==将两个目标代码文件与实现==库函数==（比如printf）的代码合并，并产生最后的可执行代码文件p，由命令行指示符-o p指定  
    ==可执行代码==是机器代码的第二种形式，它是处理器执行的代码格式**

不难发现，**在生成可执行文件的过程中，文件与文件直接有一种==依赖关系==: 我们首先要把源代码==编译==为中间代码文件，windows下后缀为`.obj`或`.o`，然后把大量的中间代码文件==链接==为可执行文件**。
大多数时候，由于源代码和生成的中间代码文件太多，而链接时有需要明显指出目标中间代码名，非常不便于编译，因此**需要为中间代码文件打包，生成”==库文件==“，后缀为 `.lib`（windows下）**
编译时，**编译器只检测程序语法和函数、变量是否被==声明==。如果函数未被声明，它会给出一个警告，但能继续生成中间代码文件。链接时，链接器会在所有中间代码文件中寻找函数的实现，如果找不到就会报错链接错误码**

makefile就是供**make命令**执行的文件，**它告诉make命令该如何去编译和链接程序**

## 1.1 使用规则
编写makefile时，我们告诉make命令编译和链接c文件与头文件的方法应该遵循以下规则：
* **1. 如果这个==工程==没有编译过，那么==所有的c文件==都需要被编译并链接**
* **2. 如果该工程的某几个c文件被==修改==，那么只编译==被修改的c文件==，并链接目标程序**
* **3. 如果该工程的==头文件==被修改，那么需要编译==引用了这些头文件的c文件==，并链接目标程序**

**粗略地看看makefile的编写规则吧：**
```
target ...: prerequisites ...
	recipe
	...
	...
```
* **target可以是==一个==目标文件、可执行文件或标签。标签特性在后面会叙述**
* **prerequisites 则是生成该target==所依赖的文件、target==（可以是多个混合）**
* **recipe前面必须跟一个`tab`，它是该target需要执行的任意==shell命令==**

它体现了之前我们说的**依赖关系：target这一个或多个的目标文件依赖于prerequisites中的文件或target生成，它们的生成规则定义在recipe中的指令里**。
也就是说，==**如果prerequisites中有一个以上的文件比target中的文件要新，recipe定义的命令就会被执行**==，这就是makefile的核心内容

## 1.2 示例与make工作流程
下面是最原始（不含任何自动推导）的一个示例：
```
edit : main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
    cc -o edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o

main.o : main.c defs.h
    cc -c main.c
kbd.o : kbd.c defs.h command.h
    cc -c kbd.c
command.o : command.c defs.h command.h
    cc -c command.c
display.o : display.c defs.h buffer.h
    cc -c display.c
insert.o : insert.c defs.h buffer.h
    cc -c insert.c
search.o : search.c defs.h buffer.h
    cc -c search.c
files.o : files.c defs.h buffer.h command.h
    cc -c files.c
utils.o : utils.c defs.h
    cc -c utils.c
clean :
    rm edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
```
这里的反斜杠'\\'只是换行，便于阅读。
其中的内容可以**保存在名字为"makefile"或"Makefile”（无后缀）的文件中**，可以使用VS code编写，然后在**该文件所在目录下**直接输入命令：
```
make
```
即可生成，如果不是这样的命名，则需要`make xxx`这样直接给出
看到上面的内容，我们看见**目标文件target的位置上的文件**有：**可执行文件edit和一系列中间目标文件（xxx.o）**；而**依赖文件prerequisites**的位置上：**在可执行文件后面是一系列中间目标文件，而中间目标文件的后面则是同名的.c文件和头文件.h**
**依赖关系的实质就是说明目标文件是由那些文件生成（更新）的**
后续的recipe行**定义了如何生成目标文件的==操作系统命令==（在对应bash下应用对应的命令），必须以一个缩进为开头**

**make命令并不管命令是这么工作的，它会比较targets和prerequisites文件的修改日期，如果==prerequisites文件比targets要更新修改、或者target不存在==，make就会执行后续定义的命令。**
而最后的`clean`并不是一个文件，它只是一个**动作名字**，**当冒号后面什么都没有时**，make不会自动去找它的依赖性，**也不会自动执行其后定义的命令：需要明显指出它的名字才能执行**：
```
make clean
```

总而言之，make的工作流程是：
1. make会在当前目录下找名字叫“Makefile”或“makefile”的文件。
    
2. 如果找到，它会找文件中的第一个目标文件（target），在上面的例子中，他会找到“edit”这个文件，并把这个文件作为最终的目标文件。
    
3. 如果**edit文件不存在**，或是edit**所依赖的后面的 `.o` 文件的文件修改时间要比 `edit` 这个文件新**，那么，他就会执行后面所定义的命令来**生成** `edit` 这个文件。
    
4. 如果 `edit` 所依赖的 `.o` 文件也不存在，那么make会在当前文件中找目标为 `.o` 文件的依赖性，如果找到则再根据那一个规则生成 `.o` 文件。（这有点像一个堆栈的过程）
    
5. 当然，你的C文件和头文件是存在的啦，于是**make会生成 `.o` 文件，然后再用 `.o` 文件生成make的终极任务，也就是可执行文件 `edit` 了**

于是在我们编程中，如果这个工程已被编译过了，当我们修改了其中一个源文件，比如 `file.c` ，那么根据我们的依赖性，我们的目标 `file.o` 会被重编译（也就是在这个依性关系后面所定义的命令），于是 `file.o` 的文件也是最新的啦，于是 `file.o` 的文件修改时间要比 `edit` 要新，所以 `edit` 也会被重新链接了。同理修改了头文件的话，所有用到该文件的文件都会被重编译，然后重链接形成可执行文件。

## 1.3 让makefile更好
### 1.3.1 在makefile中使用变量
在上面的示例中，我们看到edit的规则：
```
edit : main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
    cc -o edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
```
这里我们打了两遍字符串“edit main.o kbd.o command.o display.o \insert.o search.o files.o utils.o”
如果后面需要修改的话，若makefile很复杂，那么改起来太不方便了，此时我们希望**使用变量（即C语言的宏）来让维护更方便**
声明变量的方式很简单，名字任取，例如：
```
objects = main.o kbd.o command.o display.o \
     insert.o search.o files.o utils.o
```
这样一来，我们就可以用`$(objects)`的形式来使用这样的变量了：
```
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)
```
要修改字符串，只需要对objects变量进行修改即可。

### 1.3.2 让make自动推导
**GNU make具有很强的自动推导能力：它可以自动推导文件以及文件依赖关系后面的命令，于是我们就没必要去在每一个 `.o` 文件后都写上类似的命令**
**只要make看到一个`.o`文件，它就会自动把相应`.c`文件加到依赖文件处，无需我们自己写。**
同时，**指令`cc -c whatever.c`也会一并被推导出，我们的makefile可以变得很简洁**：
```
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)

main.o : defs.h
kbd.o : defs.h command.h
command.o : defs.h command.h
display.o : defs.h buffer.h
insert.o : defs.h buffer.h
search.o : defs.h buffer.h
files.o : defs.h buffer.h command.h
utils.o : defs.h

.PHONY : clean
clean :
    rm edit $(objects)
```
这里的`.PHONY`表示`clean`是一个**伪目标文件**，后面会讲到。

当然，**我们也可以把重复的`.h`文件也收拢起来：让使用相同`.h`头文件的文件放在相同的targets上，冒号后面是一个单独的头文件。** 但是这会比较难以看出依赖关系：
```
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)

$(objects) : defs.h
kbd.o command.o files.o : command.h
display.o insert.o search.o files.o : buffer.h

.PHONY : clean
clean :
    rm edit $(objects)
```

### 1.3.3 编写清空目录的规则
**每个makefile中都应该写一个清空目标文件和可执行文件的规则，这是编程的修养**
一般的规则使用`rm`指令进行删除，例如在上例中：
```
clean:
	rm edit $(objects)
```
一个更安全的做法是：
```
.PHONY : clean
clean :
    -rm edit $(objects)
```
这里`-rm`前面的横杠告诉它**可能某些文件会出问题，但不要管，继续执行**。当然，**不要把`clean`规则放在文件开头，它会变成使用`make`指令的默认目标**

## 1.4 make的更进一步介绍
Makefile里主要包含了五个东西：**显式规则、隐式规则、变量定义、指令和注释**。
1. 显式规则。**显式规则说明了如何生成一个或多个目标文件**。这是由**Makefile的书写者明显指出要生成的文件、文件的依赖文件和生成的命令**。
    
2. 隐式规则。由于我们的make有**自动推导**的功能，所以隐式规则可以让我们比较**简略地书写Makefile**，这是由make所支持的。
    
3. 变量的定义。在Makefile中我们要定义一系列的变量，**变量一般都是字符串，这个有点像你C语言中的宏**，当Makefile被执行时，**其中的变量都会被扩展到相应的引用位置**上。
    
4. 指令。其包括了三个部分，一个是**在一个Makefile中引用另一个Makefile，就像C语言中的==include==一样**；另一个是指**根据某些情况指定Makefile中的有效部分**，就像C语言中的预编译#if一样；还有就是**定义一个多行的命令**。有关这一部分的内容，我会在后续的部分中讲述。
   预编译#if：
   ```
   #if 条件  
    // 如果条件为真，则编译这位置上的代码  
   #else  
    //  如果条件为假，则编译这位置上的代码  
   #endif
    ```
    
5. 注释。**Makefile中只有行注释**，和UNIX的Shell脚本一样，**其注释是用 `#` 字符**，这个就像C/C++中的 `//` 一样。如果你要在你的Makefile中使用 `#` 字符，可以用反斜杠进行转义，如： `\#` 。

## 1.4.1 包含其他的makefile
在Makefile**使用 `include` 指令可以把别的Makefile包含进来**，这很像C语言的 `#include` ，被包含的文件**会原模原样的放在当前文件的包含位置**：
```
include <filename>
```

* `<filenames>` 可以是当前操作系统**Shell的文件模式**（可以包含路径和通配符）例如：文件权限rwx、脚本的解释器（shebang）如`#!/usr/local/bin/python`
* 、在 `include` 前面可以有一些空字符，**但是绝不能是 `Tab` 键开始**。 `include` 和 `<filenames>` 可以用一个或多个空格隔开
* **通配符的使用会让编写很简单**
  举例来说，若有这样几个Makefile： `a.mk` 、 `b.mk` 、 `c.mk` ，还有一个文件叫 `foo.make` ，以及一个变量 `$(bar)` ，其包含了 `bish` 和 `bash` ，那么，下面的语句：
  ```
  include foo.make *.mk $(bar)
   ```
  等价于：
```
include foo.make a.mk b.mk c.mk bish bash
```


make命令开始时，会找寻 `include` 所指出的其它Makefile，并**把其内容安置在当前的位置**。make会在当前目录下首先寻找，如果当前目录下没有找到，那么，make还会在下面的几个目录下找：

1. 如果make执行时，有 `-I` 或 `--include-dir` 参数，那么make就会在**这个参数所指定的目录**下去寻找。
    
2. 接下来按顺序寻找目录 `<prefix>/include` （一般是 `/usr/local/bin` ）、 `/usr/gnu/include` 、 `/usr/local/include` 、 `/usr/include` 。

环境变量 `.INCLUDE_DIRS` 包含当前 make 会寻找的目录列表。**你应当避免使用命令行参数 `-I` 来寻找以上这些默认目录**，否则会使得 `make` “忘掉”所有已经设定的包含目录，包括默认目录。
如果有文件没有找到的话，make会生成一条警告信息，但不会马上出现致命错误。**它会继续载入其它的文件**，一旦完成makefile的读取，make会再重试这些没有找到，或是不能读取的文件，如果还是不行，make才会出现一条致命信息。如果你想让make不理那些无法读取的文件，而继续执行，你可以在include前加一个**减号“-”**
```
-include <filenames>
```

## 1.4.2 慎定义环境变量MAKEFILES
如果你的**当前环境**中定义了环境变量 `MAKEFILES` ，那么**make会把这个变量中的值做一个类似于 `include` 的动作**。
**环境变量`MAKEFILES`中的值是其它的Makefile，用空格分隔**。只是，它和 `include` 不同的是，**从这个环境变量中引入的Makefile的“默认目标”(the default goal)不会起作用，如果环境变量中定义的文件发现错误，make也会不理**

**只要这个变量一被定义，那么当你使用make时，所有的Makefile都会受到它的影响，所以慎用**

## 1.4.3 make的工作方式
1. 读入所有的Makefile。
    
2. 读入被include的其它Makefile。
    
3. 初始化文件中的变量。
    
4. 推导隐式规则，并分析所有规则。
    
5. 为所有的目标文件创建依赖关系链。
    
6. 根据依赖关系，决定哪些目标要重新生成。
    
7. 执行生成命令

事实上，GNU make以外的make如cmake也是类似的行为
要注意，定义的变量被使用后make会把它展开在相应位置，但并不是立刻展开的：**如果变量出现在依赖关系的规则中，那么仅当这条依赖被决定要使用了，变量才会在其内部展开**

# Chapter 2. 书写规则
在Makefile中，**规则的顺序**是很重要的，因为，Makefile中**只应该有一个最终目标**，其它的目标都是被这个目标所连带出来的，所以一定要让make知道你的最终目标是什么
一般来说，定义在Makefile中的目标可能会有很多，但是**第一条规则中的目标将被确立为最终的目标**。如果第一条规则中的目标有很多个，那么，**第一个目标会成为最终的目标**。make所完成的也就是这个目标。

以下面的规则为例：
```
foo.o: foo.c defs.h       # foo模块
    cc -c -g foo.c
```
它告诉make两个信息：
1. 文件的依赖关系， `foo.o` 依赖于 `foo.c` 和 `defs.h` 的文件，如果 `foo.c` 和 `defs.h` 的文件日期要比 `foo.o` 文件日期要新，或是 `foo.o` 不存在，那么依赖关系发生。
    
2. 生成或更新 `foo.o` 文件，就是那个cc命令。它说明了如何生成 `foo.o` 这个文件。（当然，**foo.c文件include了defs.h文件**）

### 2.1 规则语法的说明
规则语法：
```
targets : prerequisites
    command
    ...
```
command也可以与targets:prerequisites在同一行，此时**需要与前面以分号作为间隔**
```
targets : prerequisites ; command
    command
    ...
```
命令太长则可以使用`\`反斜杠作为换行符，但make对一行的字符数没有限制
一般来说，make会以UNIX的标准Shell，也就是 `/bin/sh` 来执行命令。

### 2.2 使用通配符
和类UNIX的shell类似，如果我们想要定义一系列形式类似的文件，可以使用通配符进行标识，make支持 `*` ， `?` 和 `~`三个通配符。
波浪号（ `~` ）字符在文件名中也有比较特殊的用途。如果是 `~/test` ，这就表示**当前用户的 `$HOME` 目录下的test目录**。而 `~hchen/test` 则**表示用户hchen的宿主目录下的test 目录**。（这些都是Unix下的小知识了，make也支持）
windows下用户没有宿主目录，那么**波浪号所指的目录则根据环境变量“HOME”而定**
和shell类似，通配符`*`代表任意长度的字符串
```
clean:
	rm -f *.o
```
但是当通配符被用在变量中，这样并不代表它会被展开：
```
objects = *.o
```
这里objects的值就是`*.o`
如果希望通配符在变量中展开，正确的写法应该是：
```
objects := $(wildcard *.o)
```

通配符的一些使用例子：
```
objects := $(wildcard *.c)
```
列出当前目录下的所有`.c`文件
```
$(patsubst %.c,%.o,$(wildcard *.c))
```
自动编译出与所有`.c`文件相对应的`.o`文件
所以编译并链接所有`.c`和`.o`文件的简便写法：
```
objects := $(patsubst %.c,%.o,$(wildcard *.c))
foo : $(objects)
	cc -o foo $(objects)
```
关于makefile的关键字会在后面讨论

### 2.3 文件搜寻
当工程中含有大量源文件时，我们通常将源文件进行分类，存放在不同的目录下。 当make需要寻找文件的依赖关系时，我们可以**把一个路径告诉make，让make自动寻找**
makefile中的特殊变量`VPATH`用于**告诉make在当前目录找不到的情况下，在它所指定的目录下寻找**
```
VPATH=src:../headers
```
**目录由冒号`:`分隔，当前目录是搜索优先级最高的**，例如上面的代码就指定make在目录`src`和`../headers`寻找

另一个设置文件搜索路径的方法是**使用关键字`vpath`，它可以指定不同的文件在不同的目录下搜索，非常灵活**。使用方法有三种：
为符合模式`<pattern>`的文件指定搜索目录`<directories>`：
```
vpath <pattern> <directories>
```

清除符合模式`<pattern>`的文件的搜索目录：
```
vpath <pattern>
```

清除所有被设置好了的文件搜索目录：
```
vpath
```

对于模式`<pattern>`，往往要**使用通配符`%`，代表匹配零或若干个字符：**
```
vpath %.h ../headers
```
**对同一模式连续使用`vpath`语句可以指定各种不同的搜索策略的优先级：**
```
vpath %.c foo
vpath %.c blish
vpath %.c bar
```
先后在`foo`，`blish`和`bar`目录下寻找`.c`结尾的文件
效果和使用冒号`:`分隔不同目录是一样的：
```
vpath %.c foo:bar:blish
```

## 2.4 伪目标
和前面`clean`目标的例子一样，我们可以**使用一个不是文件，而是标签的目标，使得其只能被显式调用以完成我们想要的操作**，我们并不生成其文件，因此make也无法自己决定它是否应该被执行，只能显式调用
**伪目标不能和文件名重名**，不然就没有意义了，因此为了避免文件重名，**可以使用标记`.PHONY`来显式地指明该目标是伪目标，无论是否有这个文件，它都是伪目标：**
```
.PHONY : clean
clean :
	rm *.o temp
```

**可以为伪目标指定所依赖的文件；也可以使其作为默认目标：只需要把它放在makefile中的第一个目标位置即可：**
```
all : prog1 prog2 prog3
.PHONY : all

prog1 : prog1.o utils.o
    cc -o prog1 prog1.o utils.o

prog2 : prog2.o
    cc -o prog2 prog2.o

prog3 : prog3.o sort.o utils.o
    cc -o prog3 prog3.o sort.o utils.o
```
由于伪目标不会被生成文件，自然不会有`all`文件生成，因此**伪目标所依赖的文件就会被决议，从而使得我们一口气可以生成多个文件**
**伪目标同样也可以成为依赖：此时执行目标会一并执行所有的伪目标，有点类似于子程序：**
```
.PHONY : cleanall cleanobj cleandiff

cleanall : cleanobj cleandiff
    rm program

cleanobj :
    rm *.o

cleandiff :
    rm *.diff
```

## 2.5 多目标
makefile规则中的目标可以不止一个，也有可能多个目标同时依赖于同样的文件，并且执行指令类似，此时我们**可以把它们进行合并**
执行命令不是同一个时，则可以使用**自动化变量`$@`**，具体的在后面进行介绍，现在只需知道**它代表目前规则中所有目标的集合**，例如：
```
bigoutput littleoutput : text.g
	generate text.g -$(subst output,,$@) > $@
```
等价于：
```
bigoutput : text.g
	generate text.g -big > bigoutput
littleoutput : text.g
	generate text.g -little > littleoutput
```
其中，`-$(subst output,,$@)`中的`$`表示**执行一个Makefile的函数，函数名为`subst`，其参数为后面括号内的部分，其中`$@`的位置传递的是一个目标的集合，类似数组**，该函数**从集合中依此取出目标，执行命令**

## 2.6 静态模式
静态模式可以更容易地定义多目标的规则。语法：
```
<targets ...> : <target-pattern> : <prereq-patterns ...>
	<commands>
	...
```
`<targets>`定义一系列**目标文件，可以使用通配符，是目标的一个集合**
`<target-pattern>`**指明目标集合的模式**
`<prereq-patterns>`指明**目标的依赖模式，它对`target-pattern`指定的模式形成一次依赖目标的定义**
举个例子，若`<target-pattern>`定义为`%.o`，即**从目标的集合中选出所有可重定位目标文件组成的子集**；且若`<prereq-pattern>`定义为`%.c`，其意思是将`<target-pattern>`中定义的模式**只取`%`的部分**，加上`.c`形成新的集合
所以**两个模式都需要含有通配符`%`**
例如：
```
objects = foo.o bar.o

all : $(objects)

$(objects) : %.o : %.c
	$(CC) -c %(CFLAGS) $< -o $@
```
该例指明我们从集合`$object`中获取目标，`%.o`和`%.c`和上面所述的含义相同，`$<`和`$@`均为自动化变量：`$<`代表**第一个依赖文件**；`$@`代表**目标的集合**
于是该例中的规则等价于：
```
foo.o : foo.c
    $(CC) -c $(CFLAGS) foo.c -o foo.o
bar.o : bar.c
    $(CC) -c $(CFLAGS) bar.c -o bar.o
```
这样可以很轻松地对大量的目标使用各不相同的命令，效率很高。是项目中经常会用到的灵活模式

## 2.7 自动生成依赖性
makefile中的依赖关系可能会包含一系列的头文件，因为一个文件可能会`#include`很多头文件。但是这样就需要我们在加入和删除头文件时小心地对makefile进行更改，难以维护
因此，**使用编译器的`-M`选项，可以自动找寻源文件中包含的头文件并生成依赖关系：**
```
cc -M main.c
```
输出是：
```
main.o : main.c defs.h
```
但要注意，**在GNU的编译器中，`-M`会将一些标准库的头文件也包含进来**，想要像上面那样只使用提到的头文件，需要用选项`-MM`

GNU建议**将编译器为每个源文件自动生成的依赖关系放到一个文件中，为每个`name.c`文件生成一个makefile文件`name.d`，存放`.c`文件的依赖关系**
生成文件`.d`的模式规则如下：
```
%.d: %.c
    @set -e; rm -f $@; \
    $(CC) -M $(CPPFLAGS) $< > $@.$$$$; \
    sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; \
    rm -f $@.$$$$
```
从上往下每一行的意思是：
- 所有的`.d`文件依赖于所有的`.c`文件
- `rm -f $@`删除所有的目标，即`.d`文件
- 为每个依赖文件 `$<` ，也就是 `.c` 文件生成依赖文件， `$@` 表示模式 `%.d` 文件，如果有一个C文件是name.c，那么 `%` 就是 `name` ， `$$$$` 意为一个随机编号
- 使用sed命令做了一个替换，关于sed命令的用法请参看相关的使用文档
- 删除临时文件

总之就是自动更新和生成`.d`文件，生成之后在当前makefile中可使用`include`命令引入其他makefile，然后便可以