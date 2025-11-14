# Lecture 2. 多处理器编程

第一讲的take-away message：
- 程序就是状态机+系统调用，编译器将源代码翻译为对应的机器代码（二进制代码）
- 应用视角下，操作系统实际上就是一条`syscall`指令
- 除了系统调用指令，其他的指令都只是纯粹的计算，不能完成退出、屏幕显示等功能

多处理器编程的概念仅仅在状态机视角上迈出了一小步，但是它带来的复杂性却不容小觑

# 2.1 并发

并发：在同一时间存在、发生或完成（一段时间内有多个任务）
操作系统是最早的并发程序之一

对于单线程程序，如果程序中没有`random`这样的非确定性指令的话，其执行状态其实就是一条线，系统中的内容都是完全确定的；而并发程序中，多个事件的发生顺序是完全随机的，它会带来很多的不确定性，从而增加了编程的复杂度

### 2.1.1 线程和状态机视角

线程就是**共享内存的多个执行流**：
- 执行流拥有**独立的堆栈和寄存器**
- 它们共享全部的内存

我们可以发现**全局变量、全局内存（堆区）是大家一起共享的**，而栈内的局部变量和各自的程序计数器pc却**只在一个栈帧内适用**
因此，将单线程的状态机模型推广到多线程情况，只需要将状态中的“栈”推广为“多个独立的栈帧链”，**一个栈帧链就是一个线程**，栈帧链刻画的是一个线程内的栈状态变化。
在一个状态外部，分支的产生来自于：在每一步，外部的选择器选择是执行多个线程中的哪一个。假设有T1和T2两个线程，选择T1执行，就意味着**假装T2所在的线程栈帧链不存在**，但是**依然可以看见全局的变量和堆内存**，执行T1产生某个状态变化
这样的分支不断存在，意味着**并发程序的每一步都是不确定的**，我们不能假设某个时间某个线程一定会被执行

### 2.1.2 线程库

为了简化编程，课程封装了一个简短的线程库`thread.h`：
```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <stdatomic.h>
#include <assert.h>
#include <unistd.h>
#include <pthread.h>

#define NTHREAD 64
enum { T_FREE = 0, T_LIVE, T_DEAD, };
struct thread {
  int id, status;
  pthread_t thread;
  void (*entry)(int);
};

struct thread tpool[NTHREAD], *tptr = tpool;

void *wrapper(void *arg) {
  struct thread *thread = (struct thread *)arg;
  thread->entry(thread->id);
  return NULL;
}

void create(void *fn) {   //创建一个入口函数是fn的线程，fn是函数指针
  assert(tptr - tpool < NTHREAD);
  *tptr = (struct thread) {
    .id = tptr - tpool + 1,
    .status = T_LIVE,
    .entry = fn,
  };
  pthread_create(&(tptr->thread), NULL, wrapper, tptr);
  ++tptr;
}

void join() {  //
  for (int i = 0; i < NTHREAD; i++) {
    struct thread *t = &tpool[i];
    if (t->status == T_LIVE) {
      pthread_join(t->thread, NULL);
      t->status = T_DEAD;
    }
  }
}

__attribute__((destructor)) void cleanup() {
  join();
}
```

因为这个库用到了线程库`pthread`，在编译时需要新增`-lpthread`选项
```
gcc hello.c -lpthread && ./a.out
```

形式语义：
- `create`函数：就是在原来的状态机模型中创建一个新的执行流（栈帧链），它的pc初始化为0，然后新创建的执行流的执行函数就是函数指针`fn`
- `join`函数：假设系统中现在有三个线程T1、T2和T3，如果T1执行了`join()`，此时线程T1会进入一个死循环：`while (T2和T3没有结束);`，即任何一个其他线程没有结束时，若选中该线程执行，它会不断地回到同一个状态，和没执行T1前是一样的状态

例如下面这个`hello.c`的程序：
```c
#include "thread.h"

void Ta() { while (1) { printf("a"); } }

void Tb() { while (1) { printf("b"); } }

int main() {
        create(Ta);
        create(Tb);
}

```

将它编译并且执行后，我们可以在命令行界面看到很多交替的a和b的输出，这代表CPU调度执行两个线程

此时如果使用`top`指令观察CPU使用情况：
	![[Pasted image 20250309145720.png]]
	可以发现，`a.out`这个进程的CPU占用率超过了100%，这是因为每个线程被放到了不同的CPU处理器上被执行，相对于1个CPU，它的占用率要更高

#### 一些困惑

在这个简单的模型下，一些问题自然而然地会产生：
- 所有的线程都在同一个地址空间里吗？
- 如果在全局变量中设置一个指针，它指向一个线程中的某个局部变量，那么其他的线程是否能通过这个全局指针，访问别的线程的局部变量值？
- 在`create`线程的时候，是否确实共享了内存？

很多困惑通过编写测试代码是完全可以尝试解答的。例如下面这个代码：
```c
#include "thread.h"

int x = 0;

void tHello(int id) {
        usleep(id * 100000);
        printf("Hello from t #%c\n", "123456789ABCDEF"[x++]);
}

int main() {
        for (int i = 0; i < 10; i++) {
                create(tHello);
        }
}

```
设置了一个全局变量x，并利用它来在`"123456789ABCDEF"`这个字符串中索引
在输出中，可以看到如下情况：
```
normist@MIST:~/Learn/NJU-OS$ gcc memoryT.c -lpthread && ./a.out
Hello from t #1
Hello from t #2
Hello from t #3
Hello from t #4
Hello from t #5
Hello from t #6
Hello from t #7
Hello from t #8
Hello from t #9
Hello from t #A

```
可以发现各个线程显然是共享地使用了这个全局变量x



```c
#include "thread.h"

//下面的方式将变量声明为线程局部的变量，为每个线程创建一个副本
__thread char *base, *cur; // thread-local variables
__thread int id;

// objdump to see how thread-local variables are implemented
__attribute__((noinline)) void set_cur(void *ptr) { cur = ptr; }
//设置cur为ptr 

__attribute__((noinline)) char *get_cur()         { return cur; }
//获取cur的值

void stackoverflow(int n) {
  set_cur(&n);  //将参数的地址赋给cur
  if (n % 1024 == 0) {
    int sz = base - get_cur();  //base存储的是最外层参数tid的地址，而cur则是当前递归层数的局部变量n的地址，两个地址之间的间隔可以用于估计整个栈的大小
    printf("Stack size of T%d >= %d KB\n", id, sz / 1024);
    //
  }
  stackoverflow(n + 1);  //无限递归地创建栈帧，直到溢出
}

void Tprobe(int tid) {
  id = tid;
  base = (void *)&tid;  //将参数tid的地址赋值给base
  stackoverflow(0);
}

int main() {
  setbuf(stdout, NULL);
  for (int i = 0; i < 4; i++) {
    create(Tprobe);
  }
}

```
这个程序通过两个局部变量的地址相减，获取每个线程的栈的大小变化，当发生栈溢出时，我们就可以知道操作系统分配给每个线程的栈空间有多大了
将输出使用管道符连接到`sort`，即可对结果排序：
```
gcc stack-probe.c -lpthread && ./a.out | sort -nk 6
```
选项`-nk`表示以某一列作为键值进行排序，最后一个参数`6`代表第六列

最后得到最大的一行输出为：
```
Stack size of T3 >= 8128 KB

```
因为溢出时地栈大小相比最后一次输出增加了64KB，因此真实的栈空间上限为$8128+64=8192=2^{13}KB$


最后一个问题，创建线程所使用到的系统调用是什么？
这个问题可以通过使用`strace`工具追踪程序运行时发生的各个系统调用来找到答案：
```
strace a.out
```
在strace的输出中，可以看到下面这个系统调用：
```
clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, child_tid=0x7aa327a00990, parent_tid=0x7aa327a00990, exit_signal=0, stack=0x7aa327200000, stack_size=0x7fff80, tls=0x7aa327a006c0} => {parent_tid=[30094]}, 88) = 30094

```
这是`clone`系统调用的一个变体。`clone`系统调用用于创建一个新线程或进程，类似的系统调用还有我们知道的`fork()`，不过`clone()`更加灵活：**它允许调用者指定哪些资源需要被共享，哪些需要被复制**，允许更精细的资源共享控制，使得新的进程可以在很多方面与父进程共享或隔离资源。

线程库的更多介绍可以使用`man pthreads`命令查看介绍手册得到

#### 可怕的事情

在多处理器系统中，**线程的代码可能会同时执行**，例如两个线程同时往一个数据结构（如红黑树）中`insert`，或是执行某个操作时，红黑树的性质真的能够被保持吗？
这样的问题是多处理器编程最恐怖之处，例如下面这个山寨支付宝扣款程序：
```c
unsigned int balance = 100;
int Alipay_withdraw(int amt) {
  if (balance >= amt) {  //若钱数大于支付数，扣除
    balance -= amt;
    return SUCCESS;
  } else {
    return FAIL;
  }
}
```
如果两个线程T1，T2同时并发地支付100元，会怎么样？
此时有这么一种可能，T1线程进行了if判断，得到存款数足以支持扣款，但是还没进入扣款；然后**同时**（注意是多处理器多线程）T2线程被执行，因为还未发生扣款，T2依然被判定为正确，进入if分支

当钱款正好是100时，两个线程都会进行扣款，这下因为补码的性质，产生$0-100=2^{64}-100=18446744073709551516$的余额

再例如简单的求和程序：
```c
#define N 100000000
long sum = 0;

void Tsum() { for (int i = 0; i < N; i++) sum++; }

int main() {
  create(Tsum);
  create(Tsum);
  join();
  printf("sum = %ld\n", sum);
}
```
如果运行这个程序，会发现输出的和在每一次运行时都不一样，并且都不是2N的值
如果换成内联汇编实现求和，也是一样的
```c
void Tsum() { 
	for (int i = 0; i < N; i++) 
		asm volatile("add $1, %0": "+m"(sum)); 
}
```
除非使用`lock add`，才能得到正确的结果，但是运行速度却显著下降：
```c
void Tsum() { 
	for (int i = 0; i < N; i++) 
		asm volatile("lock add $1, %0": "+m"(sum)); 
}
```
锁相关的介绍会在后面的讲座进行

## 2.2 并发编程所失去的特性

### 2.2.1 原子性的丧失

**原子性：一段代码执行独占整个计算机系统**
单处理系统中“程序独占处理器执行”的基本假设在多处理器系统上并不成立
- 单处理器多线程：线程在运行时可能被中断，然后切换到别的线程
- 多处理器多线程：线程根本就是并行执行的

历史上，大家都在想办法在共享内存上实现原子性，但是在Dekker's Algorithm出现前的几乎所有实现都是错的，且只在两个线程中起作用

但是在我们之前的输出a和b的测试程序中，a和b交替出现，并且printf具有缓冲区，如果两个线程同时执行缓冲区的索引，那肯定会产生错误，但是它却没有产生因为同时输出而导致的莫名其妙的字符，例如空格
这是因为标准库的`printf`早就考虑到了线程的安全性，它保证了多线程的原子性

### 2.2.2 原子性的实现

互斥和原子性是本课程的重要主题，我们需要实现两个api：`lock(&lk)`和`unlock(&lk)`
在这两个api之间的区域，我们能够保证**实现临界区之间的绝对串行化**，与此同时程序的其他部分依然可以并行执行

***99%的并发问题都可以使用一个队列来解决***：
- 将大任务切分为可以并行的小任务，这就产生了很多的`worker thread`
- `worker thread`到锁保护的队列里取得任务
- 除去不可并行的部分，剩下的部分都可以获得线性级别的加速$T_N<T_{\intefy}+\frac{T_1}{n}$

### 2.2.3 顺序性的丧失

并行程序执行的顺序我们也没办法知道，在之前的求和程序中，如果我们在编译时采用一些优化：
```
gcc -O1 -lpthread sum.c && ./a.out

```
（注意，将`sum++`替换为内联汇编`asm`的版本优化后并没有任何区别，原因和后面讲解的一样）
此时会发现，在O1级别的优化下，sum程序的输出保持不变了，且一直为N=100000；在O2级别的优化下则一直为2N

在编译器的眼中，**系统调用是不可优化的石头，但内存是可以被优化的**
编译器**按照顺序程序（单线程）的方式来优化代码**，用状态机的角度，当执行语句在一个顺序的状态之间移动（例如x=1->x=2->x=3）时，此时编译器会直接删除前面的步骤，直接写入x=3

不妨使用`objdump`检查一下编译器是怎么进行优化的：
```
gcc -c -O1 sum.c && objdump -d sum.o
```
在O1优化下，生成的`Tsum`函数的汇编代码如下：
```
000000000000001a <Tsum>:
  1a:	f3 0f 1e fa          	endbr64
  1e:	48 8b 15 00 00 00 00 	mov    0x0(%rip),%rdx        # 25 <Tsum+0xb>
  25:	48 8d 42 01          	lea    0x1(%rdx),%rax
  29:	48 81 c2 01 e1 f5 05 	add    $0x5f5e101,%rdx
  30:	48 89 c1             	mov    %rax,%rcx
  33:	48 83 c0 01          	add    $0x1,%rax
  37:	48 39 d0             	cmp    %rdx,%rax
  3a:	75 f4                	jne    30 <Tsum+0x16>
  3c:	48 89 0d 00 00 00 00 	mov    %rcx,0x0(%rip)        # 43 <Tsum+0x29>
  43:	c3                   	ret
```

第二行：
```
  1e:	48 8b 15 00 00 00 00 	mov    0x0(%rip),%rdx  
```
将`%rip`中存储的地址所对应的值存入`%rdx`寄存器中，这里`%rip`其实就是sum全局变量的地址，将sum值存到`%rdx`中，然后之后对`%rdx`中的值进行计算，再写回给`%rip`指向的地址处

