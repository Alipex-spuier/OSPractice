## EN
https://www.joe-bergeron.com/posts/Writing%20a%20Tiny%20x86%20Bootloader/

## CN
撰写一个微型x86引导加载程序
2016年12月27日

本文所有代码/文件都可以在我的GitHub上找到。

编辑：在Hacker News上的一些讨论后，明显地，当处于16位实模式时，最好使用2字节的寄存器bp和sp，而不是4字节的esb和esp。文章和代码已经进行了更新以反映这一点。

也许是因为在休息期间被困在家里无事可做，也许是因为对底层系统设计产生了真正的兴趣，但我决定自己学习更多关于操作系统实现的知识，从引导加载程序开始。所以，我们开始吧。虽然网络上存在关于引导加载程序的各种信息，但没有比通过教授来学习更好的方式，对吧？不管怎么说，这篇文章应该作为关于引导加载程序的作用以及如何实现一个相对简单的引导加载程序（与类似GRUB这样庞大的引导加载程序相比）的入门指南。

什么是引导加载程序？
当计算机启动时，从无到有启动操作系统的工作涉及多个步骤。在x86 PC上发生的第一件事情就是BIOS的运行。我们将避免讨论BIOS的工作细节，但这里有些你需要知道的东西。当你打开电脑时，处理器会立即查看物理地址0xFFFFFFF0上的BIOS代码，通常位于计算机的某个只读ROM上。然后，BIOS进行自检，并搜索可接受的启动介质。如果BIOS认为某个驱动器是可引导的，那么它会将驱动器的前512字节加载到内存地址0x007C00，并通过一个跳转指令将程序控制权转移到该地址，交给处理器。

大多数现代BIOS程序都相当强大，例如，如果BIOS识别了几个具有适当引导扇区的驱动器，它将从具有最高预分配优先级的驱动器引导；这正是为什么大多数计算机默认从USB启动而不是硬盘，如果在启动时插入了可引导的USB驱动器。

通常，引导扇区的代码的作用是从非易失性存储器的其他地方加载一个更大的“真正的”操作系统。实际上，这是一个多步过程。例如，主引导记录（MBR）是存储设备的常见（尽管现在越来越不建议使用）引导扇区标准。由于引导扇区最多包含512字节的数据，因此MBR引导加载程序通常只是将控制权传递到存储在磁盘的其他地方的不同、更大的引导加载程序，其任务是实际加载操作系统（链式加载）。然而，现在我们不会过多关注所有这些；这里的目标不是编写操作系统（这个要在另一篇文章中介绍），而是让计算机在我们选择的屏幕上显示一些内容。

还有一点需要注意的是，处理器在实模式下而不是保护模式下将执行权传递给引导代码，这意味着（除其他外）无法访问你熟悉和喜爱的操作系统的所有伟大功能。另一方面，这意味着我们可以直接访问BIOS中断调用，提供一些很酷的低级功能。

那么，从哪里开始呢？我决定使用NASM，这是一种非常常见的汇编语言，用于这个项目。就测试而言，可以直接将编译后的汇编代码复制到USB驱动器的前512字节，然后从中启动计算机，但这没有太快的反馈，对吧？Bochs是一个非常好的x86 IBM-PC兼容仿真器，它具有许多有用的功能；我们将用它进行测试。

入门
请下载NASM编译器和Bochs。我使用Arch，所以pacman是我的软件包管理器。

```shell
sudo pacman -S nasm bochs
```

只是为了好玩，让我们从为引导加载程序创建一个小型堆栈开始。x86处理器有许多段寄存器，用于存储64k内存段的开始位置。在实模式下，内存使用逻辑地址而不是物理地址进行寻址。内存段的逻辑地址包括它所在的64k段，以及它相对于该段开始的偏移量。逻辑地址的64k段应该除以16，因此，对于从64k段A开始的逻辑地址，偏移量为B，重构的物理地址将是A*0x10 + B。

例如，处理器有一个用于数据段的DS寄存器。由于我们的代码位于0x7C00处，数据段可以从0x7C0处开始，我们可以使用以下代码来设置它：

```assembly
mov ax, 0x7C0
mov ds, ax
```

我们必须先将段加载到另一个寄存器（这里是ax），然后才能将其直接放入段寄存器。让我们将堆栈的存储从引导加载程序的前512字节直接开始。由于引导加载程序从0x7C00扩展到0x7E00，堆栈段（SS）将是0x7E0。

