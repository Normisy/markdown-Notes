# Chapter 1. Qt的编译过程
Qt提供的库允许我们编写界面
首先我们不使用QtCreator，而是编写一个qmake.cpp文件，使用Qt提供的各种库来完成一个可输入的窗口，代码如下
```
#include <QApplication>
#include <QLabel>  //标签：界面框上的文字或图片
#include <QLineEdit>  //行编辑框控件
#include <QPushButton>  //按钮控件 
#include <QHBoxLayout>  //水平布局
#include <QVBoxLayout>  //垂直布局
#include <QWidget>  //窗口


//Qt代码的编写一般来说都是这个模式
int main(int argc, char *argv[])  //命令行参数：命令行调用可执行文件时，argc可以知道有多少命令行参数被传递给程序；argv是字符串的数组。argv[0]为程序的名字i，argv[1],argv[2]...为传递给程序的其他参数
{
	QApplication app(argc, argv);
	QLabel *infoLabel = new QLabel;  //Qt的规范命名：描述+控件名，这里就是”表达信息的标签“的含义
	QLabel *openLabel = new QLabel;
	QLineEdit *cmdLineEdit = new QLineEdit;
	QPushBUtton* commitButton = new QPushButton;
	QPushBUtton* cancelButton = new QPushButton;
	QPushBUtton* browseButton = new QPushButton;
	
	//在控件上面写字
	infoLabel->setText("input cmd:");
	openLabel->setText("open");
	commitButton->setText("commit");
	cancelButton->setText("cancel");
	browseButton->setText("browse");

	//为控件布局，使用布局对象，它们都是对象
	QHBoxLayout* cmdLayout = new QHBoxLayout;  //水平布局对象
	cmdLayout->addWidget(openLabel);  //为布局对象添加控件
	cmdLayout->addWidget(cmdLineEdit);

	QHBoxLayout* buttonLayout = new QHBoxLayout;
	buttonLayout->addWidget(commitButton);
	buttonLayout->addWidget(cancelButton);
	buttonLayout->addWidget(browseButton);

	QVBoxLayout* mainLayout = new OVBoxLayout;  //把水平布局对象垂直布局
	mainLayout->addWidget(infoLabel);
	mainLayout->addWidget(cmdLayout);
	mainLayout->addWidget(buttonLayout);

	QWidget w;
	w.setLayout(mainLayout);
	w.show();

	return app.exec();  //app.exec()代表死循环执行，这是为了使得窗口一直保持停留
}
```
Qt里把窗口上的小组件称为**控件**

那么写好了这样一个文件后，Qt为我们提供了一个命令行环境，用于对Qt文件进行编译：
![[Pasted image 20240910172244.png]]
用windows的命令行比较麻烦，不建议


进入命令行后，`cd`到文件所在的目录下面，可以用`qir`显示目录下的文件：
![[Pasted image 20240910172543.png]]
然后就可以正式开始编译了

## 生成工程文件：
使用`qmake -project`生成工程文件：
![[Pasted image 20240910172817.png]]
opps，出现这种状况说明忘记**修改`qmake`的环境变量了**，我们先找到qt的**安装目录**，找到`bin`目录，进入复制路径
然后在 此电脑-> 右键->属性->高级系统设置->环境变量 中把bin目录加进去
但是再回去看，会发现它好好地生成了`.pro`文件...：
![[Pasted image 20240910175258.png]]
在visual studio打开该文件：
![[Pasted image 20240910175758.png]]
要在下面加上这么一句话，不然编译会有问题：
![[Pasted image 20240910175921.png]]
保存后就可以进行下一步了

## 生成makefile
在当前目录下使用`qmake`命令，就可以自动生成makefile，然后用`make`或`mingw32-make`执行该makefile就可生成可执行文件`qmake.exe`，然后就可以直接执行代码了。

# Chapter 2. 使用Qt creator
## 1. 设置控件
![[Pasted image 20240910182756.png]]
创建项目后就是这样的界面，这里我们选择的`widget`模式，即小窗口，后面还有更多的其他类型可供我们选择
`*.pro`是项目的文件,Headers则是存放头文件的地方。
进入`Forms`，可以看见这样的文件：
![[Pasted image 20240910190231.png]]
这就是设计文件了，双击进入设计师界面：
![[Pasted image 20240910190308.png]]
左边就是我们常用的**控件**了
- spacers：弹簧，用于排版，可以向水平或垂直占用一个空间且不显示出来
- buttons：push button和tool button差不多，就是点击的按钮；radio button就是我们问卷调查时点击后会改变黑点颜色的那个按钮，check box也差不多，打勾；
- item views：数据和显示分离，多用于数据库
- item widgets：单元控件，例如我们电脑的文件系统的形式
- containers：容器，存储别的控件
- input widgets：输入控件，通过这个进行输入
- display widgets：显示控件，显示一些信息

使用的组件会显示在右上角的框里，直接拖动相应组件进行拼凑即可：
![[Pasted image 20240910191117.png]]
也可以在这里**右键组件对象进行重命名等操作**
右下角的框则显示了各组件的**继承关系以及信息**
![[Pasted image 20240910192731.png]]
其中**可以在font处进行字体的修改**