下面这两行：
```
 25:	48 8d 42 01          	lea    0x1(%rdx),%rax
 29:	48 81 c2 01 e1 f5 05 	add    $0x5f5e101,%rdx

```
第一行首先将`%rdx`中的值加上1得到的值，加载到寄存器`%rax`上（注意，`lea`指令的作用是计算数值，而不是从地址中取值）
然后，第二行直接把0x5f5e101加到寄存器`%rdx`上，这个立即数的值就是N，相当于是sum+N

随后下面这几行进行循环：
```
  30:	48 89 c1             	mov    %rax,%rcx
  33:	48 83 c0 01          	add    $0x1,%rax
  37:	48 39 d0             	cmp    %rdx,%rax
  3a:	75 f4                	jne    30 <Tsum+0x16>

```
地址30的指令将`%rax`当前的值存储到`%rcx`中
地址33的指令将`0x1`加到`%rax`存储的值中
地址37的指令将`%rdx`中的值与`%rax`中的值相比较，若不相等，跳回到地址30处；若相等，则进入最后一部分：
```
  3c:	48 89 0d 00 00 00 00 	mov    %rcx,0x0(%rip)        # 43 <Tsum+0x29>
  43:	c3                   	ret
```
将`%rcx`中的值写给sum，然后退出

可以发现，优化版本的代码相当于直接将N写给sum，对于多个线程，都是使用`mov`进行相同值的写入，因此无论如何，得到的结果都是N

对于O2优化，情况也是类似的：
```
0000000000000020 <Tsum>:
  20:	f3 0f 1e fa          	endbr64
  24:	48 81 05 00 00 00 00 	addq   $0x5f5e100,0x0(%rip)        # 2f <Tsum+0xf>
  2b:	00 e1 f5 05 
  2f:	c3                   	ret

```
它甚至更加极端：直接将2N=0x5f5e100写到sum的地址中了，因此无论如何都是2N

另一个例子：对于下面的代码：
```c
extern int done;
void join() {
	while (!done);
}
```
第一行代表`done`这个变量是在外部定义的，在我们内部进行访问。join函数每次检查这个外部定义的变量，当它为0时保持死循环
这个代码的O1优化汇编如下：
```
0000000000000000 <join>:
   0:	f3 0f 1e fa          	endbr64
   4:	8b 05 00 00 00 00    	mov    0x0(%rip),%eax        # a <join+0xa>
   a:	85 c0                	test   %eax,%eax
   c:	75 02                	jne    10 <join+0x10>
   e:	eb fe                	jmp    e <join+0xe>
  10:	c3                   	ret

```
直接从外部（`%rip`寄存器）取出done的值，然后进行判断和跳转，若检测不为零（test指令将`%eax`与自己作按位与操作，如果得到的结果为0,零标志位ZF设为1；反之设为0。jne指令检测ZF，若为1,执行跳转，反之不跳转），那么跳转到返回语句；反之，不跳转，进入`e:  jmp e`这个死循环

可以发现，**编译器假设程序是顺序的**，它将`while (!done);`优化为`if (!done) while(1);`
当然，如果我们希望编译器对某些地方不要进行优化，**需要向代码中插入“不能优化”的barrier**
之前在求和程序中说，使用内联汇编代替`sum++`的版本在任何优化下都没有产生不同的结果，这其实就是一个barrier
我们一般可以通过设置`asm volatile("" ::: "memory");`的方式插入这样的屏障：
```c
extern int done;
void join() {
	while (!done) {
		asm volatile("" ::: "memory");
	};
}
```
这条语句代表“执行时可能会有其他人还会更改内存的值，因此不要进行优化“的意思


或者，将`extern int done`声明为 `extern int volatile done`
这要求编译器每次使用done时从内存中读取数值，而不是寄存器中

总结：在编译器对代码进行的eventually consistent（**最终一致性：在分布式系统中的数据最终会达到一致状态，但在某些时刻数据可能并不一致**）优化处理中，它假设程序顺序执行，并且把它认为的顺序状态最终的结果直接得出并赋值。但是多线程编程中，我们知道这种顺序状态是不正确的，因此得到：
**共享内存无法被作为线程进行同步的工具**，这也是编译器假设程序是顺序的带来的麻烦

### 2.2.4 多个处理器之间的即时可见性的丧失

并发程序给我们带来的最大的麻烦，可以看到以下代码：
```c
#include "thread.h"

int x = 0, y = 0;

atomic_int flag;  //开关，只使用了它的两个位
#define FLAG atomic_load(&flag)   //原子读
#define FLAG_XOR(val) atomic_fetch_xor(&flag, val)  //原子异或常数val
#define WAIT_FOR(cond) while (!(cond)) ;  //等到某个条件成立

__attribute__((noinline))
void write_x_read_y() {
  int y_val;
  asm volatile(
    "movl $1, %0;" // x = 1
    "movl %2, %1;" // y_val = y
    : "=m"(x), "=r"(y_val) : "m"(y)
  );
  printf("%d ", y_val);
}

__attribute__((noinline))
void write_y_read_x() {
  int x_val;
  asm volatile(
    "movl $1, %0;" // y = 1
    "movl %2, %1;" // x_val = x
    : "=m"(y), "=r"(x_val) : "m"(x)
  );
  printf("%d ", x_val);
}

void T1(int id) {  //该线程让
  while (1) {
    WAIT_FOR((FLAG & 1));  //等待对应开关位变为1（01或11时退出死循环）
    write_x_read_y();  //写x读y
    FLAG_XOR(1);  //将对应开关关掉
  }
}

void T2() {
  while (1) {
    WAIT_FOR((FLAG & 2));//等待对应开关位变为1（10或11时退出死循环）
    write_y_read_x();  //写y读x
    FLAG_XOR(2);  //将对应开关关掉
  }
}

void Tsync() {
  while (1) {
    x = y = 0;
    __sync_synchronize(); // full barrier，确保x=y=0被写入内存
    usleep(1);            // + delay
    assert(FLAG == 0);  //确保flag=0,两个位均为0（关闭-关闭状态）
    FLAG_XOR(3);  //与11（二进制3）进行异或，将0-0变为1-1（开启-开启状态）
    // T1 and T2 clear 0/1-bit, respectively
    WAIT_FOR(FLAG == 0);  //一直等到两个开关都处于关闭状态
    printf("\n"); fflush(stdout);
  }
}

int main() {
  create(T1);
  create(T2);
  create(Tsync);
}

```
对于T1线程，它设置x=1，然后读取y；T2线程则是设置y=1，读取x
用状态机视角，假设初始x=0;y=0;，然后
可以知道，无论如何，x=0且y=0的情况“理论上”是不存在的，因为毕竟T1和T2总得执行一个

不妨使用一下数据处理的方法
```
./a.out | head -n 100000 | sort | uniq -c
```
`uniq`用于统计，`head`用于取出前面一定数量的条目

令人惊讶的是，我们统计出来的结果是这样的：
```
normist@MIST:~/Learn/NJU-OS$ ./a.out | head -n 100000 | sort | uniq -c
  14736 0 0 
  76676 0 1 
   8586 1 0 
      2 1 1 

```
1-1的情况几乎没有，而理应不存在的0-0情况却是最多的

这是因为**现代的处理器也是（动态的）编译器！**
看上去，我们的处理器正在执行x86的汇编代码，但是实际上，**cpu会将汇编代码使用电路翻译成更小的$\mu ops$**，相当于是又经过了一次编译
每一条$\mu ops$的执行，需要经过fetch -> issue -> execute -> commit这样四个阶段的过程
（csapp中流水线的部分）

虽然$\mu ops$可以被假设是顺序执行的，但是现代处理器完全可以将两个$\mu ops$同时issue，实现并行的执行。如果$\mu ops$之间**具有数据依赖性，则需要维护它们执行的顺序**
这样，就形成了一个有向无环图
在任何时刻，处理器都会维护一个$\mu ops$的池子（流水线池），在不违反数据依赖性的情况下，每一周期向池子补充尽可能多的$\mu op$
并且，**不违反编译正确性的前提下，每一周期它会执行尽可能多的$\mu op$**，因此有了“乱序执行“和”按序提交”，t它们的目的只是尽可能地执行更多的$\mu ops$
所以虽然我们看上去T1和T2按照这样的顺序交给了cpu，但是**cpu实际上是按照一个截然不同的顺序执行这些指令**，因此才出现了很奇怪的0-0结果

***处理器也是编译器***，因此它也只需要满足内存的最终一致性就完事了（最终所有处理器对共享内存的访问会收敛到一致的状态）
但是：
**满足单处理器eventual memory consistency的执行，在多处理器上可能无法序列化！**
因为处理器在内部通过指令重排等优化加快执行，并且保证它的最终一致性，这件事对于单处理器系统来说是不会引发冲突的，**能够最终得到一定顺序的执行序列（能够序列化）**；然而对于多处理器系统，不同处理器之间**存在缓存和执行顺序的差异**，尽管能够保持最终一致性，但是处理器间的执行顺序可能无法按预期进行一致不便的序列化

在上面的例子，`write_x_read_y()` 函数中：
```
  asm volatile(
    "movl $1, %0;" // x = 1
    "movl %2, %1;" // y_val = y
    : "=m"(x), "=r"(y_val) : "m"(y)
  );
```
在这里，第一步执行的`movl $1, %0;`将立即数`1`储存到`x`这个变量中，处理器需要访问`x`所在的地址，此时**处理器需要从最近一级的缓存中查找是否存在x的地址**

注意到另外一个操作，在执行`FLAG_XOR()`的时候，其中的`atomic_fetch_xor(&flag, val)` 是一种原子操作，它执行异或运算并将结果写入内存
由于 `flag` 是一个共享变量，多个线程可能同时对其进行原子操作，此时操作系统和硬件采取缓存一致性协议来**确保所有处理器的缓存内容一致**，当对`flag`进行修改时，其他处理器需要使它们缓存中的副本失效，重新从内存中读取它的值，这通过**直接使得缓存行失效**实现

因此，**每次执行`movl $1, %0`的时候，几乎都会发生cache miss（因为处理器频繁地让缓存行失效）**，这就需要一定时间去下一级缓存中寻找x的地址，**发生了延迟写入**
但是发生了cache miss时，处理器依然可以看到下面一行的`movl %2, %1`，并且缓存行依然处于共享状态，由于乱序执行，并且数据依赖性也被满足了，处理器完全可以将这一行的$\mu op$先执行，也就是先读出y的值，因此处理器看到的是y=0
然后，另外一个处理器也发生了类似的延迟写入+超前读出，因此又看到了x=0，这就是0-0输出会出现的原因

### 2.2.5 宽松内存模型

导致2.2.4中问题发生的原因是x86系统采用宽松内存模型：
该模型大致可以描述为下图：
![[Pasted image 20250311194808.png]]
对于每个线程，它们都有一个write buffer，每次写入，都是先写入write buffer，然后经过任意长的时间后才被写进共享内存中

很可惜，这已经是市面上最好的内存模型了：对于arm等系统，它们内存模型的一致性甚至会更差：每个线程都有一份内存的副本，不同线程的内存副本之间任意进行同步（这个一致性更弱了）

### 2.2.6 实现顺序一致性

当然，处理器实现顺序一致性并不是不可能的，在内联汇编中有一条`mfence`指令，**只需要在写入指令之后增加一条`mfence`指令，处理器就会等到保证将值写入共享内存之后再执行下一条指令**：
```
asm volatile(
    "movl $1, %0;" // x = 1
    "mfence;"     //保证写入共享内存
    "movl %2, %1;" // y_val = y
    : "=m"(x), "=r"(y_val) : "m"(y)
  );
```
将所有写入作这样的改进，再编译一下并统计，就可以得到下面的结果：
```
normist@MIST:~/Learn/NJU-OS$ ./a.out | head -n 100000 | sort | uniq -c
  59098 0 1 
    782 1 0 
  40120 1 1 
```
至少在现在这个数量级下，没有出现0-0的输出了，但是错误永远可能出现在下一次执行中，这就是多处理器编程的恐怖之处

## 2.3 总结

“多处理器编程：从开始到放弃”，我们放弃的不是多处理器编程，而是放弃我们对旧版本的单处理器编程的各种性质的假设
现代处理器就是一个动态的数据流分析器，带着这样的理解去理解处理器的barrier等机制会更简单

当然，多处理器编程依然是很难的事情，需要有很深的经验和对内存模型的了解，多练习，多写程序，摒弃旧的理解，才是掌握它的正确方法

# Lecture 3. 理解并发程序执行

本讲目标：阅读、理解并发程序的行为，画出状态机理解并发程序

## 3.1 互斥

### 3.1.1 互斥算法

互斥：**保证两个线程不能同时执行一段代码**
在之前的求和程序中，我们知道由于线程的同时执行等原因，它的行为和我们希望的行为截然不同，在这个基础上，我们希望进行改正，**对希望不会发生并发的部分进行一个“上锁”，保证两个线程不会同时执行它**
具体来说，希望有类似于下面这样的一对魔法函数：
```
void Tsum() {
	for (int i = 0; i < N; i++) {
		lock();
		sum++;
		unlock();
	}
}
```
将我们希望保持互斥的基本操作包裹起来，一旦一个线程的`lock()`进行返回，那么其他线程就不能和它并发了，只有等该线程`unlock()`返回之后，才能开始下一个线程的`lock()`

我们假设这些基本操作就是对内存的读和写，并且对内存的读写可以保持顺序性和原子性（这一点可以通过定义成volatile，或者使用全内存屏障`___sync_synchornize()`包裹对应内存操作代码，来禁止指令重排、确保内存一致性）

### 3.1.2 失败的尝试

下面是一个失败的实现：
```c
int locked = UNLOCK;

void critical_section() {
retry:
	if (locked != UNLOCK) {
		goto retry;
	}
	locked = LOCK;

	//这里是我们希望执行的操作，例如sum++

	locked = UNLOCK;
}
``` 
这个程序的错误其实类似于上一讲中的山寨支付宝的错误，我们画个状态机来理解一下：