```assembly
mov ax, 0x7E0
mov ss, ax
```

在x86架构中，堆栈指针递减，因此我们必须将初始堆栈指针设置为堆栈段之后的字节数，等于堆栈的所需大小。由于堆栈段可以寻址64k内存，让我们创建一个8k的堆栈，将SP设置为0x2000。

```assembly
mov sp, 0x2000
```

现在，我们可以使用标准的调用约定安全地将控制权传递给不同的函数。我们可以使用push将调用者保存的寄存器推送到堆栈上，使用push再次将参数传递给被调用者，然后使用call将当前程序计数器保存到堆栈中，并执行跳转到给定的标签。

好了，既然上述内容都已经了解了，让我们找出一种清除屏幕、移动光标和写文本的方法。这是实模式和BIOS中断调用的地方。通过将特定寄存器与特定参数一起存储，然后将特定操作码发送到BIOS作为中断，我们可以做很多有趣的事情。例如，通过在AH寄存器中存储0x07并向BIOS发送中断码0x10，我们可以将窗口向下滚动若干行。查看规范。请注意，寄存器AH和AL指的是16位寄存器AX的最高和最低有效字节。因此，我们可以通过简单地将一个16位值推送到AX来同时更新它们的值，但我们将选择更新每个1字节子寄存器以保持更清晰的方法。

如果查看规范，您会看到我们需要将AH设置为0x07，AL设置为0x00，寄存器BH引用BIOS颜色属性，对于我们的目的将是黑色背景（0x0）和浅灰色（0x7）文本，因此我们必须将BH设置为0x07。寄存器CX和DX引用我们要清除的屏幕的子部分。标准的字符行/列数是25/80，因此我们将CH和CL设置为0x00，以设置(0,0)为要清除的屏幕左上角，DH设置为0x18 = 24，DL设置为0x4F = 79。将所有这些放在一个函数中，我们得到以下代码片段。

```assembly
clearscreen:
    push bp
    mov bp, sp
    pusha

    mov ah, 0x07        ; 告诉BIOS向下滚动窗口
    mov al, 0x00        ; 清除整个窗口
    mov bh, 0x07        ; 白色文本，黑色背景
    mov cx, 0x00        ; 指定屏幕左上角为(0,0)
    mov dh, 0x18        ; 18h = 24行字符
    mov dl, 0x4f        ; 4fh = 79列字符
    int 0x10            ; 调用视频中断

    popa
    mov sp, bp
    pop bp
    ret
```

子例程开头和末尾的开销允许我们在调用者和被调用者之间遵循标准的调用约定。pusha和popa将所有通用寄存器推送到堆栈上，我们保存了调用者的基指针（4字节），并用新的堆栈指针更新基指针。在最后，我们基本上是在做相反的操作。

好了，现在让我们编写一个子例程，它可以将光标移动到屏幕上的任意（行，列）位置。Int 10/AH=02h可以很好地完成这项任务。这个子例程将稍微不同，因为我们需要向它传递一个参数。根据规范，我们必须将寄存器DX设置为一个两字节值，第一个表示所需的行，第二个表示所需的列。AH必须为0x02，BH表示我们要将光标移动到的页号。这个参数与BIOS允许您绘制到屏幕之外的页面有关，以便在将内容显示给用户之前通过呈现屏幕之外的内容来实现更流畅的视觉过渡。这称为多重缓冲或双缓冲。然而，我们并不真的关心这个，因此我们将使用默认的页面0。

将所有这些放在一起，我们得到以下子例程。

```assembly
movecursor:
    push bp
    mov bp, sp
    pusha

    mov dx, [bp+4]      ; 从堆栈中获取参数。|bp| = 2，|arg| = 2
    mov ah, 0x02        ; 设置光标位置
    mov bh, 0x00        ; 页0 - 不重要，我们不使用双缓冲
    int 0x10

    popa
    mov sp, bp
    pop bp
    ret
```

唯一可能看起来不寻常的是mov dx, [bp+4]。这将参数从堆栈传递到DX寄存器。之所以要偏移4是因为bp的内容在堆栈上占用2个字节，而参数占用2个字节，所以我们必须从bp的实际地址偏移4个字节。还请注意，调用者有责任在被调用者返回后清理堆栈，这相当于通过将堆栈指针移动到顶部的参数来删除堆栈中的参数。