实际上这些傻瓜式操作都是由`.ui`文件背后的代码操作完成的：
![[Pasted image 20240910193037.png]]

点击右下角的这里运行或debug：
![[Pasted image 20240910193717.png]]
就可以看见我们的widget了：
![[Pasted image 20240910193836.png]]
## 2. 信号与槽
完成了这样的界面后，我们当然需要为其赋予意义与逻辑，例如点击按钮后会得到某种响应
这就涉及到qt的一个重要机制：***信号与槽**了
- **当我们点击按钮时，就相当于发出了一个==信号==；发出信号后，程序根据该信号执行一段代码（函数），这就称为==槽（槽函数）==**

信号与槽的实现方法也很简单，有四种办法实现：
### a. 右击按钮
右键按钮：
![[Pasted image 20240910195145.png]]
点击”**转到槽**“，会显示选择信号：
![[Pasted image 20240910195220.png]]
正常情况下都是**单击clicked()**，其他模式以后补充
这样就会跳转到widget.cpp文件：
![[Pasted image 20240910195414.png]]
并且多出一个函数，同时在头文件增加了红线部分的代码：
![[Pasted image 20240910195519.png]]

当我们点击该按钮的时候，就会执行函数`on_okButton_clicked()`的代码了

我们可以通过对`ui`的成员进行`->`来访问所需信息：
![[Pasted image 20240910203043.png]]
**在Qt中，提供了QString来表示字符串，它和STL的String是一样的**

一个常用的类是QProcess，它提供了根据字符串调用进程的方法：
![[Pasted image 20240910202828.png]]
这里调用进程program，比如notepad记事本等等

### b. 在类的构造函数中使用宏来写
对于lineEdit，我们希望当我们在敲击回车时能够达到和点击按钮一样的效果，可以**在构造函数中使用`connect()`函数，将信号与槽进行连接**：
![[Pasted image 20240910204225.png]]
这里接受lineEditStr发出的**回车信号**，然后让this即类自己调用on_okButton_clicked进行处理
这里使用了**SIGNAL和SLOT宏**，**它们用于定义信号（SIGNAL）与槽（SLOT）的连接**

**不同的控件可以发出很多不同的信号，当我们想寻找某控件的信号时，==善用帮助栏==：Qt拥有相当新手友好的帮助列表：**
![[Pasted image 20240910205147.png]]
![[Pasted image 20240910205237.png]]

### c. 在构造函数中使用指针的形式来写
我们依然可以

### d. 槽函数表达式

# Chapter 3. Qt使用错误总结：总结不一定是对的，但是一定很有乐子
## 1. Qt打包1：不要乱设置环境变量！
前情提要：在某次使用qmake时，我收到了以下的报错：
```
Project ERROR: Cannot run target compiler 'cl'. Output:
```
它提示我没有找到cl编译器，也就是说**我没有把含有cl编译器的路径包含到环境变量PATH**，于是经过几次添加，我发现在qt的bin文件夹下似乎并没有cl.exe：
![[Pasted image 20240912165248.png]]
于是我选择**使用visual studio的bin目录下的cl编译器，把bin的路径加到环境变量下**：
![[Pasted image 20240912165450.png]]

很好，不报错了。但是**我忘记了把之前加的环境变量给删掉了**，这就导致了后面一系列的逆天问题

## 2. Qt打包2：你这make不对吧
对于项目打包，首先我想使用`qmake`生成makefile，然后让make生成可执行程序，于是这一步又报错了：
```
34382@Normist MINGW64 /d/codecode/testpro (master)  
$ qmake  
Project ERROR: You need to set the ANDROID_NDK_ROOT environment variable to point to your Android NDK.  
Could not read qmake configuration file D:/Qt/5.14.2/android/mkspecs/android-clang/qmake.conf.  
Error processing project file: D:\codecode\testpro\project0.pro
```
它提示我**没有将环境变量ANDROID_NDK_ROOT设置到指向我自己的Android NDK**，我不知道这个android ndk为什么没有在qt creator安装时配置好，但是好吧，那就自己去下载
**AI建议我下载Android studio，在它的Andriod SDK manager下进行NDK的下载**，我寻思着正好可以把`cmake`也一起安装了，而且以后说不定要用这个IDE，就下载了
此时我的电脑处于校园网环境，这给我带来了第一个问题
在下载完android studio后，它就提示我**SDK丢失，需要下载**，然后进入SDK manager后又告诉我`Android SDK(unavaliable)`，这个真让人摸不着头脑
于是经过搜索，我知道了，**它的下载中用到了谷歌的某个远程网址，在校园网环境下似乎是被ban的状态，同时在该IDE下新建项目也需要打开某同样被ban的远程网址，所以必须换个局域网**
更换完成后，终于可以下载了，这时我使用了它提供的默认路径
该路径需要经过一个`AppData`文件夹，**它是隐藏的，没办法被qmake.exe访问到**，于是我又把路径更改一遍，重新下到了D盘的一个可见文件夹

