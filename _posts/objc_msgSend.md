> 本文开始仅仅是对@hursing的[xcode反汇编调试iOS模拟器程序（五）调试objc_msgSend函数](http://blog.csdn.net/hursing/article/details/8755491)的x86-64再现。随着深入学习，也理解了objc_msgSend函数为什么必须使用汇编实现。

###**环境**

- **Xcode 8.2.1**
- **[objc4-706源码](https://opensource.apple.com/tarballs/objc4/)**

-------------------

###**步骤**

 新建一个iOS工程，viewDidLoad中添加如下代码:
```
    UILabel *label = [[UILabel alloc] init];
    [label setText:@"xxxxcontent"];
```
main函数加断点：

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTQwNzM0NjEx?x-oss-process=image/format,png)


运行工程，**注意要在模拟器环境下调试**，跑到断点处，添加以下lldb命令，给*-[UILabel setText:]*方法加个断点

```
(lldb) b -[UILabel setText:]
```
继续运行,会停在如下的反汇编处

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTQyMDA2Njk2?x-oss-process=image/format,png)

这里就是 **-[UILabel setText:]** 的汇编实现，对runtime有了解的都知道，这个方法最终会转换成这样的一个函数，

```
objc_msgSend(id self, SEL op,...) 
/*-[UILabel setText:]会相应的传递三个参数:
   self对应label实例
   op对应@selector(setText:) 
   param对应字符串@"xxxContent"
*/
```
下面利用lldb验证下，单步调试Step over,直到objc_msgSend的入口

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTQ0MTQ3NzM2?x-oss-process=image/format,png)

cpu运行到这里，已经为objc_msgSend函数调用传递好了参数，根据[x86-64架构的ABI约定][1]
> %rdi，%rsi，%rdx，%rcx，%r8，%r9 用作函数参数，依次对应第1参数，第2参数.....

可知，此时self,op,param会放到%rdi,%rsi,%rdx三个寄存器来传递，下面验证，lldb输入以下命令，打印寄存器内容

`(lldb) register read`

结果

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTQ1MTM5Mjcx?x-oss-process=image/format,png)

看图可知，rsi寄存器确实放的是方法名：__setText:，多一个下划线是因为符合C语言调用规范的函数在转化为符号时都以下划线为前缀,rdx确实放的是字符串*@"xxxContent"*，rdi显示的是一个16进制数字，没有提示额外信息，其实这个就是label实例的地址，可以lldb以下命令验证，

`(lldb) po $rdi`

结果

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTUwMTMzNDk1?x-oss-process=image/format,png)

接着，进入到objc_msgSend函数里面，点击调试器工具栏**Setp into**,注意是**Step inot**,结果

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAzMTYwNDAyNDgx?x-oss-process=image/format,png)

这里就是有名的objc_msgSend函数了，下面翻译这段汇编

-  假设对象为nil,

```
0x10066dac0 <+0>:   testq  %rdi, %rdi             ; 根据前文，rdi就是self,test指令对self逻辑与，结果从新保存在rdi，如果是0，Z标志位置1，相当于高级语言 if(self == nil)
0x10066dac3 <+3>:   jle    0x10066db10            ;jle指令根据前面test指令结果做判断，执行响应跳转，这里就是高级语言的if……else……，这里假设self为nil,下一步会调转到0x10066db10处，
0x10066db10 <+80>:  je     0x10066db47            ; <+135> je同样成立，继续跳转
0x10066db47 <+135>: xorl   %eax, %eax             ;xor指令清0，rax存放返回值,eax代表rax的低32位，
0x10066db49 <+137>: xorl   %edx, %edx             ;~~
0x10066db4b <+139>: xorps  %xmm0, %xmm0           ;xmm寄存器存放浮点数，对它的了解还不很深
0x10066db4e <+142>: xorps  %xmm1, %xmm1           ;~~
0x10066db51 <+145>: retq                          ;相当于高级语言return，返回调用函数 的地方

// 至此对象为nil的情况执行完毕，所以oc里面向一个空对象发消息，是允许的，只是什么都没做
```
  - 假设对象不为空