在理想的状态下，最开始的状态应该是locked = UNLOCK，假设有两个线程T1和T2，如果执行T1，此时pc_1=1，pc_2=0，但是还没有将locked锁上，即locked 还是=UNLOCK；然后下一次执行如果执行T2，那么pc_1=1，pc_2=1，locked = UNLOCK
可以发现，这个时候，两个线程看到的locked都是UNLOCK，两个线程都进入了操作代码，这个锁完全没有起到作用

这段代码错误的根本原因是**locked变量的检测（`if (locked != UNLOCK)`）和写入（`locked = LOCK;`）的两步操作不是原子的，它们之间会被中断**，这就导致了这个互斥协议的作用形同虚设


### 3.1.3 Peterson算法：正确性不明的奇怪尝试

Peterson算法使用的模型是共享内存模型：有很多个变量，每次可以`store`一个值到变量里，或是`load`读取一个变量的值
理解算法的最快方法是将线程想象为物理世界中真实存在的人，每个人的脑袋里都有自己的私有状态，空间就是共享内存，其中有很多个“牌子”，它们可以被人改变状态（`store`），也可以被人看到状态（`load`），我们看到的状态一定是过去变成的，但是它并不一定能代表现在这个时刻，或是回头做些什么事情时的状态

假设A和B争用厕所的一个隔间：
- 想进入隔间之前，A/B都需要先举起自己的牌子（`store`一个本地的变量）
	- A希望上厕所时：确认牌子举好以后，向厕所门上贴上“B正在使用”的标签（厕所门上的标签是一个共享变量，用`store`写入），不管标签原来有没有标志，下同
	- B希望上厕所时：确认牌子举好以后，向厕所门上贴上“A正在使用”的标签
- 然后，**如果对方的牌子是举起来的状态，==并且==厕所门上的标签写的不是自己的名字，那么进行等待**，否则可以进入隔间
- 每个人上完厕所，出隔间之后，把自己的牌子放下来

这个协议看上去有点绕，但是直观地看来似乎没什么问题：
核心思想就是“手快有，手慢无”，因为一个人贴的标签总是写着另一个人的名字，那么假设A快B慢，A贴上了B的名字，B覆盖地贴上了A的名字，那么虽然看上去是谦让对方，但是实际上还是以贴标签的顺序决定上厕所的顺序

这么粗糙的”证明“其实不好，因为看上去奇怪但有道理的思路很可能是危险的，还是要测试一下比较好
一个人工的方法是把状态机的图给画出来，每一步执行都可能有两个线程，枚举五元组$(PC_1, PC_2, x, y, turn)$，其中PC是线程的程序计数器，一个线程每运行一步对应的PC加一；x和y是两个人的举手状态，1是举手，0是放下；turn则是门上的标签，写着两个人其中一个的名字

列举的时候，我们就可以画出一个“环”（除了PC以外的其他状态在某一步执行后回到了之前出现的一个状态），而且线程进入执行状态的顺序和它们举手的顺序是一致的，因此大致可以知道是正确的

