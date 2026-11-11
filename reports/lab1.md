# Chapter 3 report

- [Chapter 3 report](#chapter-3-report)
  - [Chapter 3 功能实现](#chapter-3-功能实现)
    - [实现的方法和思路A](#实现的方法和思路a)
      - [方法A出现的问题](#方法a出现的问题)
      - [问题修复](#问题修复)
    - [实现的方法和思路B](#实现的方法和思路b)
  - [Chapter3 简答题](#chapter3-简答题)

## Chapter 3 功能实现
> 说明：在chapter3 中实现新的系统调用 ``sys_trace``（ID 为 410）用来追踪当前任务系统调用的历史信息,<br>定义如下：
>```fn sys_trace(_trace_request: usize, _id: usize, _data: usize) -> isize ```
### 实现的方法和思路A

1. 第一种为 修改 TCB（TaskControlBlock）中的参数，增加一个 syscall_id_counts <br>的变量来记录每一个任务中的系统调用的次数，其类型为 `[usize;MAX_SYSCALL_ID(512)]`，<br>,之后只需要在 `TASK_MANAGER` 中 利用其中的`current_id`来找到当前任务的id,来<br>实现相关的`record_syscall`函数，同理 查询次数的功能与 `record_syscall` 函数类似。
2. 在我们实现了功能函数之后，需要在合适的区域进行记录，所以由于本次的syscall也会被记录<br>其中,所以我们需要修改 trap/mod.rs文件，来修改 syscall 分支，在真正地调用SYSCALL<br>之前，来对syscall count进行记录
#### 方法A出现的问题
**问题描述：**
> 我的`sys_trace` 函数的功能可以正常地通过，但是对于`sys_get_time`却出现了问题，但是<br>我并没有修改相关的函数逻辑，但是我在时钟初始化之后立即调用`sys_get_time`函数也没有<br>问题，好像只有`ch3_sleep.rs` 出现了问题。

我并不知道问题出现在哪里？在使用 Ai 分析的情况下，我明白了问题的所在，**内核的启动栈溢出了**,<br>这是我第一次遇到这种bug。<br>
**启动栈溢出bug分析:**<br>
**1.bug出现前的内存布局：**
```
高地址
0x802ad000  ebss / ekernel
              .bss 的其它内容（堆 HEAP_SPACE 128KB、TASK_MANAGER 的静态存储 ~66KB……）
0x8027c000  boot_stack_top  ← 启动时 sp 从这里开始，向下生长
              │
              │  启动栈，总共 64KB
              │  （entry.asm: .space 4096 * 16）
              ▼
0x8026c000  boot_stack_lower_bound  ← 栈底"地板"
            ─────────────────────────────────────────
0x8026bbf8  .data 末尾：LLVM 常量池
0x8026b9d8    .LCPI14_0 = 0xabcc77118461cefd   ← 除法要用的魔数
0x8026b9d0  app_12_end（最后一个内嵌用户程序结束）
              …
0x80237000  .data 开头：_num_app、13 个 app 的 .bin
0x80236000  .rodata 结束（KERNEL_STACK / USER_STACK 被 LLVM 放进了只读段）
0x80205000  .rodata 开头
0x80200000  .text 内核入口
低地址
```
**2.栈的使用:**<br>
`rust_main()` 运行在启动栈上，每调用一层函数，sp就需要向下增长.<br>
**3.bug出现:**<br>
当TASK_MANAGER在第一次访问时会出现资源的初始化，其第一次访问在`run_first_task()`,
<br>我在`TCB`中增加的参数占据内存为 512*8B = 4KB，并且对于管理器而言有16个任务,所以<br>共占据64KB,而在entry.asm中我们的启动栈为64KB大小，并且上述资源的初始化在运行中进行，<br>其出现在函数中，被认为时函数中的局部变量，编译器首先将它放在栈上构造，于是<br>出现了**栈溢出**并且我们的零初始化将部分的`.data section`给覆盖了，故出现了问题<br>

**4.时间线总结:**<br>
```
rust_main 启动 …… 此时 .data 里魔数还在，get_time_us() 正常
      ↓
trap::init / load_apps / set_next_trigger   （都没问题）
      ↓
task::run_first_task()
      ↓
第一次解引用 TASK_MANAGER → lazy_static 初始化 → 构造 66KB 数组
      ↓
栈溢出，魔数被清零        ← 分界点在这里！
      ↓
切到用户程序，ch3_sleep 第一次调 get_time() = 0
```
#### 问题修复
使用了`[u32;MAX_SYSCALL_ID]`来记录，使得占用大小减少一半（32KB）

### 实现的方法和思路B
1. 使用 `static mut SYSCALL_COUNT_IDS` 来记录，类型为`[usize;MAX_APP_NUMS*MAX_SYSCALL_ID]`,<br>大小同样为64KB，但是这样的写法会将变量归类为静态变量,在编译/链接时就已经分配好地址，数据位于.bss段，内核启动栈中不会出现64KB,<br>
2. 其他的功能实现与方法A类似

## Chapter3 简答题
>问题一： 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。请同学们可以自行测试这些内容（运行 三个 bad 测例 (ch2b_bad_*.rs) ）， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。<br>

**解答：**
```
Rustsbi 版本为0.4.1 ，RISCV SBI 版本为 2.0

ch2b_bad_address.rs 出错行为：
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.

ch2b_bad_instructions.rs 出错行为：
[kernel] IllegalInstruction in application, kernel killed it.

ch2b_bad_register.rs 出错行为：
[kernel] IllegalInstruction in application, kernel killed it.
```

> 问题二：深入理解 trap.S 中两个函数 __alltraps 和 __restore 的作用，并回答如下问题:


>L40：刚进入 __restore 时，sp 代表了什么值。请指出 __restore 的两种使用情景。

**解答：**
```text
sp 代表了 当前要恢复应用的内核栈，其中的内核栈 栈帧指向了构造的 TrapContext.

__restore的使用场景：
1.构造初始的TaskContext时作为 寄存器ra的返回值，来使用应用能够第一次进入U态
2.已经换出的应用的TrapContext的切换

```

>L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。<br>

```
ld t0, 32*8(sp)
ld t1, 33*8(sp)
ld t2, 2*8(sp)
csrw sstatus, t0
csrw sepc, t1
csrw sscratch, t2
```
**解答：**
```
特殊处理了 sstatus sepc sscratch 寄存器，
其中 sstatus 记录了trap前的CPU运行级别，方便之后的CPU运行级别的恢复
sepc 记录了trap返回之后应该执行的指令地址
sscratch 作为中转寄存器来保存和恢复寄存器
```

>L50-L56：为何跳过了 x2 和 x4？

```
ld x1, 1*8(sp)
ld x3, 3*8(sp)
.set n, 5
.rept 27
   LOAD_GP %n
   .set n, n+1
.endr
```

**解答：**
```
x2为 sp寄存器，因为经过 csrrw sp, sscratch,sp 指令之后，当前我们正在使用的sp寄存器为内核栈的指针，并不是我们要保存的用户栈的指针，只有在将其他的通用寄存器保存之后，才能保存 sepc sscratch sstatus等特殊寄存器

x4 为 thread pointer ,应用使用不到
```

>L60：该指令之后，sp 和 sscratch 中的值分别有什么意义？

**解答：**
```
sp 指向了应用的应用态栈 ，sscratch指向了 应用的内核栈
```

>__restore：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？

**解答：**
```
sret 指令发生状态切换
该指令会使用CPU按照当前的特权级别将 sstatus 的 SPP字段设置为 U或者S ，之后会跳转到spec 寄存器指向的指令，继续执行
```

>L13：该指令之后，sp 和 sscratch 中的值分别有什么意义？

**解答：**
```
sp 指向了应用的内核栈 ,sscratch 记录了应用执行过程中的应用栈
```

>从 U 态进入 S 态是哪一条指令发生的？

**解答：**
```
ecall 指令
```