我们要编写的最后一个子例程只是给定一个指向字符串开头的指针，然后将该字符串从当前光标位置开始打印到屏幕上。使用AH=0Eh的视频中断代码很好地完成了这项任务。首先，我们可以定义一些数据并使用以下方式存储指向其起始地址的指针。

```assembly
msg:    db "Oh boy do I sure love assembly!", 0
```

末尾的0将字符串以空字符终止，这样我们就知道字符串何时结束。我们可以通过msg引用该字符串的地址。然后，其余的部分基本上与我们刚刚在movecursor中看到的类似。我们使用一些标签和条件跳转，但为了不啰嗦，理解代码留给读者作为练习。

```assembly
print:
    push bp
    mov bp, sp
    pusha
    mov si, [bp+4]      ; 获取数据的指针
    mov bh, 0x00        ; 页号，仍然是0
    mov bl, 0x00        ; 前景颜色，无关紧要 - 在文本模式中
    mov ah, 0x0E        ; 打印字符到TTY
.char:
    mov al, [si]        ; 从指针位置获取当前字符
    add si, 1           ; 保持增加si，直到看到空字符
    or al, 0
    je .return          ; 如果字符串结束了，就结束
    int 0x10            ; 如果没有结束，就打印字符
    jmp .char           ; 继续循环
.return:
    popa
    mov sp, bp
    pop bp
    ret
```

这就差不多了，朋友们。将我们到目前为止所学的内容组合在一起，我们得到以下现实生活中的引导加载程序。

```assembly
bits 16

mov ax, 0x07C0
mov ds, ax
mov ax, 0x07E0      ; 07E0h = (07C00h+200h)/10h, 堆栈段的开始。
mov ss, ax
mov sp, 0x2000      ; 8k的堆栈空间。

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

    mov ah, 0x07        ; 告诉BIOS向下滚动窗口
    mov al, 0x00        ; 清除整个窗口
    mov bh, 0x07        ; 白色文本，黑色背景
    mov cx, 0x00        ; 指定屏幕左上角为(0,0)
    mov dh, 0x18        ; 18h = 24行字符
    mov dl, 0x4f        ; 4fh = 79列字符
    int 0x10            ; 调用视频中断

    popa
    mov sp, bp
    pop bp
    ret

movecursor:
    push bp
    mov bp, sp
    pusha

    mov dx, [bp+4]      ; 从堆栈中获取参数。|bp| = 2，|arg| = 2
    mov ah, 0x02        ; 设置光标位置
    mov bh, 0x00        ; 页0 - 不重要，我们不使用双缓冲
    int 0x10

    popa
    mov sp, bp
    pop bp
    ret

print:
    push bp
    mov bp, sp
    pusha
    mov si, [bp+4]      ; 获取数据的指针
    mov bh, 0x00        ; 页号，仍然是0
    mov bl, 0x00        ; 前景颜色，无关紧要 - 在文本模式中
    mov ah, 0x0E        ; 打印字符到TTY
.char:
    mov al, [si]        ; 从指针位置获取当前字符
    add si, 1           ; 保持增加si，直到看到空字符
    or al, 0
    je .return          ; 如果字符串结束了，就结束
    int 0x10            ; 如果没有结束，就打印字符
    jmp .char           ; 继续循环
.return:
    popa
    mov sp, bp
    pop bp
    ret

times 510-($-$$) db 0
dw 0xAA55
```

至此，我们已经准备好将代码编译成二进制文件。可以通过使用以下命令完成此操作。

```shell
nasm -f bin -o boot.bin boot.asm
```

接下来，我们需要将此二进制文件复制到USB驱动器上。假设您的USB驱动器在/dev/sdb上，可以使用以下命令将二进制文件写入驱动器。

```shell
sudo dd if=boot.bin of=/dev/sdb
```

请注意，这会将二进制文件写入设备，因此请确保您选择了正确的设备。不要使用/dev/sdb1或其他分区设备。

最后，插入USB驱动器，启动计算机。如果您配置了正确的BIOS设置，计算机应该从USB驱动器引导。如果一切正常，您应该看到屏幕上清空的提示。这只是一个简单的示例，但是它向您展示了如何编写一个引导加载程序并与硬件进行交互。如果您想了解更多关于x86汇编语言和引导加载程序开发的知识，可以查找更多资源，探索更复杂的示例并继续深入学习。希望这篇文章对您有所帮助！

## details

根据上述文章，以下是从头开始编写一个简单的x86引导加载程序的详细步骤。这个示例引导加载程序将清除屏幕、移动光标并在屏幕上输出文本。