然后就是设置环境变量，在**电脑的高级选项设置的环境变量中设置ANDROID_NDK_ROOT为环境变量并没有解决这个报错**，于是这时我决定**直接使用shell指令设置就好了：**
```
export ANDROID_NDK_ROOT="D:\SDK\ndk\27.1.12297006"
```
然后果然不报错了，我们运行`qmake`和`make`试试看：
![[Pasted image 20240912171959.png]]
嗯，makefile有了，中间目标文件和一系列的文件也有了，但是我**可执行文件**呢
后来我才意识到，**这是因为make并没有找到代码里用到的依赖文件`*.dll`（文件扩展），所以没办法这么生成**

## 3. Qt打包3：找到正解的第一步
总之经过一番搜索，我找到了正确的打包方法：
- **1. 点进qt creator，打开项目文件夹，点进左边的“项目”一栏，可以找到构建设置：**
  ![[Pasted image 20240912172520.png]]
  - **2. 根据需要更改构建配置为`release`（发布）、`debug`和`profile`（简要版本），这里选择release。然后选择构建目录到自己想要的位置**
  - **3. 点击上排菜单栏“构建”位置，选择把这个项目进行构建：**
  ![[Pasted image 20240912173106.png]]
  - **4. 然后就可以在相应路径下找到构建好的文件夹了：**
![[Pasted image 20240912173223.png]]

## 4. Qt打包4：使用windeployqt，但是前面的伏笔吻了上来
**在上述文件夹下的`release`子文件夹下，我们可以找到`项目名.exe`的这个可执行程序，对它使用：
```
windeployqt 项目名.exe
```
就可以自动把所需的平台插件以及依赖文件加到`release`子文件夹了**

问题也就出现在这里了，还记得前面**忘记删除不需要的环境变量**的伏笔吗，它被回收了。**windeployqt会自动根据环境变量Path中的==Qt目录下的靠前的那条路径==来寻找所有的依赖文件以及平台插件，将其整合到`release`文件夹中**
因此我忘记删的这些路径中的第一条android路径就成了默认目标：
![[Pasted image 20240912165248.png]]
但是这里面**没有大部分依赖文件和平台插件**，因此会有这样的报错：
```
34382@Normist MINGW64 /d/DS2024/build-project0-Desktop_Qt_5_14_2_MinGW_32_bit-Release/release
$ windeployqt project0.exe
Unable to find dependent libraries of D:\Qt\5.14.2\android\bin\Qt5Gui.dll :Cannot open 'D:/Qt/5.14.2/android/bin/Qt5Gui.dll':
34382@Normist MINGW64 /d/DS2024/build-project0-Desktop_Qt_5_14_2_MinGW_32_bit-Release/release
$ windeployqt project0.exe
Unable to find dependent libraries of D:\Qt\5.14.2\android\bin\Qt5Widgets.dll :Cannot open 'D:/Qt/5.14.2/android/bin/Qt5Widgets.dll':

```
也就是找不到各种各样的依赖文件。就算我们把所有相应的文件加到目录下，也会有这样的问题：
```
34382@Normist MINGW64 /d/DS2024/build-project0-Desktop_Qt_5_14_2_MinGW_32_bit-Release/release
$ windeployqt project0.exe
D:\DS2024\build-project0-Desktop_Qt_5_14_2_MinGW_32_bit-Release\release\project0.exe 32 bit, release executable
Direct dependencies: Qt5Core Qt5Gui Qt5Widgets
All dependencies   : Qt5Core Qt5Gui Qt5Widgets
To be deployed     : Qt5Core Qt5Gui Qt5Widgets
Unable to find the platform plugin.

```
依赖文件找到了，但找不到平台插件
这时我是把要的工具都加到了路径下，然后成功生成了：
![[Pasted image 20240912174350.png]]
在这之后才后知后觉到**应该把环境变量改掉才对！改成工具齐全的：
```
D:\Qt\Tools\QtCreator\bin
```
不就没这么多事了吗！！**

好吧刚写完没多久就被打脸了：**请把所有的bin文件路径添加到环境变量，==并且视情况来把某个路径放置到最前面==**
这是因为在打包后碰到了一个这样的问题：
![[WQ1XST@14N[O9(_XCA[]4LI.png]]
而且这个问题出现频率还不低，网上的大部分教程都告诉我们**修改环境变量，没加的bin都加上，都加上了就改改次序，把这些bin轮流放到它们的集合的一号位上试试**
我个人最后成功打开的环境变量顺序如下：
![[Pasted image 20240912195457.png]]


## 5. Qt打包5：良好的编程习惯


# Chapter 4. 一次小作业的经验
## 4.1 控制组件的大小不会被轻易改动
只需要设置最小尺寸等于最大尺寸就行了
![[Pasted image 20241015195236.png]]


## 4.2 设计控件样式
可以查找`QstyleSheet`的代码用法，也可直接在`ui`界面右下角`styleSheet`这一行图形化地设置
![[Pasted image 20241015210814.png]]