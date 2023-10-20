# EN
[Writing a Tiny x86 Bootloader](https://www.joe-bergeron.com/posts/Writing%20a%20Tiny%20x86%20Bootloader/)

# CN
## 编写一个微小的x86引导程序
##### 2016年12月27日

本文中的所有代码/文件都可以在我的[Github](https://github.com/Jophish/tiny-bootstrap)上找到。

编辑：在[good people on Hacker News](https://news.ycombinator.com/item?id=13268781)上的一些讨论后，已经明确，在16位实模式下，最好使用2字节寄存器bp和sp，而不是4字节的esb和esp。文章和代码已经进行了更新，以反映这一点。

___

可能是因为被困在家里，没有休息时可做的事情，或者可能是出于对低级系统设计的真正兴趣，但我已经决定自己学习有关操作系统实施的更多知识，首先从引导程序开始。因此，让我们开始吧。虽然在网上已经存在各种其他地方的所有这些信息，但通过教授来学习无疑是更好的方法，对吧？无论如何，这篇文章应该作为对引导程序到底是什么以及如何实现一个相对简单引导程序的入门指南（与像[GRUB](https://en.wikipedia.org/wiki/GNU_GRUB)这样的庞然大物相比，它显然是一个独立的小型操作系统）。

### 什么是引导程序（Bootloader）？

当计算机启动时，从无到有，使操作系统正常运行涉及多个步骤。在x86 PC上，首先发生的是BIOS的操作。我们将避免讨论BIOS工作的细节，但以下是你需要知道的内容。当你打开计算机时，处理器会立刻查看物理地址0xFFFFFFF0以获取BIOS代码，通常存储在计算机中某个只读的ROM存储器中。然后，BIOS进行POST（电源自检），并搜索可接受的启动介质。如果BIOS认为某个驱动器是可引导的，那么它会将磁盘的前512字节，也就是引导扇区（boot sector），加载到内存地址0x007C00，并通过跳转指令将程序控制传递到这个地址，交给处理器来执行。

大多数现代BIOS程序都非常强大，例如，如果BIOS识别到几个具有适当引导扇区的驱动器，它将从具有最高预分配优先级的驱动器引导。这也是为什么大多数计算机默认从USB引导而不是硬盘引导，如果在启动时插入了可引导的USB驱动器的原因。

通常，引导扇区的代码的作用是加载存储在非易失性存储器的更大的“真正”操作系统。实际上，这是一个多步骤的过程。例如，主引导记录（Master Boot Record，MBR）是一个非常常见（尽管现在越来越被废弃）的分区存储设备引导扇区标准。由于引导扇区最多只能包含512字节的数据，因此MBR引导程序通常只是将控制传递给存储在磁盘的其他位置的更大的引导程序，而后者的任务是实际加载操作系统（链式加载）。但现在，我们不会过多关注这些；在这里的目标不是编写操作系统（将其留到另一篇文章中），而只是让计算机在我们选择的屏幕上输出一些内容。

还值得注意的是，执行权在处理器处于[实模式](https://en.wikipedia.org/wiki/Real_mode)而不是[保护模式](https://en.wikipedia.org/wiki/Protected_mode)下转交给引导代码，这意味着（除其他事项外）对你所熟悉和喜欢的操作系统的所有出色功能的访问都不可用。另一方面，这意味着我们可以直接访问BIOS的[中断调用](https://en.wikipedia.org/wiki/BIOS_interrupt_call)，提供了一些有趣的低级功能。

那么，从哪里开始呢？我决定在这个项目中使用[Netwide Assembler（NASM）](https://en.wikipedia.org/wiki/Netwide_Assembler)，这是一种相当广泛使用的汇编语言。至于测试，完全可以将编译后的汇编代码使用dd命令写入USB驱动器的前512字节，然后从中引导计算机，但这样的反馈速度不太快，对吧？[Bochs](https://en.wikipedia.org/wiki/Bochs)是一个不错的x86 IBM-PC兼容模拟器，具有许多有用的功能；我们将使用它进行测试。

### 入门指南

首先，下载NASM编译器和Bochs模拟器。我使用Arch Linux，所以我的包管理器是pacman。

```shell
sudo pacman -S nasm bochs
```

仅仅是为了好玩，让我们从为我们的引导程序编写一个小型堆栈开始。x86处理器有许多[_段寄存器_](http://wiki.osdev.org/Segmentation)，用于存储64K内存段的起始位置。在实模式下，内存是使用逻辑地址而不是物理地址寻址的。内存的逻辑地址包括它所在的64K段以及相对于该段开始的偏移量。逻辑地址的64K段应该被16整除，因此，给定一个从64K段A开始，偏移量为B的逻辑地址，重构的物理地址将是A\*0x10 + B。

例如，处理器有一个用于数据段的DS寄存器。由于我们的代码位于0x7C00，数据段可以从0x7C0开始，我们可以通过以下方式设置：

```assembly
mov ax, 0x7C0
mov ds, ax
```

首先，我们必须将段加载到另一个寄存器（这里是ax）中，然后才能将其直接放入段寄存器。让我们将堆栈的存储直接放在引导程序的512字节之后。由于引导程序从0x7C00扩展到0x7E00的512字节，堆栈段SS将是0x7E0。

```assembly
mov ax, 0x7E0
mov ss, ax
```

在x86架构中，堆栈指针（SP）递减，因此我们必须将初始堆栈指针设置为距离堆栈段的字节数，等于堆栈的所需大小。由于堆栈段可以寻址64K内存，让我们设置一个8K堆栈，将SP设置为0x2000。

```assembly
mov sp, 0x2000
```

现在，我们可以使用标准的调用约定来安全地将控制权传递给不同的函数。我们可以使用push将"调用者保存"寄存器推送到堆栈上，再次使用push将参数传递给被调用者，然后使用call将当前程序计数器保存到堆栈上，并无条件跳转到给定的标签。

好了，现在所有这些都解决了，让我们找出一种方法来清除屏幕、移动指针并写入一些文本。这就是实模式和BIOS中断调用发挥作用的地方。通过存储特定寄存器和参数，然后将特定的操作码发送到BIOS作为中断，我们可以执行许多酷炫的操作。例如，通过在AH寄存器中存储0x07并将中断码设置为0x10，我们可以将窗口向下滚动多行。查看[规范](http://www.ctyme.com/intr/rb-0097.htm)。请注意，寄存器AH和AL分别指的是16位寄存器AX的最高和最低有效字节。因此，我们可以通过将16位值直接推送到AX来有效地同时更新它们的值，不过，我们将选择以更清晰的方式逐个字节更新它们的子寄存器。

如果你查看规范，你会看到我们需要将AH设置为0x07，AL设置为0x00。寄存器BH的值是[BIOS颜色属性](https://en.wikipedia.org/wiki/BIOS_color_attributes)，对于我们的目的，它将是黑色背景（0x0）和浅灰色文本（0x7），因此我们必须将BH设置为0x07。寄存器CX和DX是我们要清除的屏幕子区域。标准字符行/列的数量是25/80，所以我们将CH和CL设置为0x00，以将(0,0)设置为要清除的屏幕左上角，DH设置为0x18 = 24，DL设置为0x4f = 79。将所有这些放在一个函数中，我们得到以下代码段。
```assembly
moveto:
    push bp
    mov bp, sp
    pusha

    mov ah, 0x02     ; set cursor position
    mov bh, 0x00     ; page number
    mov dx, [bp+4]   ; load row and col from stack
    int 0x10

    popa
    mov sp, bp
    pop bp
    ret
```

这个子例程开头和结尾的开销允许我们遵守调用者和被调用者之间的标准调用约定。pusha和popa将所有通用寄存器推送到堆栈上并从堆栈中弹出。我们保存调用者的基指针（4字节），并使用新的堆栈指针更新基指针。在最后，我们基本上反映了这个过程。

不错。现在让我们编写一个子例程，将光标移动到屏幕上的任意（行，列）位置。[Int 10/AH=02h](http://www.ctyme.com/intr/rb-0087.htm)可以很好地实现这一点。这个子例程将稍有不同，因为我们需要向它传递一个参数。根据规范，我们必须将寄存器DX设置为一个两字节的值，第一个表示所需的行，第二个表示所需的列。AH必须是0x02，BH表示我们要将光标移动到的页码。这个参数与BIOS允许你在屏幕之外的页面上绘制有关，以便在向用户显示之前渲染屏幕之外的内容以实现更平滑的视觉过渡。这被称为多重缓冲或双缓冲。然而，我们不太关心这一点，所以我们将使用默认的页0。

把这些都放在一起，我们有以下子例程。
```assembly
movecursor:
    push bp
    mov bp, sp
    pusha

    mov dx, [bp+4]      ; get the argument from the stack. |bp| = 2, |arg| = 2
    mov ah, 0x02        ; set cursor position
    mov bh, 0x00        ; page 0 - doesn't matter, we're not using double-buffering
    int 0x10

    popa
    mov sp, bp
    pop bp
    ret

print:
    push bp
    mov bp, sp
    pusha
    mov si, [bp+4]      ; grab the pointer to the data
    mov bh, 0x00        ; page number, 0 again
    mov bl, 0x00        ; foreground color, irrelevant - in text mode
    mov ah, 0x0E        ; print character to TTY
.char:
    mov al, [si]        ; get the current char from our pointer position
    add si, 1       ; keep incrementing si until we see a null char
    or al, 0
    je .return          ; end if the string is done
    int 0x10            ; print the character if we're not done
    jmp .char       ; keep looping
.return:
     popa
     mov sp, bp
     pop bp
     ret
```

在这段代码中，可能看起来不寻常的是 **mov dx, \[bp+4\]**。这将参数从栈中移到DX寄存器。之所以要偏移4，是因为bp的内容占用栈上的2个字节，参数也占用2个字节，因此我们必须从bp的实际地址偏移4个字节。此外，调用者有责任在被调用者返回后清理栈，这意味着通过移动栈指针来删除栈顶的参数。

我们要编写的最后一个子例程只是一个简单的子例程，它接收一个指向字符串开头的指针，并从当前光标位置开始在屏幕上打印该字符串。使用视频中断代码和[AH=0Eh](http://www.ctyme.com/intr/rb-0106.htm)非常方便。首先，我们可以定义一些数据并使用类似下面的方式存储指向其起始地址的指针。

```assembly
msg:    db "Oh boy do I sure love assembly!", 0
```

末尾的0用于以空字符终止字符串，这样我们就知道字符串何时结束。我们可以使用_msg_引用该字符串的地址。然后，剩下的部分基本上与我们刚刚在movecursor中看到的类似。我们使用了更多的标签和条件跳转，但为了避免过于详细，代码的理解留给读者自己去练习；)。

这差不多就是了，朋友们。将我们到目前为止拥有的一切连接在一起，我们就得到了以下实际的引导程序。
```
bits 16

mov ax, 0x07C0
mov ds, ax
mov ax, 0x07E0      ; 07E0h = (07C00h+200h)/10h, beginning of stack segment.
mov ss, ax
mov sp, 0x2000      ; 8k of stack space.

call clearscreen

push 0x0000
call movecursor
add sp, 2

push msg
call print
add sp, 2

cli
hlt

clearscreen:
    push bp
    mov bp, sp
    pusha

    mov ah, 0x07        ; tells BIOS to scroll down window
    mov al, 0x00        ; clear entire window
    mov bh, 0x07        ; white on black
    mov cx, 0x00        ; specifies top left of screen as (0,0)
    mov dh, 0x18        ; 18h = 24 rows of chars
    mov dl, 0x4f        ; 4fh = 79 cols of chars
    int 0x10        ; calls video interrupt

    popa
    mov sp, bp
    pop bp
    ret

movecursor:
    push bp
    mov bp, sp
    pusha

    mov dx, [bp+4]      ; get the argument from the stack. |bp| = 2, |arg| = 2
    mov ah, 0x02        ; set cursor position
    mov bh, 0x00        ; page 0 - doesn't matter, we're not using double-buffering
    int 0x10

    popa
    mov sp, bp
    pop bp
    ret

print:
    push bp
    mov bp, sp
    pusha
    mov si, [bp+4]      ; grab the pointer to the data
    mov bh, 0x00        ; page number, 0 again
    mov bl, 0x00        ; foreground color, irrelevant - in text mode
    mov ah, 0x0E        ; print character to TTY
.char:
    mov al, [si]        ; get the current char from our pointer position
    add si, 1       ; keep incrementing si until we see a null char
    or al, 0
    je .return          ; end if the string is done
    int 0x10            ; print the character if we're not done
    jmp .char       ; keep looping
.return:
    popa
    mov sp, bp
    pop bp
    ret


msg:    db "Oh boy do I sure love assembly!", 0

times 510-(\$-$$) db 0
dw 0xAA55

```
这里可能有一些不太熟悉的东西。程序的第一行告诉汇编器我们正在使用16位实模式。在我们完成打印后的_cli_和_hlt_两行告诉处理器不接受中断并停止处理。最后，请记住，引导扇区中的代码必须完全是512字节，并以0xAA55结尾。最后两行将二进制文件填充到510字节的长度，并确保文件以适当的引导标志结束。

就是这些了。

哦，你实际上是否想要_运行_这段代码？请将上面的代码保存到一个文件中，比如_boot.asm_。

然后，使用以下命令从我们的汇编引导程序代码生成一个漂亮的二进制文件。
然后，在相同的目录中，创建一个名为_bochsrc.txt_的文件，并将其填充为以下内容：

```plaintext
megs: 32
romimage: file=/usr/share/bochs/BIOS-bochs-latest, address=0xfffe0000
vgaromimage: file=/usr/share/bochs/VGABIOS-lgpl-latest
floppya: 1_44=boot.com, status=inserted
boot: a
log: bochsout.txt
mouse: enabled=0
display_library: x, options="gui_debug"
```

这只包含了一些Bochs的简单配置信息，没有太复杂的内容。基本上，你只是告诉Bochs你的启动介质是一个1.44兆软盘，其中包含了你的二进制文件。最后，你只需要运行以下命令：

```shell
bochs -f bochsrc.txt
```

使用你刚刚编写的配置文件来运行Bochs，_voila_，你应该会看到类似以下的东西。

![Bochs Screenshot](https://www.joe-bergeron.com/images/bochs.png)

哇，相当无聊，不是吗？如果你周围有一个USB驱动器，你可以做一些稍微酷一点的事情。插入USB驱动器并找出它的位置（使用dmesg或其他工具）。我的USB驱动器位于_/dev/sdb_。使用_dd_，运行以下命令：

```shell
sudo dd if=boot.com of=/dev/sdb bs=512 count=1
```
这将复制你的引导程序的前512字节（也就是全部），到USB驱动器的第一个512字节。如果你想确保一切都复制得很好，你可以将_if=/dev/sdb_和_of=test.com_，然后比较这两个文件。它们应该是相同的。接下来，只需要重新启动计算机（可能需要更改引导优先级以首先从USB引导），你应该会看到与几分钟前在模拟器中看到的相同无聊的文本。干得好。

再次强调，大多数_真正的_引导程序比这个复杂得多，但我认为这是一个相当好的概念验证/学习工具。_希望_你从中学到了一些东西，我肯定学到了一些，即使最终的结果远远不如我期望的那么令人激动。