**步骤 1：准备开发环境**

首先，确保你的系统上安装了NASM（汇编编译器）和Bochs（x86模拟器）。你可以使用你的Linux发行版的包管理器来安装它们。以下是在Ubuntu上的安装示例：

```bash
sudo apt-get update
sudo apt-get install nasm bochs
```

**步骤 2：创建一个新目录**

创建一个新的目录来容纳你的引导加载程序代码。你可以在终端中使用以下命令创建一个目录：

```bash
mkdir bootloader_project
cd bootloader_project
```

**步骤 3：创建汇编代码文件**

使用文本编辑器创建一个汇编代码文件，你可以使用nano或任何其他你喜欢的文本编辑器。在这个文件中，你将编写引导加载程序的汇编代码。创建一个名为`boot.asm`的文件：

```bash
nano boot.asm
```

在 `boot.asm` 文件中，将下面的汇编代码粘贴进去。这个代码将执行屏幕清除、光标移动和文本输出操作。

```assembly
bits 16

start:
    mov ax, 0x07C0
    mov ds, ax
    mov ax, 0x07E0
    mov ss, ax
    mov sp, 0x2000

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

    mov ah, 0x07
    mov al, 0x00
    mov bh, 0x07
    mov cx, 0x00
    mov dh, 0x18
    mov dl, 0x4f
    int 0x10

    popa
    mov sp, bp
    pop bp
    ret

movecursor:
    push bp
    mov bp, sp
    pusha

    mov dx, [bp+4]
    mov ah, 0x02
    mov bh, 0x00
    int 0x10

    popa
    mov sp, bp
    pop bp
    ret

print:
    push bp
    mov bp, sp
    pusha

    mov si, [bp+4]
    mov bh, 0x00
    mov bl, 0x00
    mov ah, 0x0E

.char:
    mov al, [si]
    add si, 1
    or al, al
    jz .return
    int 0x10
    jmp .char

.return:
    popa
    mov sp, bp
    pop bp
    ret

msg db "Hello, Bootloader!", 0

times 510-($-$$) db 0
dw 0xAA55
```

**步骤 4：汇编代码**

保存并关闭 `boot.asm` 文件。接下来，使用NASM来将汇编代码编译为二进制文件：

```bash
nasm -f bin boot.asm -o boot.bin
```

这将生成一个名为 `boot.bin` 的二进制文件，它是引导加载程序的可执行文件。

**步骤 5：创建Bochs配置文件**

创建一个名为 `bochsrc.txt` 的Bochs配置文件，其中包含有关如何运行Bochs的配置信息。使用文本编辑器创建这个文件：

```bash
nano bochsrc.txt
```

将以下内容粘贴到 `bochsrc.txt` 文件中：

```
megs: 32
romimage: file=/usr/share/bochs/BIOS-bochs-latest, address=0xfffe0000
vgaromimage: file=/usr/share/bochs/VGABIOS-lgpl-latest
floppya: 1_44=boot.bin, status=inserted
boot: a
log: bochsout.txt
mouse: enabled=0
display_library: x, options="gui_debug"
```

**步骤 6：运行Bochs**

现在，你可以使用Bochs来模拟引导加载程序的运行。在终端中运行以下命令：

```bash
bochs -f bochsrc.txt
```

Bochs会加载你的引导加载程序，并在模拟环境中运行它。你应该看到屏幕上输出 "Hello, Bootloader!"。

**步骤 7：在USB驱动器上运行引导加载程序（可选）**

如果你想在物理计算机上运行引导加载程序，可以将编译后的 `boot.bin` 写入一个USB驱动器。首先，插入USB驱动器并找出其设备名称（例如，/dev/sdb）。然后，使用以下命令将引导加载程序写入USB驱动器：

```bash
sudo dd if=boot.bin of=/dev/sdX bs=512 count=1
```

将 "of=/dev/sdX" 替换为你的USB驱动器的设备名称，确保不要弄错设备，否则可能会导致数据丢失。

最后，重新启动计算机，并根据需要更改引导顺序以从USB设备启动。你应该能够在物理计算机上看到与Bochs模拟器相同的输出。

这些步骤应该帮助你创建并运行一个简单的x86引导加载程序。这是一个入门级的示例，让你了解引导加载程序的基本概念和操作。如果你想深入学习操作系统开发和引导加载程序的更复杂方面，你可以探索更多的资源和示例。