画状态机实在是很消耗精力的事情，为什么不能用程序帮助我们自动化这个过程呢？当然可以，事实上，课程为我们提供了一个[很好的python脚本](https://jyywiki.cn/pages/OS/2022/demos/model-checker.py)，它能为我们绘出程序整体的状态空间（这个程序有150行，用c或java写可能就要上千了）
这个程序利用python的特性：**使用`yield`关键字可以创建一个函数的生成器对象，只要函数中存在`yield`关键字就成为了一个生成器，它能够通过被调用`__next__()`方法得到每次执行一次函数后的被`yield`关键字修饰的对象的状态**
使用下面的声明来创建一个或多个生成器对象：
```python
T1 = function()
T2 = function()
...
# function就是那个含有被yield关键字修饰的状态变量的函数
```
T1和T2相当于成为了两个线程，每次调用：
```python
T1.__next__()
```
的时候，相当于让T1往前运行了一步，输出所有`yield`状态。对T2也是一样的，这两个线程各自有它们的状态

有了这个特性，进行状态的枚举其实就是一个bfs的过程了：使用队列，把初始状态入队，在队列非空时每次取出一个状态，枚举它的线程走一步后到达的状态，将没有出现过的状态入队即可。当然还要维护一个边集用于记录状态之间的连接关系
这启示我们完全可以把状态机问题转换为图论问题，在枚举并建图以后，我们可以使用很多算法来进行一些在意的性质的检测，例如强连通分量的检测、环的检测等等
如果真实的状态空间太大（状态太多，memory的值太大等等），完全可以


另外，像之前的例子一样，还需要注意现代处理器执行程序时进行的优化不会导致产生莫名其妙的错误，这可能需要写程序来验证了
不妨用下面这个枚举程序来检验：
```c
#include "thread.h"

#define A 1
#define B 2

atomic_int nested;
atomic_long count;

void critical_section() {
  long cnt = atomic_fetch_add(&count, 1);  //计数器用来检查是否有两个线程进入了临界区，原子地为它加一
  assert(atomic_fetch_add(&nested, 1) == 0);  //这个断言也用于检测是否有两个进程进入临界区，实际上，这个断言被触发了
  atomic_fetch_add(&nested, -1);  //原子减1
}

int volatile x = 0, y = 0, turn = A;

void TA() {
    while (1) {
/* PC=1 */  x = 1;  // 相当于“举起牌子”
/* PC=2 */  turn = B;  //把标签贴成对方的名字
/* PC=3 */  while (y && turn == B) ;  //如果对方举起了牌子且标签名字不是自己的，等待
            critical_section();
/* PC=4 */  x = 0;
    }
}

void TB() {
  while (1) {
/* PC=1 */  y = 1;
/* PC=2 */  turn = A;
/* PC=3 */  while (x && turn == A) ;
            critical_section();
/* PC=4 */  y = 0;
  }
}

int main() {
  create(TA);
  create(TB);
}

```
在执行这个程序一段时间后，编译器显示断言被触发了

```c
#include "thread.h"

#define A 1
#define B 2

#define BARRIER __sync_synchronize()

atomic_int nested;
atomic_long count;

void critical_section() {
  long cnt = atomic_fetch_add(&count, 1);
  int i = atomic_fetch_add(&nested, 1) + 1;
  if (i != 1) {
    printf("%d threads in the critical section @ count=%ld\n", i, cnt);
    assert(0);
  }
  atomic_fetch_add(&nested, -1);
}

int volatile x = 0, y = 0, turn;
void TA() {
  while (1) {
    x = 1;                   BARRIER;
    turn = B;                BARRIER; // <- this is critcal for x86
    while (1) {
      if (!y) break;         BARRIER;
      if (turn != B) break;  BARRIER;
    }
    critical_section();
    x = 0;                   BARRIER;
  }
}

void TB() {
  while (1) {
    y = 1;                   BARRIER;
    turn = A;                BARRIER;
    while (1) {
      if (!x) break;         BARRIER;
      if (turn != A) break;  BARRIER;
    }
    critical_section();
    y = 0;                   BARRIER;
  }
}

int main() {
  create(TA);
  create(TB);
}

```


现代计算机科学已经对互斥这个问题有了很深刻的了解，深入阅读材料：[myths about the mutual exclusion problem](https://zoo.cs.yale.edu/classes/cs323/doc/Peterson.pdf)

# Lecture 4.  并发控制：互斥

Peterson算法实现互斥的效率很低，本讲将介绍性能更优的两种锁：**自旋锁和互斥锁**
锁：如果某个线程持有锁，则其他线程的`lock`函数无法返回
## 4.1 共享内存上的互斥

实现互斥的最大困难是**不能同时读/写共享内存**：
- `load`的时候不能写，只能“看一眼就马上闭眼”（获取一个数据后就不再在意内存），因此：
	- 看到的东西马上就过时了：我们不知道获取的数据在下一个操作之前是不是就过时了
- `store`的时候不能读，只能“闭着眼睛动手”（写数据的时候没办法知道对象原来的数据是多少），因此：
	- 不知道把什么改成了什么：无法确定原有状态以做出不同响应

除了提出算法来解决问题之外，另一种解决办法是改变假设：软件不够，硬件来凑

### 4.1.1 自旋锁

#### 原子指令，xchg的实现
**假设硬件能够为我们提供一条“瞬间完成”的读+写指令**，也就是说把所有其他人的工作都暂停（不许读或写了），然后立刻让对应线程执行自己的工作，执行完成后重启其他人的工作
这就意味着，所有线程的地位是对等的，每个线程都可以执行这样的指令，让其他人暂停，因此**这就需要我们在有多个线程申请执行这样的指令时规定一个顺序**，例如先到先执行

前面的课程中，计算多线程求和的正确版本中出现了如下的内联汇编指令：
```
asm volatile("lock addq $1, %0" : "+m"(sum));
```
这里`addq`指令前面使用了`lock`进行修饰，它的意思是在执行这条指令的时候其他人什么都不要干——实际上，只需要保证执行这条指令的时候其他人不会对`sum`进行操作即可（实现上的优化）

更多的原子指令可以[看这里](https://en.cppreference.com/w/cpp/header/stdatomic.h)，这些原子指令中，能够帮助我们实现互斥的最简单的一条指令叫做`xchg`：
```c 
int xchg(volatile int *addr, int newval) {
	int result;
	asm volatile ("lock xchg %0, %1" : "+m"(*addr), "=a"(result), "1"(newval));
	//实际上，xchg指令自带lock前缀，不用写lock也可以
	return result;
}
```
这条指令的作用是提供一个最小的`load`和`store`：它**先把一个变量的值读出来，然后写进去自己的值**，这个操作是原子的
也就是所谓的“交换”名字的含义：它将两个数值进行交换，既能储存原来的值，又能写进去给出的值

#### 使用`xchg`实现互斥
在Peterson算法的基础上，如果我们拥有了原子指令，那协议的过程就不再需要那么复杂了，用上厕所为比喻，如果我们希望协调若干人的上厕所问题：
- 首先，在厕所门口放一个篮子（共享变量）
  - 初始时，篮子内有一把钥匙

对于想上厕所的人：
- 对共享变量内的值进行原子的`xchg`交换：
  - 看一眼篮子内有什么东西（可能是钥匙或其他东西）
  - 将自己的东西放到篮子内（覆盖掉之前有的任何东西）
- 进行完交换之后
   - 如果自己拿到了钥匙，那么可以进入厕所
   - 如果没有拿到钥匙，直到拿到钥匙为止才能进厕所

对于出厕所的人：
- 将厕所钥匙放到篮子内部

#### 自旋锁的代码
根据之前的协议，我们将它翻译为代码：
```c
int table = YES; //篮子，yes即为钥匙

void lock() {
retry:
	int got = xchg(&table, NOPE);
	if (got == NOPE)
		goto retry;
	assert(got == YES);
}

void unlock() {
	xchg(&table, YES);
}
```
这就是非常简单的一个自旋锁的实现，它甚至可以更精简：
```c
int locked = 0;
void lock() { while (xchg(&locked, 1)); }
void unlock() { xchg(&locked, 0); }
```
（后者是我们经常会看见的写法）

在python实现的`model-checker`中，**利用生成器所作出的假设是每一行代码的执行是原子的，而行与行之间可能并发**，因此，python中的原子交换就更简单了：
```python
x, y = y, x
```

下面就是两个线程T1和T2进行互斥的python代码
```python
class Spinlock:
    locked = ''

    @thread
    def t1(self):
        while True:
            while True:
                self.locked, seen = '🔒', self.locked # 原子交换
                if seen != '🔒': break
            cs = True
            del cs
            self.locked = ''

    @thread
    def t2(self):
        while True:
            while True:
                self.locked, seen = '🔒', self.locked
                if seen != '🔒': break
            cs = True
            del cs
            self.locked = ''

    @marker
    def mark_t1(self, state):
        if localvar(state, 't1', 'cs'): return 'blue'

    @marker
    def mark_t2(self, state):
        if localvar(state, 't2', 'cs'): return 'green'

    @marker
    def mark_both(self, state):
        if localvar(state, 't1', 'cs') and localvar(state, 't2', 'cs'):
            return 'red'  # 红色状态代表有两个线程同时进入了临界区

```

可以使用之前的model-checker程序处理这段程序，初步地检查理论上是否会出现同时进入临界区的情况
model-checker认为每一行的执行都是原子的，因此绘制出来的状态机看上去是没问题的。那么，处理器上的代码的原子性是否真的能通过xchg操作保持？
答案是肯定的，因为对于多个执行了`lock()`的线程，处理器能够确保它们之间排出一个先后顺序，并且每个线程内部的指令流不会被乱序执行，且后面的执行能够看到所有前面发生的变化

排序的实现就是并行性的消灭，原子指令确保了这一点的实现

### 4.1.2 原子指令的硬件（x86的硬件包袱）

现代计算机服务器的主板上往往会有两个或多个CPU，对于多个CPU，它们访问内存时的行为是什么样子的？
如果一个cpu需要往内存中写数据，它不希望另外的CPU干扰它的工作，此时就会在内存中加一把锁，告诉其他CPU不能访问；如果有多个CPU都想访问内存，此时由总线bus来为它们排序，规定一个顺序，只有前面的CPU用完释放了锁之后，后面的才能上锁接着用
这样的实现是从硬件级别上完成的，在X86中，`lock`是一个指令前缀，在处理读完这个前缀的字节时，就会立刻进行一个总线锁，然后再处理后面的指令字节

而今天的计算机中，CPU和内存之间又新增了高速缓存，并且，缓存不是在主板中的（如果是在主板上，直接锁住CPU与缓存交界就行了），而是在cpu里的，这个时候lock的操作就很麻烦了（把指令预取到缓存中代表该cpu接下来要执行的行为，但是如果不加以管控的话会出现并发问题）

在这种情况下，访问相同对象的指令可能同时出现在多个CPU的缓存中，为了使只有一个CPU能进行内存的lock和访问，必须想办法使得其他CPU的缓存中的访问命令去除掉，也就是**保持缓存一致性**的重要性
为了实现这一点，需要将所有CPU的L1缓存用总线连接起来，在每次遇到内存访问时，将其他CPU缓存中相应的指令去除
这就是**在L1缓存层保持一致性**，相当于是每一个缓存行都有各自的锁，需要注意缓冲区存储和乱序执行带来的问题
MESI分别代表了脏值、独占访问、只读共享和不拥有缓存行四种模式，L1缓存行根据这些状态进行协调

### 4.1.3 RISC-V 的另一种原子操作设计

回过头来思考一下我们对原子指令的需求是什么？对于原子指令，我们往往需要一个读操作`load(a)`和一个写操作`store(a, B)`，对常见的原子操作：
- 原子检测和设置标志位`test-and-set`
	- `reg = load(x); if (reg == XX) {store(x, YY); }`
	- 将x位置处的变量`load`读取，然后若满足一定条件，在x位置写入值YY
- 带锁交换`lock xchg`
	- `reg = load(x); store(x, XX);`
	- 先将x位置的值存储到寄存器reg中，然后向x写入值XX
- 带锁递增`lock add`
	- `t = load(x); t++; store(x, t);`
	- 首先将x值存储到t中，在t中对值递增，然后把t的值回写到x中

原子指令的目的是我们希望在改变一个状态的时候确保我们真的将它进行了改变，达成了我们想要的状态，本质上都是以下三种操作：
- `load` 读操作
- `exec` 处理器对本地寄存器的计算操作
- `store`写操作

#### LA/SC机制

为此，RISCV设计了**LA/SC：Load-Reserved/Store-Conditional 机制**
它的思想是：对于观测一个共享内存中的值，然后做一些本地计算，最后写入共享内存的整个操作，本地计算的部分是不重要的，重要的是共享状态的观测和写入，因此本地计算的性能

因此，对此需求增加了以下机制：

- `LR`：在从共享内存中读取值的时候，为内存加上一个标记`reserved`，这个标记会在所有其他处理器写入的情况下被消除
```
lr.w  rd, (rs1)     # 读取内存M中rs1处的值，rd是存值寄存器
  rd = M[rs1]
  reserve M[rs1]
```

读取完成之后，会将值拿到本地寄存器进行计算，这部分不涉及共享内存的读写。在计算完成之后，准备向共享内存写入，此时需要检查`reserved`标记
- `SC`：如果标记`reserved`没有被解除，说明该内存没有被其他线程访问，此时可以知道真实值相比本地计算所用值没有改变，可以放心进行写入
```
sc.w  rd, rs2, (rs1)  # rd是标志位寄存器，rs2是写入值
if still reserved: 
	M[rs1] = rs2     # 只有在reserved没被消除时，才向内存M的rs1处写入rs2值
	rd = 0           # 设置返回状态为0
else:
	rd = nonzero    # 没有标记，设置返回状态为非0
```

如果标记被解除了，那就从lr开始重新再来一次，直到sc成功

借助这样的机制，可以实现所有原子读写以及更麻烦的原子操作。例如，下面是原子比较并交换操作`cas`的LR/SC实现
`cas`操作的c语言描述，也就是先让`addr`处的原值与`cmp_val`比较，如果相等，更新为新值`new_val`：
```C
int cas(int *addr, int cmp_val, int new_val) {
	int old_val = *addr;
	if (old_val == cmp_val) {
		*addr = new_val; 
		return 0;
	}
	else return 1;
}
```
对应的原子实现，使用`lr.w`和`sc.w`来达成：
```lr/sc
cas:
	lr.w    t0, (a0)     # 读取old_val，同时设置reserved
	bne     t0, a1, fail   # 检测old_val(t0)与a1值是否相同，若不同，跳转fail处
	sc.w    t0, a2, (a0)   # 尝试向内存a0处写值，此时t0存储的old_val不需要使用了，使用t0作为存储返回状态标志的寄存器
	bnez    t0, cas       # 若返回值不为0,跳转回cas开头，重新读+尝试写
	li      a0, 0        # 若返回值是0,那就说明sc.w成功完成写操作，a0重用于储存返回状态0
	jr      ra        # return
fail:
	li      a0, 1    # 返回状态1,说明值不同，无法交换
	jr      ra       # return
```

这样的机制还可以用于检测拥堵的发生，在真正的CPU代码实现中也广泛应用，不断尝试重新开始实际上也就是一种自旋锁

### 4.1.4 Scalability：并发编程的性能衡量

自旋锁的性质就是一把钥匙，多个线程不断与其交换，只有得到钥匙的人才能进入临界区
这样的机制会带来一种缺陷：
- 性能问题：每次自旋时都会进行一个原子交换指令，一个`lock`会触发所有处理器之间的同步，尽管同步比较高效，但仍然有开销
- 当有两个线程同时争抢一把锁时，总会有一个线程在忙等待，不断地goto死循环，处理器一直在工作
- 哪怕只有一个cpu,它仍然可以分时运行多个线程，这个时候有钥匙的线程和没钥匙在忙等待的线程互相分时，那么就会出现操作系统将获得自选锁钥匙的线程切换出去，而去执行在忙等待的那个线程，产生了100%的资源浪费

当我们考虑锁的实现的时候，除了是否正确之外，还需要考虑其性能和适用条件（类似于时空复杂度分析），在多处理器并发编程中，我们引入另一个衡量参数：scalability（可扩展性）
对于同一个计算任务，时间（CPU周期：即CPU执行一个基本操作的时间）和空间（映射内存：操作系统在进程的地址空间中为其分配的虚拟内存区域）会**随着处理器数量的增长而变化**

scalability就描述了一个系统在增加硬件资源（例如，增加CPU核心或处理器数量）时，性能是否能按比例地提升，或者是否存在性能的瓶颈

下面是一个示例程序：
```c
#include "thread.h"
#include "thread-sync.h"

#define N 10000000
spinlock_t lock = SPIN_INIT();

long n, sum = 0;

void Tsum() {
  for (int i = 0; i < n; i++) {
    spin_lock(&lock);    //自旋锁
    sum++;
    spin_unlock(&lock);
  }
}

int main(int argc, char *argv[]) {
  assert(argc == 2);
  int nthread = atoi(argv[1]);  // 线程数量参数
  n = N / nthread;   // 将N次递增任务平均分给每个线程
  for (int i = 0; i < nthread; i++) {
    create(Tsum);
  }
  join();
  assert(sum == n * nthread);
}

```
这个程序将一个N次的求和任务平均分到n个线程上，使用自旋锁维护临界区的访问顺序
随着输入的参数`nthread`改变，我们可以使用`time`命令查看到运行时间的变化：
```bash
normist@MIST:~/Learn/NJU-OS$ time ./a.out 1

real	0m0.184s
user	0m0.181s
sys	0m0.002s
normist@MIST:~/Learn/NJU-OS$ time ./a.out 2

real	0m0.548s
user	0m1.057s
sys	0m0.002s
normist@MIST:~/Learn/NJU-OS$ time ./a.out 4

real	0m1.558s
user	0m5.630s
sys	0m0.006s

```
可以发现，同样的工作量，线程越多性能反而越差了：
- 使用原子指令本身就会带来开销
- 处理器的低效使用
- 空转资源浪费

度量多处理器的性能实际上很困难，CPU首先是一个动态规划的功耗，根据散热等不同会有不同的策略，以及系统中其他的进程干扰
所谓的benchmarking crime就是通过调节很多其他因素，导向一个看上去很好的测试结果，但是实际上的性能远不如这个结果的不公正的手段


### 4.1.5 自旋锁的适用场景

- 临界区几乎不拥堵：多个线程之间锁的争抢情况很少，例如，一个任务分给多个线程，维护一个队列，每个线程**从队列中取任务（这个过程就是访问临界区）**，每个线程从队列中取任务所花费时间远远小于线程执行自己的任务的时间，所以取任务时争抢临界区的锁的时间很短，几乎不会发生拥堵
- 持有自选锁的时候，禁止执行流的切换：这点应用程序是做不到的，操作系统并不会允许进程做这么危险的行为，但是在**操作系统内核的实现**中对某些保护类型数据结构会使用这种禁止切换机制（虚拟机的操作系统的禁止切换实际上没有真正禁止……）

自旋锁的特点：
- 线程直接共享locked, 这会带来更快的fast path：一旦xchg成功，就立刻进入临界区，开销很小
- 但xchg失败就会浪费cpu进行自旋等待（slow path很慢）


### 4.1.6 睡眠互斥锁Mutex

因此，如果我们希望在写共享内存时不被打断，我们会希望CPU与其忙等待，不如让别的不等待的线程先执行
但是这个“让”并不是C语言代码可以做到的，正如我们之前所说，C代码只能计算，想要实现其他功能需要操作系统来完成，即需要系统调研
因此，我们**把锁的实现放到操作系统中**就好了：
```
syscall(SYSCALL_lock, &lk)
# 试图exch获得锁lk,若成功获得锁（*lk 不为 locked），进行返回；如果失败，切换到其他进程
```
```
syscall(SYSCALL_unlock, &lk)
# 把锁进行释放，如果有等待锁的线程，就进行唤醒
```
操作系统上的线程想进行长临界区的互斥，可以结合系统调用

此时操作系统类似于一个管理员的身份：
- 对于先申请锁的线程
	- 操作系统允许它获得锁，进入临界区
	- 把此时`*lk`的状态设置为`used`
- 对于后申请锁的线程
	- 不能进入临界区，需要排队等待
	- 把线程**放入等待队列**，执行线程切换，使得线程`yield`（让线程自愿放弃cpu的控制权，不再执行这个线程）
- 对于使用完共享内存，出临界区的线程：
	- 将锁交还给操作系统，操作系统再把锁给其他进程
	- 如果此时等待队列非空，从等待队列中取出一个线程，允许其执行
	- 如果等待队列为空，则`*lk = unused`

这就是睡眠锁：通过系统调用访问locked状态，如果线程申请锁没有成功，进入yield状态，即放弃自己被cpu执行的权利，睡眠直到它是队列中下一个被允许进入临界区的线程
睡眠锁的性质在于：
- 更快的slow path：上锁失败以后，线程放弃cpu的执行权，不再进行占用
- 更慢的fast path：即使是上锁成功，都需要经过syscall进出内核，系统调用指令总是会需要一些时钟周期来进行检查等操作

### 4.1.7 Futex：我全都要

自旋锁和睡眠锁在fast path和slow path的性能优劣不同，既然它们各有自己的优势，那我们不妨在不同的场景下调用不同的策略，以达到最佳性能

这种方式是操作系统中最常用的优化技巧：我们只看平均情况，只要大部分情况是好的，那就是好的
对于两种情况：
- fast path：在线程申请到锁的时候，执行一条原子指令，上锁成功后立即返回
- slow path：上锁失败，执行系统调用指令进行睡眠

例如下面就是使用POSIX线程库中的互斥锁版本的代码：
```c
#include "thread.h"
#include "thread-sync.h"

#define N 10000000
mutex_t lock = MUTEX_INIT();

long n, sum = 0;

void Tsum() {
  for (int i = 0; i < n; i++) {
    mutex_lock(&lock);
    sum++;
    mutex_unlock(&lock);
  }
}

int main(int argc, char *argv[]) {
  assert(argc == 2);
  int nthread = atoi(argv[1]);
  n = N / nthread;
  for (int i = 0; i < nthread; i++) {
    create(Tsum);
  }
  join();
  assert(sum == n * nthread);
}
```
使用time测量运行时间：
```bash
normist@MIST:~/Learn/NJU-OS$ time ./a.out 1

real	0m0.212s
user	0m0.210s
sys	0m0.002s
normist@MIST:~/Learn/NJU-OS$ time ./a.out 2

real	0m0.622s
user	0m0.622s
sys	0m0.565s
normist@MIST:~/Learn/NJU-OS$ time ./a.out 4

real	0m0.714s
user	0m1.020s
sys	0m1.607s

```
在多线程情况下确实有所优化
不妨使用`strace`追踪一下每个线程的系统调用，使用`-f`选项
```
starce -f ./a.out 4
```


一个非常简单的futex实现模拟程序如下：
```python
class Futex:
    locked, waits = '', ''

    def tryacquire(self):
        if not self.locked:
            # Test-and-set (cmpxchg)
            # Same effect, but more efficient than xchg
            self.locked = '🔒'
            return ''
        else:
            return '🔒'

    def release(self):
        if self.waits:
            self.waits = self.waits[1:]
        else:
            self.locked = ''

    @thread
    def t1(self):
        while True:
            if self.tryacquire() == '🔒':     # User
                self.waits = self.waits + '1' # Kernel，使用字符串作为等待队列
                while '1' in self.waits:      # Kernel
                    pass
            cs = True                         # User
            del cs                            # User
            self.release()                    # Kernel

    @thread
    def t2(self):
        while True:
            if self.tryacquire() == '🔒':
                self.waits = self.waits + '2'
                while '2' in self.waits:
                    pass
            cs = True
            del cs
            self.release()

    @thread
    def t3(self):
        while True:
            if self.tryacquire() == '🔒':
                self.waits = self.waits + '3'
                while '3' in self.waits:
                    pass
            cs = True
            del cs
            self.release()

    @marker
    def mark_t1(self, state):
        if localvar(state, 't1', 'cs'): return 'blue'

    @marker
    def mark_t2(self, state):
        if localvar(state, 't2', 'cs'): return 'green'

    @marker
    def mark_t3(self, state):
        if localvar(state, 't3', 'cs'): return 'yellow'

    @marker
    def mark_both(self, state):
        count = 0
        for t in ['t1', 't2', 't3']:
            if localvar(state, t, 'cs'):
                count += 1
        if count > 1:
 
```


# Lecture 5. 并发控制：同步

## 5.1 线程同步

同步：两个或以上的随时间变化的量，在变化过程中保持一定的相对关系

例如，在手机相册和云端之间进行同步，就是将云端的图像情况与手机相册的图像文件情况保持相同； 在硬件上也有同步电机和同步电路等结构存在

相对应的就是异步，也就是几个量不保持一定相对关系

并发环境中，程序的步调很难保持完全一致，并发程序的同步的程度会更弱一些：
- 线程同步：在**某个时间点**共同达到了互相已知的状态

例如，两个并发程序各自在执行一个很长的计算，我们不知道它们什么时候结束，但是可以约定在某个时间点，一个线程等待另一个线程做完某件事后才继续执行，谁先完成，谁就要等待对方完成
这就是同步问题了，我们的目的是编程实现一个正确的协议来解决这样的问题

## 5.2 生产者-消费者问题

99%的实际并发问题都可以使用生产者-消费者模型解决
假设我们有以下两个函数：
```c
void Tproduce() { while (1)  printf("("); }
void Tconsume() { while (1)  printf(")"); }
```
在第一讲中我们说过两个线程分别打印a和b的例子，当时a和b会随机地交替出现
现在我们对这两个打印左右括号的函数，在`printf`的前后处增加代码，使得打印的括号序列满足：
- 给定n，n代表括号嵌套的最深层数，即最多只能连续出现n个左括号
- 一定是一个**合法的括号序列的前缀**：每个左括号要么各自对应一个右括号，要么不对应有右括号，但是不能出现单独的右括号

例如，下面是合法输出：
- n = 3, `((())())(((`
非法输出：
- n = 3, `(((())))` 或  `(()))`

初步可以分析这个问题的同步点在于：
- 对于左括号：只有存在空位（连续左括号个数小于等于n-1）时，才可以打印左括号
- 对于右括号：只有前面存在没有配对的左括号时才可以打印右括号

### 5.2.1 问题分析
括号问题可以被归纳为以下两类线程：
- 左括号：负责生产资源（任务），把任务（左括号）放到队列中
- 右括号：从队列中取出资源（任务）进行执行（打印出与左括号对应的右括号，没有左括号就不能打印右括号）

在实际的并发开发中，如果我们希望执行一个很大的计算的并行化，就需要**把一个大问题切分为多个小任务**，分配到多个线程上
就像一个口袋，生产者相当于是向口袋中扔进一个任务，消费者则是从口袋中摸出一个任务，当有一个空闲的线程时，就可以从包里摸出一个任务来执行，调度器负责调度这个过程，这样可以很好地利用各个线程的计算能力

一个很直观的解决方案是，为了防止两个线程同时访问“口袋”取任务，每个线程**每次访问时先上一个互斥锁**；对于生产者，也是访问前先上个互斥锁，检查“口袋”是否已经满了，如果还没满，那就继续放入任务
下面的代码就是这个解决方式的一段实现
```c
#include "thread.h"
#include "thread-sync.h"

int n, count = 0;
mutex_t lk = MUTEX_INIT();

void Tproduce() {
  while (1) {
retry:
    mutex_lock(&lk);   // 生产者先上锁
    if (count == n) {  // 如果“口袋”已经满了
      mutex_unlock(&lk);  // 解锁，允许消费者取任务
      goto retry;  // 一直自旋，直到可以放入
    }
    count++;
    printf("(");
    mutex_unlock(&lk);
  }
}

void Tconsume() {
  while (1) {
retry:
    mutex_lock(&lk);  // 消费者上锁
    if (count == 0) {  // 如果“口袋”为空，无法取得任务
      mutex_unlock(&lk);
      goto retry;  // 自旋
    }
    count--;
    printf(")");
    mutex_unlock(&lk);
  }
}

int main(int argc, char *argv[]) {
  assert(argc == 2);
  n = atoi(argv[1]);
  setbuf(stdout, NULL);
  for (int i = 0; i < 8; i++) {
    create(Tproduce);
    create(Tconsume);
  }
}
```

再次重申，学会一门脚本语言的一个好处是当你写好一个自己觉得可以的程序时，你能够通过脚本批量检查输出的合法性，用来初步检测自己的程序是否在大数据量的情况下不出错
自测程序的编写是必要的！

### 5.2.2 条件变量：万能同步方法
对于上面的解决方法，无论“口袋”是否为满或是否为空，线程都会访问它进行一次查看，这会导致性能会被浪费（有一个类似自旋的过程）

我们的几个同步问题都有一个共性，先上锁->检测条件->若不满足条件，解锁自旋
如果在条件不成立时，线程可以不进入自旋，而是原子地进行sleep，当条件满足时能够被喊醒，这样自旋的开销就被降低了

条件变量cv就是这样一个机制，它的api包括：
- `wait(cv, mutex)`：
	- 首先调用它时**需要确保已经获得了互斥锁**`mutex`
	- 在条件变量cv下，释放互斥锁mutex，进入睡眠状态
- `signal/notify(cv)`：唤醒单一线程，**不保证唤醒的线程是哪个**，**任何一个睡眠的线程都可能被唤醒**
	- 对于一个睡眠的线程，当另外一个线程可以满足它执行的条件后，将其叫醒
	- `notify`在唤醒线程之后会自动重新试图获取互斥锁
- `broadcast/notifyAll(cv)`：唤醒**所有在睡眠等待的线程**
	- 和`signal/notify`的区别只在它会在线程可以满足执行条件后唤醒所有正在睡眠的线程


利用条件变量实现生产者-消费者，伪代码可以写成以下的形式 ：
```c
void Tproduce() {
  mutex_lock(&lk);  // 生产者上锁
  if (count == n) cond_wait(&cv, &lk);  // 若不满足触发条件，等待，wait时锁会被释放，等到返回以后（即被唤醒时）会试图重新获得锁
  printf("("); count++; cond_signal(&cv);  // 被其他线程唤醒，打印括号，增加计数，唤醒消费者线程（cond_signal）
  mutex_unlock(&lk);  // 解锁
}

void Tconsume() {
  mutex_lock(&lk);  // 消费者上锁
  if (count == 0) cond_wait(&cv, &lk);  // 若不满足触发条件（count 不为0），等待
  printf(")"); count--; cond_signal(&cv);  // 被唤醒，消费，唤醒生产者线程
  mutex_unlock(I&lk);  // 解锁
}
```

#### 有问题的实现：条件变量cv应该有多个？
一个尝试性的压力测试代码如下，我们：
```c
#include "thread.h"
#include "thread-sync.h"

int n, count = 0;
mutex_t lk = MUTEX_INIT();  // 互斥锁
cond_t cv = COND_INIT();  // 共享变量

void Tproduce() {
  while (1) {
    mutex_lock(&lk);
    if (count == n) {
      cond_wait(&cv, &lk);
    }
    printf("("); count++;
    cond_signal(&cv);
    mutex_unlock(&lk);
  }
}

void Tconsume() {
  while (1) {
    mutex_lock(&lk);
    if (count == 0) {
      pthread_cond_wait(&cv, &lk);
    }
    printf(")"); count--;
    cond_signal(&cv);
    mutex_unlock(&lk);
  }
}

int main(int argc, char *argv[]) {
  assert(argc == 2);
  n = atoi(argv[1]);
  setbuf(stdout, NULL);
    create(Tproduce);
    create(Tconsume);
  
}
```
如果运行这段程序会发现，在只有一个生产者和一个消费者，或少量生产者消费者时，代码没问题
但当生产者消费者数量多起来了，例如下面这样创建多个线程：
```c
 for (int i = 0; i < 8; i++) {
	create(Tproduce);
	create(Tconsume);
}
```
不对，好像出问题了，括号个数好像不太对

当执行深度为1的代码时，会出现类似于以下的输出情况：
```
()()())))...（后面省略）
```
显然消费者进程在错误的时间被唤醒了，导致右括号的个数比左括号个数要多

通过输出我们可以往这样的方向猜测，不妨考虑一个单生产者，多消费者的情况：
- 生产者P1、 消费者C1、C2
考虑以下流程：
1. 消费者C1初始时由于`count == 0`，进入休眠状态；C2同理也休眠
2. 生产者P1检测到`count != n`，生产左括号，唤醒**其余线程中的随机一个**（这种情况下是两个消费者的其中一个）后，解锁
3. 消费者C1被唤醒，此时它输出一个右括号，与之前一个左括号对应，随即唤醒**其余线程中的随机一个**，解锁
4. 问题就在这里了：**随机一个被唤醒的线程不是生产者，而是另一个消费者**，唤醒时它不会对当前的count作任何检测，而是直接输出左括号，然后又在其余的线程中随机唤醒一个，这就导致了连续出现右括号的情况

因此，整个问题出现的原因在于**同类唤醒**，例如：
- 生产者唤醒了另一个生产者，导致嵌套括号数量大于n
- 消费者唤醒了另一个消费者，导致右括号个数大于左括号个数
为了解决这个问题，我们能想到的第一种方式是，**应该在==每种==生产者和消费者之中各自设置一个条件变量**，在唤醒时注意生产者只能唤醒消费者，消费者只能唤醒生产者

这种方式似乎可行，实际上也是后面正确的条件变量同步方法的一个来源，只不过后面的形式更加简单，不需要为每种消费者和生产者设置一个条件变量

下面的model-checker脚本展示了多条件变量的过程，利用**脚本的原子性**重写并发代码，而不使用线程库是检测对概念理解程度的好方法，因此也贴上相应代码：
```python
class ProducerConsumer:
    locked, count, log, waits = '', 0, '', ''

    def tryacquire(self):   # 尝试获取锁
        self.locked, seen = '🔒', self.locked  # 原子交换
        return seen == ''  # 返回值判断是否获取成功

    def release(self):  # 释放锁
        self.locked = ''

    @thread
    def tp(self):  # 生产者
        for _ in range(2):
            while not self.tryacquire(): pass # mutex_lock()
            # 自旋锁：一直尝试去获取锁，没得到锁时死循环

            if self.count == 1:  # 如果count值是1,代表已经有一个左括号了
                # cond_wait
                _, self.waits = self.release(), self.waits + '1' 
                # 执行wait：同时（原子）地把锁释放掉并且将自己加入等待队列，队列使用字符串表示，如果有1,那就证明该生产者在等待队列中
                while '1' in self.waits: pass 
                # 如果还在等待队列中，死循环
                while not self.tryacquire(): pass
                # 如果不在等待队列中，需要先重新获得互斥锁才能继续执行

            self.log, self.count = self.log + '(', self.count + 1
            # log代表打印总览，打印一个左括号，把count加一，整个过程都是原子的
            self.waits = self.waits[1:] # cond_signal
            # 从等待队列中去掉'1'这个字符
            self.release() # mutex_unlock()
            # 交出互斥锁

    @thread
    def tc1(self):  # 消费者1
        while not self.tryacquire(): pass

        if self.count == 0:
            _, self.waits = self.release(), self.waits + '2'
            # 字符‘2’代表消费者1位于等待队列中
            while '2' in self.waits: pass
            while not self.tryacquire(): pass

        self.log, self.count = self.log + ')', self.count - 1

        self.waits = self.waits[1:]
        # 去掉的是代表生产者的1,只唤醒生产者
        self.release()

    @thread
    def tc2(self):
        while not self.tryacquire(): pass

        if self.count == 0:
            _, self.waits = self.release(), self.waits + '3'
            # 字符‘3’代表消费者2位于等待队列中
            while '3' in self.waits: pass
            while not self.tryacquire(): pass

        self.log, self.count = self.log + ')', self.count - 1

        self.waits = self.waits[1:]
        self.release()

    @marker
    def mark_negative(self, state):
        count = 0
        for ch in self.log:
            if ch == '(': count += 1
            if ch == ')': count -= 1
 
```

可以看到，生产者和两个消费者使用了不同的字符在等待队列中标识，在唤醒时，消费者只唤醒生产者

#### 条件变量正确的打开方式：持续检测+集体唤醒
回忆一下之前讲的条件变量API：
- `signal/notify(cv)`：唤醒单一线程，**不保证唤醒的线程是哪个**，**任何一个睡眠的线程都可能被唤醒**
	- 对于一个睡眠的线程，当另外一个线程可以满足它执行的条件后，将其叫醒
	- `notify`在唤醒线程之后会自动重新试图获取互斥锁
- `broadcast/notifyAll(cv)`：唤醒**所有在睡眠等待的线程**
	- 和`signal/notify`的区别只在它会在线程可以满足执行条件后唤醒所有正在睡眠的线程

`signal`对它唤醒的线程不作保证，因此会出现同类唤醒导致故障发生的情况

1. 将单次检测`if`改为持续检测`while`
首先，错误的实现中，每个线程的`if`条件判断条件是否成立，**一次`if`检测并不足以保证在不正确的情况下线程一定不会被执行**：
在if分支执行，进入休眠后，若该线程被其他线程唤醒，是不会再回过头对`count`等条件进行进一步检测的，而是直接执行它的操作
为了解决这个问题，我们**将`if`判断改为在`while`的条件判断中不断进行**，即使被唤醒并且拿到锁了，只要条件不成立，仍然要重新进入睡眠
**相当于是取消了对“唤醒线程绝对正确”的信任，靠持续检测确保无错**

2. 将随机单进程唤醒`signal`改为群体唤醒`broadcast`
另外，`cond_signal`导致了线程可能会唤醒它的同类线程，使得故障发生，既然我们已经将`if`改成了不断检测的`while`，那么完全没必要只唤醒一个线程：**同时唤醒所有在等待的线程，由于每个线程都通过`while`不断检测当前是否应该继续等待，所以那些不满足执行条件的线程会继续回到等待情况，而满足执行条件的线程则跳出`while`继续执行**

再次提醒，`cond_wait`会自动在等待结束后尝试重新抢占锁，不需要我们额外执行
正确的代码如下：
```c
#include "thread.h"
#include "thread-sync.h"

int n, count = 0;
mutex_t lk = MUTEX_INIT();  // 互斥锁
cond_t cv = COND_INIT();  // 共享变量

void Tproduce() {
  while (1) {
    mutex_lock(&lk);
    while (count == n) {    // 将if(count==n)换成while(count==n)
      cond_wait(&cv, &lk);
    }
    printf("("); count++;
    cond_broadcast(&cv);   // 将signal改成broadcast
    mutex_unlock(&lk);
  }
}

void Tconsume() {
  while (1) {
    mutex_lock(&lk);
    while (count == 0) {  // 将if(count==0)换成while(count==0)
      pthread_cond_wait(&cv, &lk);
    }
    printf(")"); count--;
    cond_broadcast(&cv);   // 将signal改成broadcast
    mutex_unlock(&lk);
  }
}

int main(int argc, char *argv[]) {
  assert(argc == 2);
  n = atoi(argv[1]);
  setbuf(stdout, NULL);
    create(Tproduce);
    create(Tconsume);
  
}
```
经过压力测试后的输出，我们会发现没有什么明显的问题了

#### 利用条件变量，将工作并行化：线程池
在我们有了正确的条件变量实现后，就可以利用这种万能的同步方法进行各种任务的并行计算了
只要算法可以并行化，那我们都能把它利用条件变量的形式进行并行化

下面是一个典型的**线程池工作模型**：
```C
// 一个线程可以独立完成的很小的计算任务
struct job {
	void (*run)(void *arg);   // 指向执行任务实际操作的函数指针
	void *arg;  // 指向任务参数的指针
}

// 下面是每个线程的执行代码模板
while(1) {
	struct job *job;

	mutex_lock(&mutex);  // 首先获得对任务队列的独占互斥锁
	while (!(job = get_job())) {  // 检查任务队列中是否有任务
		wait(&cv, &mutex);  // 如果没有任务，等待任务，这个过程通过while持续检测
	}
	mutex_unlock(&mutex);  // 分配任务后，解除对任务队列的互斥锁

	job->run(job->arg);  // 执行取出的任务
	// 执行任务的过程无需持有锁，因为不涉及共享变量的访问
}
```

最长公共子序列算法就可以拓展到并行情况。我们在数据结构课程中学习过，最长公共子序列算法实际上是一个dp算法，它的性质在于之前计算的结果可以被用于其他状态结果的计算（**计算之间具有依赖关系**）
因此我们可以**将每次计算的状态，使用图进行表示**，这也是并行算法转换的一个基础流程

在使用图表示计算状态之后，有向边就指示了**各个结点之间的依赖关系**。因此，此时就可以对图进行**拓扑排序**，优先计算深度较小的结点，**计算深度是不可逾越的瓶颈**：因为计算需要依赖前面的结果，我们**只能在相同层数进行并行化**
对于相同深度的结点，我们就可以将计算分散到多个线程上，加快计算速度

#### 面试题示例

> [!QUESTION] 古怪的并发算法习题
> 假设现在存在三种类型的线程，它们分别打印`<`、`>`和`_`
> 我们需要对这些线程同步，使得这些线程打印出来的序列总是`<><_`和`><>_`的组合

使用条件变量解决这个问题，只需要回答三个子问题，它们都是关于在`cond_wait`外侧的`while(!condition)`检测的：
1. 在什么条件下，可以打印`<`？
2. 在什么条件下，可以打印`>`？
3. 在什么条件下，可以打印`_`?

这可以绘制**状态机图**来解决，首先，初始状态，根据题意可知只能打印`<`或`>`，形成了两个分支；随后，每个节点的下一步已经固定了（因为只能是两种组合，`<`后面只能是`>`，而`>`后面也只能是`<`），最后一步也都是只能打印`_`；并且在此之后回到初始节点
就这样，我们可以绘制出一个**环状的状态机图**，里面仅有6个节点，有了这张状态图，我们就可以定义出如下一个结构：
```c
enum { A = 1, B, C, D, E, F, };

struct rule {
  int from, ch, to;
};

struct rule rules[] = {
  { A, '<', B },
  { B, '>', C },
  { C, '<', D },
  { A, '>', E },
  { E, '<', F },
  { F, '>', D },
  { D, '_', A },
};
```
这里`rule`结构体代表了状态图中，每个节点的来源节点`from`、节点处输出的字符值`ch`和节点的汇节点`to`，并且**使用结构体数组唯一对应了图中的每个节点的转移**（因为有环【 从`_`节点到初始节点】 ，所以是七个结构体）
这些结构体的`ch`值代表当前状态下可以打印的字符

算法的**条件变量往往会被拆成多个部分**，例如这个算法中，需要检测的条件变量分为两个部分：
1. 当前是否有其他线程在进行打印？如果有其他线程打印，那么需要等它们打印完后才可以打印（类似于锁），这个条件变量由`quota`控制
2. 当前状态节点是否存在出边（后继节点）？只有存在后继节点，才能把后继节点的`ch`进行打印，这个条件变量由`int next`函数的返回值控制

首先需要有一个计算**当前状态的下一个输出可以是哪个字符**的函数`next`：
```c
int current = A, quota = 1;   // quota指示当前是否有其他线程在打印
// quota == 1代表没有其他线程在打印；quota == 0代表有其他线程在打印，暂时不能打印

pthread_mutex_t lk   = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cond = PTHREAD_COND_INITIALIZER;

int next(char ch) {
  for (int i = 0; i < LENGTH(rules); i++) {
    struct rule *rule = &rules[i];
    if (rule->from == current && rule->ch == ch) {
      return rule->to;  // 如果当前输出的字符与从结构体数组中取出来的rule结构体的前驱和值一样，那么说明下一个应该输出的就是rule的后继
    }
  }
  return 0;  // 如果遍历整个结构体数组都没找到，说明出问题了，返回0
}
``` 
如果该函数返回0，说明不存在后继节点，出错了

利用**条件变量算法**，每个线程的操作可以归纳为以下三步骤：
1. **不断检测**当前是否有其他线程在打印&&当前状态是否存在后继节点（能否打印）
2. 在无其他线程打印且可以打印时，打印字符；反之，线程进行**休眠**

3. 醒着的线程完成了它的打印工作后，修改`current`至当前状态、修改`quota`值为1,代表其他线程可以打印了，随后**唤醒其他休眠线程并解除锁**
```c
void fish_before(char ch) {  // 检测函数
  pthread_mutex_lock(&lk);  // 首先获得锁
  while (!(next(ch) && quota)) {  // 若当前状态不存在后继输出并且没有进程在打印
    // can proceed only if (next(ch) && quota)
    pthread_cond_wait(&cond, &lk); 
  }
  quota--;
  pthread_mutex_unlock(&lk);
}

void fish_after(char ch) {
  pthread_mutex_lock(&lk);
  quota++;
  current = next(ch);
  assert(current);
  pthread_cond_broadcast(&cond);
  pthread_mutex_unlock(&lk);
}

const char roles[] = ".<<<<<>>>>___";

void fish_thread(int id) {
  char role = roles[id];
  while (1) {
    fish_before(role);
    putchar(role); // can be long; no lock protection
    fish_after(role);
  }
}

int main() {
  setbuf(stdout, NULL);
  for (int i = 0; i < strlen(roles); i++)
    create(fish_thread);
}
```

## 5.3 信号量

### 5.3.1 互斥锁机制的拓展：锁->令牌->信号量
复习一下，互斥锁的工作原理是：
- 先申请锁的线程：成功获得锁，进入临界区；让锁变为“使用中”状态，系统调用直接返回
- 后申请锁的线程：不能获得锁，需要放入等待队列中进行等待，执行线程切换
- 得到锁并执行完毕的线程：归还锁，由操作系统交给排队的线程：若等待队列不空，从队列中取出一个线程允许执行；若为空，让锁变为“可用状态”
**操作系统使用自旋锁来确保自己处理锁的过程是原子的**

这里控制锁的数量为1，而实际情况中，**完全没有必要限制锁的数量，临界区可以容纳多个线程同时执行**：
- 操作系统可以持有**任意数量的锁**，锁的数量代表了临界区的容量上限
- 当所有锁都用完之后，才需要等待线程执行结束离开临界区

那么更进一步，能否**让线程自己“变出”一把锁**？也就是**动态地产生资源来控制访问临界区**
- 此时锁更类似于一种令牌：
	- 得到令牌的线程可以进入临界区执行
	- 令牌可以随时被创建

在这样的思想下，前人为其总结了一个非常漂亮的建模形式：
$$ 锁=令牌=一个资源=信号量Semaphore$$
信号量的本质是**对资源进行计数的数学概念**，它表示**当前系统中可以并发访问的资源数量**，其值也就是**锁（或令牌）的数量**，本质上是一种计数器

### 5.3.2 信号量及其PV操作
一个典型的信号量实现如下，它主要具有PV两个操作：
- P操作：也称为**等待操作**，当一个线程请求访问资源时，它会先检查信号量（计数器）的值，若值大于0，说明线程可以访问资源，为其分配一把钥匙，**计数器-1**；若信号量为0，线程必须**等待**
- V操作：也称为**释放操作**，当一个线程在临界区内完成它的操作，需要释放钥匙时，要**将信号量的值加1**，并且**通知其他线程可以访问资源**
```python
class Semaphore:
    token, waits = 1, ''  # token就是信号量计数器，wait是等待队列

    def P(self, tid):  # 等待操作
        if self.token > 0:  # 若剩下的钥匙数量大于0
            self.token -= 1   # 信号量减1
            return True   # 返回真说明成功获得钥匙
        else:    # 若信号量等于0
            self.waits = self.waits + tid  # 将自己加入等待队列中等待钥匙
            return False  # 返回假说明没有成功获得钥匙，而是在队列中等待

    def V(self):  # 释放操作
        if self.waits:  # 如果等待队列不为空
            self.waits = self.waits[1:]  # 直接从开头取出一个线程让它获得钥匙
        else:  # 等待队列为空，钥匙交不出去，交给信号量保存
            self.token += 1  # 信号量加1
 
    @thread
    def t1(self):
        self.P('1')
        while '1' in self.waits: pass  # 如果该线程在等待队列中，休眠
        cs = True
        del cs
        self.V()  # 释放，唤醒等待队列第一个线程，或信号量++

    @thread
    def t2(self):
        self.P('2')
        while '2' in self.waits: pass
        cs = True
        del cs
        self.V()

    @marker
    def mark_t1(self, state):
        if localvar(state, 't1', 'cs'): return 'blue'

    @marker
    def mark_t2(self, state):
        if localvar(state, 't2', 'cs'): return 'green'

    @marker
    def mark_both(self, state):
        if localvar(state, 't1', 'cs') and localvar(state, 't2', 'cs'):
            return 'red'
```

### 5.3.4 使用信号量实现生产者-消费者
类似于之前学习的条件变量，**信号量也可以有多种类型**
下面的代码示例中，就有两种信号量`empty`和`fill`，首先回忆一下生产者和消费者执行时的条件：
- 生产者需要在任务数量没有到达上限时（类似于口袋里还有空位时）生产任务
- 消费者需要在任务数量非0时（类似于口袋里还有东西时）取出任务并且执行
因此，我们可以发现：
- **对于生产者来说，消费者是那个变出钥匙的对象**：只有消费者取出口袋里的任务之后，生产者能够访问口袋进行生产的钥匙才会增加
- **对于消费者来说，生产者是那个变出钥匙的对象**：只有生产者往口袋中放入任务之后，消费者能够访问口袋进行任务的取出的钥匙才会增加

所以，我们设置了两种信号量，分别代表**从生产者的角度看到的钥匙`empty`** 和 **从消费者的角度看到的钥匙`fill`**
在下面的代码中我们可以发现，<u>生产者和消费者线程的PV操作与信号量的组合各不相同</u>，唯一保持一致的是**永远是先执行P操作，再执行V操作**
```c
#include "thread.h"
#include "thread-sync.h"

sem_t fill, empty;

void producer() {
  while (1) {
    P(&empty);  // 生产者角度来看，empty是它能够进入临界区的钥匙（口袋有空位）
    printf("(");  // 假设printf是线程安全的
    V(&fill);  // 生产者交还钥匙只能给消费者，而不能同类唤醒
  }
}

void consumer() {
  while (1) {
    P(&fill);  // 消费者角度来看，fill是它能够进入临界区的钥匙（口袋有任务）
    printf(")");
    V(&empty);  // 消费者交还钥匙只能给生产者，不能同类唤醒
  }
}

int main(int argc, char *argv[]) {
  assert(argc == 2);
  SEM_INIT(&fill, 0);
  SEM_INIT(&empty, atoi(argv[1]));
  for (int i = 0; i < 8; i++) {
    create(producer);
    create(consumer);
  }
}
```
在信号量的设计中，重点在于**考虑钥匙（每一单位的资源）是什么？它由谁创造，由谁获取？** 就是解决生产者消费者问题的核心
比起最原始的条件变量，这样的方法在代码密度上看简直太友好了，我们很容易会想经常使用这种方法，的确，PV原语方法在 **“一单位资源”能够明确是什么** 的问题上很好用
但是，对于[很复杂的问题](####面试题示例)，它看上去就不太好用了。而且，因为我们的**钥匙是凭空创造的**，而不是非常直观的“只有一把钥匙”，那么在解决问题的时候就**需要格外谨慎地去检查程序的正确性**，确保它不会带来类似于死锁等问题

## 5.4 哲学家吃饭问题
下面介绍一个更加复杂的经典并发问题：
> [!QUESTION] 哲♂️学家吃饭问题
> 哲♂️学家（线程）有时思考，有时吃饭，在每两个人之间放置了一把叉子：
> - 当吃饭时，需要同时得到左手和右手边的叉子
> - 当叉子被其他人占用时，必须思考（等待）
> 如何解决这样的线程之间的同步问题（使得每个人都能吃上饭）？能否使用互斥锁/信号量实现？

#### 失败的尝试：死锁
下面是一个**失败的尝试**，它使用了信号量进行实现，它为每一个哲学家都赋予一个编号`id`，以及一个信号量`locks[N]`，根据一个哲学家的`id`可以获得每个哲学家左手边或右手边的那个哲学家的`id`，每个**信号量代表每个哲学家位置上左手边的那个叉子**
```c
#include "thread.h"
#include "thread-sync.h"

#define N 3
sem_t locks[N];  // 为每一个哲学家都创建了一种信号量（哲学家左手边的叉子）

void Tphilosopher(int id) {
  int lhs = (N + id - 1) % N;   // 该哲学家左手边的哲学家的id 
  int rhs = id % N;   // 该哲学家右手边的哲学家的id
  while (1) {
    P(&locks[lhs]);  // 首先试图举起左手边的叉子
    printf("T%d Got %d\n", id, lhs + 1);   // 得到叉子
    P(&locks[rhs]);  // 然后试图举起右手边的叉子
    printf("T%d Got %d\n", id, rhs + 1);  // 得到叉子
    V(&locks[lhs]);
    V(&locks[rhs]);  // 将独占的叉子们释放掉
  }
}

int main(int argc, char *argv[]) {
  for (int i = 0; i < N; i++) {
      SEM_INIT(&locks[i], 1);
  }
  for (int i = 0; i < N; i++) {
    create(Tphilosopher);
  }
}
```
诶，看上去好像很符合直觉？毕竟P操作只是试图获取，没拿到就等待呗，那我们运行一下逝世：
```shell

normist@MIST:~/Learn/NJU-OS$ gcc -lpthread -g ph*.c && ./a.out 
T1 Got 1
T2 Got 2
T3 Got 3
哦嚯，卡住了，完全没动静了
```
居然**每个线程都进入睡眠了**，没办法继续运行下去，一点输出没有
~~年轻真好，倒头就睡~~ 这是为什么呢？

假设餐桌上的每一个哲学家都举起了他左手边的叉子，这意味着每个线程都执行到了
```c
void Tphilosopher(int id) {
  int lhs = (N + id - 1) % N;   
  int rhs = id % N;   
  while (1) {
    P(&locks[lhs]);  
    printf("T%d Got %d\n", id, lhs + 1);   
    // ------这个地方！------
```
然后，任何一个哲学家试图举起他们右手边的叉子，但是这个时候，**每个人的右手边的叉子都被叉子右手边的那个哲学家占用了**，所以**此时每个人都拿到了不完全的资源，但是也没办法申请到其他人占用的资源**，只能倒头就睡了

死锁是一个很容易检测出来的bug，毕竟这个时候没有任何线程有进展，大家都停下来了，一测就能知道

#### 成功的尝试，万能的方法（互斥锁+条件变量）
请务必牢记：**不要在操作系统课上试图以聪明的方法解决并发的问题**，因为并发问题的状态图实在是太大了，如果仅仅只是几百行的小程序，我们能对它的复杂度进行估计和把握，但是面对并发问题，体量巨大的问题无法很好地控制（讲了那么多的奇怪解法，大多数都错了）

因此，成功的方法也不会很复杂，它**采用了互斥锁+条件变量而不是信号量**
在每个线程的检测和操作中，执行以下的代码：
```c
mutex_lock(&mutex);  // 首先先获得一把互斥锁：大家先别动，让我看看！
while (!(avail[lhs] && avail[rhs])) {  // 如果左边和右边的叉子有一个不在，进入休眠
  wait(&cv, &mutex);  // 睡眠，直到被其他人唤醒
}
avail[lhs] = avail[rhs] = false;  // 此时左边和右边的叉子都在，于是我拿走了，标记为“不在”
mutex_unlock(&mutex);  // 解除互斥锁：你们随意！
// 两把叉子都稳稳地拿到了，在线程本地进行“吃饭“操作

// 下面要还叉子了
mutex_lock(&mutex);  // 获得互斥锁：先让我还！
avail[lhs] = avail[rhs] = true;  // 还叉子，置为可用状态
broadcast(&cv);  // 唤醒所有在睡眠的线程
mutex_unlock(&mutex);  // 解除互斥锁，让其他线程获得叉子
```

#### 更好的方法：Master/slaves方法（使用PV操作）
别让每个人直接管理叉子，让一个人集中管理吧！
为了确保并发的安全，我们干脆不要让每个人都单独管理它左手边的叉子了，而是**让一个线程集中地管理所有的叉子**，这种解决思路在**分布式系统**中很常见
我们建立了一个新的线程`Twaiter`，它负责**为哲学家们提供服务，分配叉子**：
- 如果想要吃饭，去**向服务员申请叉子**，如果**叉子数量够**，那么服务员会给你叉子让你吃饭；如果不够，那么就**等其他人吃完饭回收叉子**给你用
- 如果吃完了饭，那么把叉子**交给服务员进行回收**
```c
void Tphilosopher(int id) {
  send_request(id, EAT);  // 我想吃饭了，向服务员发送一个申请EAT
  P(allowed[id]); // waiter 会把叉子递给哲学家（请求叉子）
  philosopher_eat();  // 吃饭
  send_request(id, DONE);  // 吃完饭了，把叉子给服务员回收，发送消息DONE
}

void Twaiter() {
  while (1) {
    (id, status) = receive_request();
    if (status == EAT) { ... }
    if (status == DONE) { ... }
  }
}
```
这里哲学家通过`send_request(id, EAT)`和`send_request(id, DONE)`向服务员发送消息（实际上是**向共享等待队列中**发送了消息）
这种模式下，多个线程之间其实是不并行的，大家都认为自己可以成功获取，是一种**集中式的同步**
而且，**服务员对每个人吃饭的时间和叉子占用情况都知道得一清二楚**，因此可以进行一定**调度**，例如对于占用叉子时间很长的线程，我们可以故意让它晚一点得到叉子，或是给关系好的线程开一些很高的权限

或许我们会认为，管叉子的人会是性能的瓶颈，例如对于很多很多的顾客，每个人都叫服务员，那得花多长时间啊
但是不要忘记，操作系统不是算法，我们面对的永远是现实中的物理假设，追求的是实际的运行性能，**在我们不明确地知道性能瓶颈在哪里之前，不要提前进行优化**

<u>抛开 workload 谈优化就是耍流氓</u>
例如，
- 如果叫服务员的时间远远低于吃饭的时间，或者很少人在吃饭的时候，此时服务员不会成为瓶颈
- 如果的确有很多人叫服务员，导致一个manager扛不住，那我们可以**把资源分下来：让服务员之间形成一个"树“，有总服务员和分服务员，每个服务员开始时持有一定叉子，在顾客向它申请时给出去，如果实在不够用了，向上一级服务员申请叉子**(fast/slow path思想)
- 把系统设计好，使得集中管理不成为瓶颈（设计模式）
[参考资料](https://www.usenix.org/conference/nsdi20/presentation/brooker)

# Lecture 6. 真实世界的并发编程


> [!REVIEW] 复习：
> - 并发编程的基本工具是线程库、互斥和同步
> - 操作系统的暂停机制帮助我们协同多个线程更高效地完成更复杂的任务

借助前五讲的知识，我们实际上已经满足了并发编程的基本需求，本节课所需要回答的问题就是：**什么样的任务是需要并行/并发的？它们应该如何实现？**
主要内容包括：
- 高性能计算中的并发编程
- 数据中心中的并发编程
- 身边软件中的并发编程

## 6.1 高性能计算中的并发编程
世界上最早的超级计算机实际上只有一个处理器核心，但是它是支持向量化操作的：使用一条指令对多个数据值进行同样的处理（参见[CMU并行计算](./CMU并行编程)笔记）
对于高性能计算，和其名字一样，它主要集中在那些要求高计算力的任务上，例如：
- 系统模拟：天气预报自然系统模拟、能源、分子生物学
- 人工智能：神经网络训练
- 挖矿：计算hash值（在区块链中，工作量证明机制要求矿工进行复杂的数学计算求解加密问题，即反复改变输入，结合链中前一个区块的哈希值进行哈希计算，直到计算得到的某个哈希值满足网络的共识规则设定的难度目标，此时矿工就有权将这个新区块加入区块链，获得加密货币奖励）

当我们希望一个程序能够被放在超级计算机上运行，这就意味着它必须是一个能够被分解的计算任务
- 该程序的算法对应的计算图需要能被很容易地并行化：将算法的状态机画成图，一个节点指向另一个节点的边代表计算的依赖关系，按照拓扑排序分层，在每一层之内把节点分组，不同组的节点计算交给不同的CPU，通过一个`join`操作最后汇总，这是大部分超级计算机上的程序的构造
- 超级计算机（分布式环境下）的编程比起普通的多处理器编程的另一个区别在于，程序是在多台机器（计算节点）上进行计算的（可以参考[Flink学习笔记](./Apache Flink)中关于该框架的介绍），因此数据计算主要分为两层：
	1. 应该放到哪一台机器上进行计算；
	2. 每台机器上的线程及其通信是怎么进行的
		对于很多关于现实生活的模拟系统程序来说，它们的模拟基于“局部连续性”：一个人/原子/物体的行动轨迹在一定时间内一定在原位置一定距离内，因此对于基于空间位置的模拟，它将空间划分为多个块，每个线程负责一个块内的变化，只要块的数量足够，绝大部分的交互都在本地（当然，还是要考虑块与块之间的交互的），所谓的有限元分析也就类似于这种思想

具体来说，下面这个图像绘制程序允许用户输入线程数以展示绘制图形的情况：
```c
#include "thread.h"
#include <math.h>

int NT;
#define W 6400   // 图像宽度
#define H 6400   // 图像高度
#define IMG_FILE "mandelbrot.ppm" // 输出图像名

static inline int belongs(int x, int y, int t) {   // belongs判断一个像素是否属于当前线程的工作区域
  return x / (W / NT) == t;
}

int x[W][H];  // 代表整个图像矩阵
int volatile done = 0;

void display(FILE *fp, int step) {   // 接受文件指针和step参数，通过fp指针输出给显示文件，setp参数控制每次输出的步长（每个线程输出图像的不同部分）
  static int rnd = 1;
  int w = W / step, h = H / step;
  // STFW: Portable Pixel Map
  fprintf(fp, "P6\n%d %d 255\n", w, h);  // 使用ppm格式打印文件，P6\n宽 高 灰度深度，接下来则是rgb颜色矩阵
  for (int j = 0; j < H; j += step) {
    for (int i = 0; i < W; i += step) {
      int n = x[i][j];  // 根据图像矩阵x每个位置的值，计算显示时对应像素位置的RGB值
      int r = 255 * pow((n - 80) / 800.0, 3);
      int g = 255 * pow((n - 80) / 800.0, 0.7);
      int b = 255 * pow((n - 80) / 800.0, 0.5);
      fputc(r, fp); fputc(g, fp); fputc(b, fp);
    }
  }
}

void Tworker(int tid) {  // 计算Mandelbrot集合，线程id为tid
  for (int i = 0; i < W; i++)
    for (int j = 0; j < H; j++)
      if (belongs(i, j, tid - 1)) {  // 如果该像素属于该线程tid的工作区域就计算+存储
        double a = 0, b = 0, c, d;   
        while ((c = a * a) + (d = b * b) < 4 && x[i][j]++ < 880) {
          b = 2 * a * b + j * 1024.0 / H * 8e-9 - 0.645411;
          a = c - d + i * 1024.0 / W * 8e-9 + 0.356888;
        }
      }
  done++;
}

void Tdisplay() {
  float ms = 0;
  while (1) {
    FILE *fp = popen("viu -", "w"); assert(fp);  // popen将标准输出数据以管道传递给第一个参数，返回一个文件指针用于写入数据
    display(fp, W / 256);
    pclose(fp);
    if (done == NT) break;
    usleep(1000000 / 5);
    ms += 1000.0 / 5;
  }
  printf("Approximate render time: %.1lfs\n", ms / 1000);

  FILE *fp = fopen(IMG_FILE, "w"); assert(fp);
  display(fp, 2);
  fclose(fp);
}

int main(int argc, char *argv[]) {
  assert(argc == 2);
  NT = atoi(argv[1]);
  for (int i = 0; i < NT; i++) {
    create(Tworker);
  }
  create(Tdisplay);
  join();
  return 0;
}
```

对于线程数（输入参数）为1的情况：
<img src="gif/thread_1.gif" alt="thread=1" />

对于线程数为32的情况：
<img src="gif/thread_32.gif" alt="thread=32"/>
可以看到，算法中对每个线程的绘制范围做了分区，不同线程负责它所指定的部分的绘制，在并行环境下，整个图像的绘制都要比单线程环境更快

## 6.2 数据中心里的并发编程：协程
数据中心的程序和高性能计算中的程序都具有计算量大的特点，它的不同在于：
- 以数据计算以及存储为中心，用于支持各类互联网应用，例如微信、视频软件等等
- 服务器数量巨大，如果某种算法/实现能实现能耗或速度的提升，哪怕一点节约的成本都很大

在数据中心里，我们关注的是数据，例如我们不希望自己的数据会因为故障导致丢失，所以我们一般会设置多个数据中心，每个数据中心都具有用户数据的一个副本（容错机制）
而低延迟访问希望我们对数据的更改能尽量快地进行同步，实际上这是一个不可能三角：
CAP
- 数据要保持一致性（副本同步）C
- 服务要时刻保持可用（低延迟）A
- 能够容忍机器离线（容错性）P
我们想要获得任意两个功能就得舍弃另一个，这就是数据中心面对的一个很大的问题

数据中心仍然是以多台计算机组成的，对于每台计算机，它会连接存储设备以及网络，最终计算机会收到来自网络的请求，让它将输入数据进行存储或从磁盘中读取某数据输出到网路
我们可以把数据中心的一台电脑看作是一个`key - value`字典，存储这些键值对以进行数据的读写
值得注意的是，数据中心内的数据map都是保存在持久存储中的，在断电时数据不会丢失

对数据中心的性能需求就是希望能够**尽可能多地完成用户并行的对读写的大量请求**，每提升一些每秒钟处理请求的数量，机器成本就能得到很大的提升，这个工具就是**线程或协程**
关键的指标：QPS（每秒处理请求的数量）、tail latency（尾部延迟：即使大部分人的请求很快被处理了，但是还是有少部分人受到极大的延迟）...

对于数据中心，我们关注其中的单个电脑，每台电脑具有磁盘存储，同时可通过网络与其他计算机存储交互
我们现有的工具：
- 线程
```c
thread(start = true) {
	println("$Thread.currentThread() has run.")
}
```
操作系统可以将多个线程放置在不同或相同的cpu中执行，当一个cpu执行线程的切换时，也就是从前一个状态机切换到后一个状态机上，此时需要保存所有寄存器的状态，这会带来比较大的开销

### 实验M2
在本节附带的实验中，我们会实现在C代码中创建（多个）状态机，并且使用`yield()`来切换到另外一个状态机进行执行，但是使用比线程开销小很多的方式

这是通过协程实现的：协程是比线程更轻量的执行单元，它可以**在单个线程中**实现**多任务并发**。协程的**调度是由程序员或运行时系统控制的，而非操作系统**。
协程的切换`co_yield()`仍然是一个函数调用，但是它并不需要陷入内核状态执行系统调用，而是**全程保持在用户状态下的**，在执行它的时候并非所有状态都需要被保存
所以协程就只是一个轻量级的函数集合，它们的栈和状态都很小，因此切换、创建和销毁的代价非常低，相对应的代价是对于CPU密集型计算无效，因为所有协程都在同一个线程中

回到数据中心，假设我们在一个数据中心中有多个线程和协程，使用线程的好处是可以利用多个CPU并行，但是问题在于每个线程都会占用可观的操作系统资源；使用协程的好处是切换、创建和销毁非常简单，然而它们不受操作系统调度，例如若协程需要`read`磁盘中的数据的话，整个线程就会被阻塞，因为协程自己无法系统调用，此时其他协程也停下来等这个系统调用完成

可见线程和协程都不能完美解决这个问题，需要进行一定的结合

### Go和Goroutine：非阻塞式I/O
概念上是线程，实际上是线程和协程的混合体的一种模型

Go语言对于我们计算机上的多个CPU，每个CPU上绑定一个线程，称为Go worker，每个线程上可以有多个协程，称为Goroutine
当其中一个协程申请I/O时，因为使用系统调用的`read`需要阻塞整个线程及其底下的所有协程，但是linux提供了另外一种形式的非系统调用I/O的API，此时Go调用该API以完成无需阻塞的读入。
若没有读到数据，该协程会迅速`yield`至另外一个协程运行，直到其所申请的数据被加入进来之后才将原来被`yield`的协程标记为可执行状态

这样的工作模式相当于是协程向操作系统传送了一个事件信号：我马上就要干这件事了，是否能够成功？若失败，之后立刻转到另外一个协程运行它的工作；若成功就继续执行
一个Go程序中可以创建上百万个Gorotine，它的调度器会对这些协程进行管理，而真正并行运行的就只有CPU绑定的那些Goworker线程，无需进行线程间的切换，通过这种方式将操作系统和CPU利用到了极致

每个goroutine概念上都是线程，实现上都是协程，本质上是异步的

下面是一段Go程序的示例，其语法风格与C非常相似，因为就是从C来的：
```go
// Example from "The Go Programming Language"

package main

import (
  "fmt"
  "time"
)

func main() {
  go spinner(100 * time.Millisecond)  // 调用函数，括号内为参数，go在概念上就是创建了“线程”
  const n = 45
  fibN := fib(n) // slow  // 非常慢
  fmt.Printf("\rFibonacci(%d) = %d\n", n, fibN)
}

func spinner(delay time.Duration) {  // 会被调度至一个另外的CPU执行
  for {  // 死循环
    for _, r := range `-\|/` {  // 输出等待符号
      fmt.Printf("\r%c", r)
      time.Sleep(delay)
    }
  }
}

func fib(x int) int {  // 计算斐波那契
  if x < 2 { return x }
  return fib(x - 1) + fib(x - 2)
}
```
这个程序的作用就是在计算非常慢的递归斐波那契数列的同时，运行`spinner`输出等待转圈符号，直到计算完成之后输出计算结果

## 6.3 现代编程语言上的系统编程
实际应用的时候程序员往往不需要去写并发算法，因为这是非常危险的行为，在业务中如果让每个成员都去写这样的代码，会导致整个项目问题频发

其中最大的万恶之源就是共享内存，因为对其的访问如果没有遵守严格的规定，在奇怪的调度下各种并发bugs就会频发，而且条件变量和信号量机制的使用也可能不好用或不高效

Go语言就是因为这样的需求出现的，它的思想是**不要直接和共享内存交互**，既然**生产者-消费者模型可以解决大部分问题**，那么将其封装起来提供一个API就好了

~~你不准用共享内存做进程间的通信.jpg~~

生产者-消费者的API具体应该如何提供？下面是Go语言为了实现协程上的通信编写的**channel通道机制**：
```go
package main

import "fmt"

var stream = make(chan int, 10)  // make用于创建带缓冲区的通道，容量为10,类型为int（这个“类型”也可以是一个任务结构体）
const n = 4

func produce() {  // 生产者
  for i := 0; ; i++ {    // 无限循环地生产i值
    fmt.Println("produce", i)
    stream <- i  // 将i“扔进”stream这个通道，尽管i值无限生产，但在通道中的i值大于容量上限时会被阻塞
  }
}

func consume() {
  for {
    x := <- stream   // 从通道中取得任务（i值）
    fmt.Println("consume", x)
  }
}

func main() {
  for i := 0; i < n; i++ {
    go produce()
  }
  consume()
}
```
通道通过`make(chan <type>, <num>)`来创建，其中`<num>`代表该通道最多容纳的`<type>`类型资源的个数，当生产者向通道内提供的变量大于该数量时，生产者就会被阻塞

这样的代码看上去就非常易读了，比起条件变量中需要思考的`while(!cond){ wait(); }`，这种方式更加流畅：将分解成的子任务组织成为一个任务结构体塞入通道，然后让消费者进行计算，计算完成后如果更加复杂，还能将计算结果扔进另一个通道等待下一步计算
值得一提的是，GO里面还是有共享内存的，比如说通道中的对象是指针，可以将指针传递给另外一个线程，这个线程看到这个指针就可以在很低的cost的情况下从该地址把任务取过去

在数据中心中，用户对数据中心的请求往往是访问数据，此时这个请求被给到线程，申请一个系统调用，这个系统调用是需要延迟时间的，因此GO语言采用了线程-协程模型，在延迟的时候转到其他协程运行，这也是很多数据中心开发会使用到GO语言的原因

## 6.4 我们身边的并发编程
Web2.0：人与人之间联系更加紧密的互联网
`frontpage`
在早期的互联网中，所有的网站都是几乎没有交互的：用户只能阅读网站上的内容（门户网站），由一大批员工负责维护这些门户网页
而在web2.0的网站中，用户能够发送自己的博客，或是之后的微博客：更短的博客；人与人之间的更加紧密了，浏览器的页面从静态变为动态了

那么web2.0是受到什么技术支持的？
### 浏览器中的并发编程
最早的互联网中，浏览器是静态的，注意，这指的并非是“没有会移动的东西”这样含义的静态，事实上，在这样的浏览器中，通过VBScript等~~过时~~技术，能够向浏览器中嵌入脚本，控制图案在页面中的位置的改变可以实现一些土味特效等效果

在2000年以后，用户能够在浏览器中发送一个网络请求，在网页渲染完成以后，通过鼠标移动等事件发送网页请求，在发送的过程中用户所见的网页并不会发生任何改变，而是等待请求返回时将返回结果同步到网页中
这就是AJAX（Asynchronous JavaScript + XML）技术，其目的是在不重新加载整个网页的情况下，使用**异步请求**从服务器中加载数据并动态更新网页
具体来说，该技术的基本构成为：
- Asynchronous异步：浏览器发送请求到服务器并接受响应的过程，不需要浏览器持续等待，网页上的其他操作可以继续进行，页面内容在后台加载
- JavaScript：与服务器进行交互的语言，用于发起ajax请求并处理响应数据
- XML或JSON：响应数据所采用的格式，由JS解析这些数据并动态更新网页中的内容

ajax技术就是浏览器中的并发编程，与远端的服务器异步地交互起来

互联网之所以普及，与其提供给用户的良好的界面也离不开关系，在浏览器中，我们所看到的一切东西都是由HTML(DOM tree) + CSS组成的：
- HTML：超文本标记语言，定义了网页的基本结构，例如标题、段落、列表、图片等等，是Web界面的骨架。在浏览器中，HTML被解析为DOM（文档对象模型）树的形式，该树结构的每一个节点都是文档中的一个部分，例如文本、属性等等，可以由用户进行改变
- CSS：层叠样式表。用于控制网页内容的外观和布局，对HTML元素应用样式以美化和布局网页

### 人机交互程序的特点与挑战
人机交互程序中的并发相比计算中心等不算太复杂：
- 没有太多的计算：DOM Tree不可能太大，它是要给人看的，而且树的渲染绘制方式也由浏览器程序员实现了，和web程序员没有太大关系
- 没有太多的I/O：网络请求的特点是请求调用很快，事件发生后请求就被发出去了；而处理返回很慢，需要在网络中进行数据的传输和返回，io数量不算多

面对的挑战主要在于前端程序员太多了，他们并不一定具有并发编程的基础，不可能让他们直接写共享内存上的多线程代码，否则问题就很大了
因此主要的挑战在于”用户友好“的接口和模型应该如何设计，让非专业人员也能很好地处理网络请求

#### 单线程 + 事件模型
该模型完全屏蔽了多处理器模型，它认为全局只有一个线程在运行，基本运行单位是事件。线程上的每个事件一旦开始运行，就必须运行至结束，没有任何人可以打断它，而不需要上锁
对于普通的网页更新渲染，实际上事件时间是非常短的，几乎不需要考虑
而唯一耗时的是网络请求，当发出一个网络请求（调用别的网站）时，一方面是等待时间长，不可能在它返回之前什么都不干地等它；另一方面是由于网络不稳定，网络请求可能会失败
因此，不可能把所有任务放置在一个事件中，**当一个请求发出以后，这个事件实际上就应该结束了**，随后根据成功或失败的情况，执行另外一个事件

在这样的模型下，用户能够为一个事件以像下面这样的方式进行定义：
```javascript
$.ajax( { url: 'https://xxx.yyy.zzz/login',
  success: function(resp) {
    $.ajax( { url: 'https://xxx.yyy.zzz/cart',
      success: function(resp) {
        // do something
      },
      error: function(req, status, err) { ... }
    }
  },
  error: function(req, status, err) { ... }
);
```
它定义了一个事件发生后，若成功(`success`)的行为应该如何，若失败(`error`)之后的行为应该如何
其中。`$.ajax(...)`就定义了一个事件。多个事件的执行通过`$.ajax()`的嵌套完成

这里每个函数的执行是原子的，由于今天的处理器性能较高，完全可以支持这样的非并行；同时，API依然可以并行，通过浏览器内部的线程、协程等等复杂机制完成同时向多个网页发送的请求，适合大部分时间花在渲染和网络请求的场景中

坏处就在于Callback Hell，层层堆叠很容易变成代码屎山（一些小程序莫名其妙卡死可能就是这样的原因），这种问题的本质在于我们想的是流程图，但写出来的是一层一层的“回调”

为了解决这个问题，Promise语言提供了一个改进的方式
```promise
loadScript("/article/promise-chaining/one.js")
  .then( script => loadScript("/article/promise-chaining/two.js") )
  .then( script => loadScript("/article/promise-chaining/three.js") )
  .then( script => {
    // scripts are loaded, we can use functions declared there
  })
  .catch(err => { ... } );
```


```
a = new Promise( (resolve, reject) => { resolve('A') } )
b = new Promise( (resolve, reject) => { resolve('B') } )
c = new Promise( (resolve, reject) => { resolve('C') } )
Promise.all([a, b, c]).then( res => { console.log(res) } )
```

```
A = async () => await $.ajax('/hello/a')
B = async () => await $.ajax('/hello/b')
C = async () => await $.ajax('/hello/c')
hello = async () => await Promise.all([A(), B(), C()])
hello()
  .then(window.alert)
  .catch(res => { console.log('fetch failed!') } )
```

# Lecture 7. 并发bug与应对

生产代码的同时必然生产BUG，在检查BUG的时候，需要抱着“始终假设自己的代码是错的”的心态
代码是需求在信息世界上的投影，软件的BUG在于编程语言在表达现实对象时信息的有损性

一个检查BUG的方式是使用断言，在程序运行过程中报告确切的错误

## 7.1 防御性编程
增加断言的过程实际上就是将代码缺失的信息捞回来的过程，对于并发程序，如果不添加断言，大部分程序正常执行时是无法确定行为和我们预期的是否相同的，因此断言的设置对于程序的编写非常重要

断言为我们提供了一个编程的底层思路：我们的目的是**将需求投射到代码上**，断言允许我们**直接对这段代码应该实现的功能做出规约**，只要你断言写得够多，检查的粒度够细，代码的编写就变成了很清晰的事情了（因此，甚至有工具能够根据断言来自动编写过程中的代码），也就是你对每一行代码之后的程序状态情况了然于心，通过控制开头和结果（需求）就能获得一个“好的代码”
这样的防御性编程和规约给我们的启发是：我们知道很多变量的含义，只要变量的含义发生了预期之外的改变，就说明产生了奇怪的问题。这在C语言这种没有内存保护的语言中会经常使用到，尤其是在开发内核等面向底层的编程时，几乎没有任何检查和保护，所以这就需要编程者格外谨慎

## 7.2 并发bug：死锁
死锁的定义非常简单：多个线程占用一部分资源，同时也在等待获取被其他线程占用的资源后才能释放，导致多个线程持续等待

举个例子，在学习自旋锁spinlock的时候提到，不要