```
0x10066dac0 <+0>:   testq  %rdi, %rdi             ;判断self是否为空
0x10066dac3 <+3>:   jle    0x10066db10            ;self不为空，不跳转
0x10066dac5 <+5>:   movq   (%rdi), %r10           ;这里很重要，这条指令的意思是将内存中地
址为rdi的8字节数据，移动到r10寄存器，根据runtime源码objc-private.h中objc_object结构体的定义知道，对象在内存中的第一个数据一定是isa，内存布局参见最下面附图一，所以这里r10就是label实例的类信息isa,你可以单步Step over这条命令之后打印r10指向的对象，(lldb) po $r10,验证是否是UILalel，这一步就是常说的根据对象的isa指针遍历类信息
0x108919ac8 <+8>:   movq   %rsi, %r11             ;将要查找的SEL(_setText：)送至r11寄存器，很多人都见过SEL,但是它具体是什么，源码没有告知，其实他就是char *指针，你可以在执行完这条指令后 打印字符内容：(lldb) p (char *)$r11，把它强转成字符指针，结果就会显示"_setText:",那SEL跟char *差别在哪，差别就在SEL存储的这个地址经过HASA处理，用来快速定位方法，接下来的指令就会看到这个地址怎么用.
0x108919acb <+11>:  andl   0x18(%r10), %r11d      ;这条指令是将内存中地位为（r10+0x18）的4字节与r11寄存器的低32位按位与，高32位置0，结果保存在r11寄存中。r10就是isa，指向objc_class结构体，objc_class的内存布局参见附图二，0x18就是十进制24。联合起来，就是把字符串"_setText:"的地址和cache的_mask做了一个逻辑与处理，这条指令之后我这边r11结果为2，读取r11内容(lldb) register read $r11 ，结果r11 = 0x2
0x108919acf <+15>:  shlq   $0x4, %r11             ;r11内容左移4位，r11 = 0x20,这个0x20就是SEL哈希后对应的index,可以凭index直接定位到cache哈希表里SEL的位置,SEL为什么要这么跟_mask与再左移4位，我也不知道，应该是跟使用的HASA算法有关吧，知道的可以告诉我。
0x108919ad3 <+19>:  addq   0x10(%r10), %r11       ;r10指向objc_class,根据附图二，objc_class偏移16字节位置是cache,0x10(%r10)里面就是就是struct bucket_t *，即缓存表的入口，加上index后，就直接定位到了SEL的位置，参见图三，r11寄存器此时就指向了偏移为0x20处的_key,
0x108919ad7 <+23>:  cmpq   (%r11), %rsi           ;比较待查找的SEL和_key，这就是我们经常听说的先从缓存查找SEL
0x108919ada <+26>:  jne    0x108919ae0            ;没有找到的话跳转，找到的话执行IMP,也就是下一条指令
0x108919adc <+28>:  jmpq   *0x8(%r11)             ;根据bucket_t的结构，可知r11再偏移8字节就是IMP,jmp进去执行，至此，函数任务达成，完毕。
0x108919ae0 <+32>:  cmpq   $0x1, (%r11)           ;缓存里没有找到SEL,会跳到这里，看看r11指向的_key是否小于1，其实就是判断缓存表是否遍历完了
0x108919ae4 <+36>:  jbe    0x108919af3            ;_key小于等于1跳转，否则继续
0x108919ae6 <+38>:  addq   $0x10, %r11            ;r11加0x10，指向缓存表的下一个_key,
0x108919aea <+42>:  cmpq   (%r11), %rsi           ;再次比较待查找的SEL是否就是_key,
0x108919aed <+45>:  jne    0x108919ae0            ;不是的话跳回到0x108919ae0,12行--16行就是个for循环遍历cache，直到找到SEL,或者遍历完毕，如果你单步Step over,会看到他一直在这循环好久
0x108919af3 <+51>:  jb     0x108919b52            ;经过我的调试，本例最终会跳到这里来，也就是cache遍历完毕都没有找到SEL，你可以在这行设置断点，然后continue这里，而不用陷在循环里反反复复
0x108919b52 <+146>: jmp    0x10891a510            ; _objc_msgSend_uncached 很明显这就是缓存没有命中后的处理，Step into，进里面，会看到附图四，里面有个_class_lookupMethodAndLoadCache3函数，这个函数在runtime源码里面是有的，而且注释很清楚，里面就是经常听说的runtime消息转发机制，等待这个函数执行完后，会把IMP返回到%rax中
~~~~~~~~~~~~~
0x10891a58c <+124>: jmpq   *%r11                  ;在这个位置设置断点，忽略lookupMethodAndLoadCache3的执行过程，等cpu到这里，Step into进去就是_setText:的IMP了，见附图五

//至此，self不为nil的情况结束了，找到IMP,并实现了

```


![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTYxMDQyMzcw?x-oss-process=image/format,png)

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTcxNTQzMDEx?x-oss-process=image/format,png)

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTczNDU0NDAx?x-oss-process=image/format,png)

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTc1NzM3Mjg0?x-oss-process=image/format,png)

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMzAyMTgwNDQxOTQy?x-oss-process=image/format,png)

---------
###**参考**
1. https://objccn.io/issue-19-2/ lldb入门经典
2. http://wiki.ubuntu.org.cn/%E7%94%A8GDB%E8%B0%83%E8%AF%95%E7%A8%8B%E5%BA%8F  gdb调试
3. http://lldb.llvm.org/lldb-gdb.html gdb与lldb对比
4. 《汇编语言》王爽著   ；汇编入门

---------------------------

[1]: http://www.udpwork.com/item/9450.html