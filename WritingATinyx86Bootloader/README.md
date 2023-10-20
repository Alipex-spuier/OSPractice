## EN
Writing a Tiny x86 Bootloader
December 27, 2016

All the code/files from this post are available on my Github.


Edit: After some discussion by the good people on Hacker News, it's become clear that when in 16-bit real mode, it's best to use 2-byte registers bp, sp, instead of the 4-byte esb, esp. The article and code have been updated to reflect this.

It might be from being stuck at home with nothing to do over break, or it might be from an actual interest in low-level systems design, but I've taken it upon myself to learn more about OS implementation, starting with the bootloader. So, here we go. All of this information exists in various other places on the web, but there's no better way to learn than by teaching, right? Either way, this piece should serve as primer on what exactly a bootloader does and how to implement a relatively simple one (compared to a beast like GRUB which is ostensibly its own little operating system).

What is a bootloader?
When a computer boots up, the job of getting from nothing to a functioning operating system involves a number of steps. The first thing that happens on an x86 PC is the operation of the BIOS. We'll eschew the discussion of the intricacies of how the BIOS works, but here's what you need to know. When you turn your computer on, the processor immediately looks at physical address 0xFFFFFFF0 for the BIOS code, which is generally on some read-only piece of ROM somewhere in your computer. The BIOS then POSTs, and searches for acceptable boot media. The BIOS accepts some medium as an acceptable boot device if its boot sector, the first 512 bytes of the disk are readable and end in the exact bytes 0x55AA, which constitutes the boot signature for the medium. If the BIOS deems some drive bootable, then it loads the first 512 bytes of the drive into memory address 0x007C00, and transfers program control to this address with a jump instruction to the processor.

Most modern BIOS programs are pretty robust, for example, if the BIOS recognizes several drives with appropriate boot sectors, it will boot from the one with the highest pre-assigned priority; which is exactly why most computers default to booting from USB rather than hard disk if a bootable USB drive is inserted on boot.

Typically, the role of the boot sector code is to load a larger, "real" operating system stored somewhere else on non-volatile memory. In actuality, this is a multi-step process. For example, Master Boot Record, or MBR, is a very common (though now becoming more and more deprecated) boot sector standard for partioned storage devices. Since the boot sector may contain a maximum of 512 bytes of data, an MBR bootloader often simply does the job of passing control to a different, larger bootloader stored somewhere else on disk, whose job in turn is to actually load the operating system (chain-loading). Right now, though, we won't concern ourselves with all this; the goal here isn't to write an operating system (saving that one for another post), but just to get the computer to spit something out onto the screen of our choosing.

It's also important to note that the execution is passed over to bootstrap code while the processor is in real mode, rather than protected mode, which means that (among other things,) access to all of those great features of operating systems that you know and love is out the window. On the other hand, it means that we can directly access the BIOS interrupt calls, which offer some neat low-level functionality.

So, where to begin? I decided to use NASM, a pretty ubiquitous flavor of assembly, for this project. As far as testing goes, it's very much possible to just dd the compiled assembly onto the first 512 bytes of a USB drive and boot the computer from that, but that doesn't have a very fast turnaround, no? Bochs is a neat little x86 IBM-PC compatible emulator which has a bunch of useful features; we'll use this for testing.

Getting Started
Go ahead and download the NASM compiler and Bochs. I use Arch, so pacman is my package manager.

sudo pacman -S nasm bochs
Just for fun, let's start by writing a little stack for our bootloader to use. x86 processors have a number of segment registers, which are used to store the beggining of a 64k segment of memory. In real mode, memory is addressed using a logical address, rather than the physical address. The logical address of a piece of memory consists of the 64k segment it resides in, as well as its offset from the beginning of that segment. The 64k segment of a logical address should be divided by 16, so, given a logical address beginning at 64k segment A, with offset B, the reconstructed physical address would be A*0x10 + B.

For example, the processor has a DS register for the data segment. Since our code resides at 0x7C00, the data segment may begin at 0x7C0, which we can set with

mov ax, 0x7C0
mov ds, ax
We have to load the segment into another register (here it's ax) first; we can't directly stick it in the segment register. Let's start the storage for the stack directly after the 512 bytes of the bootloader. Since the bootloader extends from 0x7C00 for 512 bytes to 0x7E00, the stack segment, SS, will be 0x7E0.

mov ax, 0x7E0
mov ss, ax
On x86 architectures, the stack pointer decreases, so we must set the initial stack pointer to a number of bytes past the stack segment equal to the desired size of the stack. Since the stack segment can address 64k of memory, let's make an 8k stack, by setting SP to 0x2000.

mov sp, 0x2000
We're now free to use the standard calling convention in order to safely pass control over to different functions. We can use pushin order to push caller-saved registers on to the stack, pass parameters to the callee again with push, and then use callto save the current program counter to the stack, and perform an unconditional jump to the given label.

Alright, now that all that is out of the way, let's figure out a way to clear the screen, move the pointer, and write some text. This is where real mode and BIOS interrupt calls come in to play. By storing certain registers with certain parameters and then sending a particular opcode to the BIOS as an interrupt, we can do a bunch of cool stuff. For example, by storing 0x07 in the AH register and sending interrupt code 0x10 to the BIOS, we can scroll the window down by a number of rows. See the spec here. Note that the registers AH and AL refer to the most and least significant bytes of the 16 bit register AX. Thus, we could effectively update both their values at once by simply pushing a 16 bit value to AX, however, we'll opt for the clearer approach of updating each 1-byte subregister at a time.

If you look at the spec, you'll see that we need to set AH to 0x07, and AL to 0x00. the value of register BH refers to the BIOS color attribute, which for our purposes will be black background (0x0) behind light-gray (0x7) text, so we must set BH to 0x07. Registers CX and DX refer to the subsection of the screen that we want to clear. The standard number of character rows/cols here is 25/80, so we set CH and CL to 0x00 to set (0,0) as the top left of the screen to clear, and DH as 0x18 = 24, DL as 0x4f = 79. Putting this all together in a function, we get the following snippet.

clearscreen:
    push bp
    mov bp, sp
    pusha

    mov ah, 0x07        ; tells BIOS to scroll down window
    mov al, 0x00        ; clear entire window
    mov bh, 0x07            ; white on black
    mov cx, 0x00        ; specifies top left of screen as (0,0)
    mov dh, 0x18        ; 18h = 24 rows of chars
    mov dl, 0x4f        ; 4fh = 79 cols of chars
    int 0x10        ; calls video interrupt

    popa
    mov sp, bp
    pop bp
    ret
The overhead at the beginning and end of the subroutine allows us to adhere to the standard calling convention between caller and callee. pushaand popapush and pop all general registers on and off the stack. We save the caller's base pointer (4 bytes), and update the base pointer with the new stack pointer. At the very end, we essentially mirror this process.

Nice. Now let's write a subroutine for moving the cursor to an arbitrary (row,col) position on the screen. Int 10/AH=02h does this nicely. This subroutine will be slightly different, since we'll need to pass it an argument. According to the spec, we must set register DX to a two byte value, the first representing the desired row, and second the desired column. AH has gotta be 0x02, BH represents the page number we want to move the cursor to. This parameter has to do with the fact that the BIOS allows you to draw to off-screen pages, in order to facilitate smoother visual transitions by rendering off-screen content before it is shown to the user. This is called multiple or double buffering. We don't really care about this, however, so we'll just use the default page of 0.

Putting it all together, we have the following subroutine.

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
The only thing that might look unusual is the mov dx, [bp+4]. This moves the argument we passed into the DX register. The reason we offset by 4 is that the contents of bp takes up 2 bytes on the stack, and the argument takes up two bytes, so we have to offset a total of 4 bytes from the actual address of bp. Note also that the caller has the responsibility to clean the stack after the callee returns, which amounts to removing the arguments from the top of the stack by moving the stack pointer.

The final subroutine we want to write is simply one that, given a pointer to the beginning of a string, prints that string to the screen beginning at the current cursor position. Using the video interrupt code with AH=0Eh works nicely. First off, we can define some data and store a pointer to its starting address with something that looks like this.

msg:    db "Oh boy do I sure love assembly!", 0
The 0 at the end terminates the string with a null character, so we'll know when the string is done. We can reference the address of this string with msg. Then, the rest is pretty much like what we just saw with movecursor. We use some more labels and a conditional jump, but at risk of being too verbose, understanding the code is left as an excercise to the reader ;).

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
And that'll just about do it folks. Plugging everything we have so far together, we get the following real life bootloader.

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
Some things might not be familiar in there. The first line of the program tells the assembler that we're working in 16-bit real mode. The lines cli and hlt after we finish printing tell the processor not to accept interrupts and to halt processing. Finally, remember that the code in a bootsector has to be exactly 512 bytes, ending in 0xAA55? The last two lines pad the binary to a length of 510 bytes, and make sure the file ends with the appropriate boot signature.

That's it folks.


Oh, did you actually want to run the code? Go ahead and save the code above into a file, say boot.asm.

Then, the following command generates a nice binary from our asm bootloader code.

    nasm -f bin boot.asm -o boot.com
Then, in the same directory, whip up a file called bochsrc.txt, and fill it up with the following

    megs: 32
    romimage: file=/usr/share/bochs/BIOS-bochs-latest, address=0xfffe0000
    vgaromimage: file=/usr/share/bochs/VGABIOS-lgpl-latest
    floppya: 1_44=boot.com, status=inserted
    boot: a
    log: bochsout.txt
    mouse: enabled=0
    display_library: x, options="gui_debug"
This just contains some simple config stuff for Bochs, nothing too fancy. Basically you're just telling Bochs that your boot medium is a 1.44 Meg floppy with your binary loaded on it. Finally, you can just call

bochs -f bochsrc.txt
to run Bochs using the config file you just wrote, and voila, you should see something along the lines of this.


Wow. Pretty boring, huh? If you have a USB drive laying around anywhere, you can do something marginally cooler. Plug that puppy in and find out where it lives (use dmesg or something). Mine was on /dev/sdb. Using dd, run

sudo dd if=boot.com of=/dev/sdb bs=512 count=1
This will copy the first 512 bytes of your bootloader (read: all of it), to the first 512 bytes of your USB drive. If you want to make sure everything copied over all well and good, you can let if=/dev/sdb and of=test.com, then diff the two files. They should be identical. Then, it's just a matter of restarting your computer (and potentially changing boot priority to boot from USB first), and you should see the same boring text you see in an emulator just minutes ago. Well done.

It should be said, again, that most real bootloaders are orders of magnitutde more complex than this one, however I think this is a pretty good proof of concept/learning tool. Hopefully you learned something from this - I certainly did, even if the end result was far more underwhelming than I expected it to be.

## CN
编写一个小型x86引导加载程序
2016年12月27日

这篇文章中的所有代码/文件都可以在我的Github上找到。

编辑：经过Hacker News上的一些讨论，很明显，在16位实模式下，最好使用2字节寄存器bp和sp，而不是4字节esb和esp。文章和代码已经更新以反映这一点。

这可能是因为我被困在家里没有事情可做，或者可能是因为对低级系统设计产生了真正的兴趣，但我已经自己承担起了更多关于操作系统实现的知识，从引导加载程序开始。所以，让我们开始吧。所有这些信息都存在于网络上的各种其他地方，但没有比教学更好的学习方式，对吧？无论如何，这篇文章应该作为关于引导加载程序究竟是什么以及如何实现一个相对简单的引导加载程序的入门文章（与像GRUB这样的庞然大物相比，它显然是自己的小型操作系统）。

什么是引导加载程序？
当计算机启动时，从无到有变成一个正常运行的操作系统涉及多个步骤。在x86 PC上发生的第一件事是BIOS的运行。我们将避开讨论BIOS的工作细节，但这里有一些你需要知道的内容。当你打开计算机时，处理器立即查看物理地址0xFFFFFFF0以获取BIOS代码，通常在计算机的某个只读ROM部分中。然后，BIOS进行POST（电源自检），并搜索可接受的引导介质。如果BIOS将某个驱动器视为可引导的，那么它会将驱动器的前512个字节加载到内存地址0x007C00，并通过跳转指令将程序控制传递到该地址的处理器。

大多数现代BIOS程序都非常强大，例如，如果BIOS识别到具有适当引导扇区的多个驱动器，它将从具有最高预分配优先级的驱动器引导；这正是为什么大多数计算机在引导时插入可引导的USB驱动器时会默认从USB而不是硬盘引导的原因。

通常，引导扇区代码的作用是加载存储在非易失性存储器的另一个较大的“真正”的操作系统。实际上，这是一个多步骤的过程。例如，主引导记录（Master Boot Record，MBR）是一个非常常见（尽管现在越来越被弃用）的分区存储设备的引导扇区标准。由于引导扇区最多包含512字节的数据，MBR引导加载程序通常只是将控制权传递给存储在磁盘的其他地方的不同、较大的引导加载程序的任务，而后者的任务是实际加载操作系统（链式加载）。然而，现在我们不会关心这一切；我们的目标不是编写操作系统（将它留到另一篇文章中），只是让计算机在我们选择的屏幕上输出一些内容。

还值得注意的是，在处理器处于实模式而不是受保护模式时将执行代码，这意味着（除其他事项之外），无法访问你所熟悉和喜爱的操作系统的所有出色功能。另一方面，这意味着我们可以直接访问BIOS中断调用，提供了一些有趣的低级功能。

那么，从哪里开始呢？我决定在这个项目中使用NASM，这是一种非常常见的汇编语言。至于测试，可以将编译后的汇编代码使用`dd`命令复制到USB驱动器的前512字节，然后从中引导计算机，但这不会有很快的反馈，对吧？Bochs是一个很棒的x86 IBM-PC兼容模拟器，它具有许多有用的功能；我们将用它进行测试。

开始吧
请先下载NASM编译器和Bochs。我使用Arch Linux，所以`pacman`是我的软件包管理器。

```bash
sudo pacman -S nasm bochs
```

为了好玩，让我们从为引导加载程序编写一个小型堆栈开始。x86处理器有多个段寄存器，用于存储64K段内存的起始位置。在实模式下，内存是使用逻辑地址而不是物理地址进行寻址的。内存的逻辑地址由它所在的64K段和它距离该段开头的偏移量组成。逻辑地址的64K段应该除以16，所以，假设逻辑地址从64K段A的开始，偏移量为B，那么重建的物理地址将是A*0x10 + B。

例如，处理器有一个数据段寄存器DS。由于我们的代码位于0x7C00，数据段可以从0x7C0开始，我们可以使用以下代码设置它：

```assembly
mov ax, 0x7C0
mov ds, ax
```

我们必须首先将段加载到另一个寄存器（这里是ax）中，然后才能将它直接放入段寄存器。让我们将堆栈的存储区从引导加载程序的前512字节之后开始。由于引导加载程序从0x7C00扩展到512字节

到0x7E00，堆栈段SS将为0x7E0。

```assembly
mov ax, 0x7E0
mov ss, ax
```

在x86架构中，堆栈指针(SP)减小，因此我们必须将初始堆栈指针设置为距离堆栈段的一些字节，等于堆栈的所需大小。由于堆栈段可以寻址64K内存，我们可以创建一个8K堆栈，将SP设置为0x2000。

```assembly
mov sp, 0x2000
```

现在，我们可以使用标准的调用约定来安全地将控制权传递给不同的函数。我们可以使用`push`来将调用者保存的寄存器推送到堆栈上，再次使用`push`将参数传递给被调用函数，然后使用`call`将当前程序计数器保存到堆栈上，并跳转到给定的标签。

好了，既然所有这些都已经讲清楚，让我们想出一种方法来清除屏幕、移动光标并写一些文本。这就是实模式和BIOS中断调用派上用场的地方。通过存储带有特定参数的寄存器，然后将特定的操作码发送给BIOS作为中断，我们可以做许多很酷的事情。例如，通过将AH寄存器中存储的0x07和发送给BIOS的中断代码0x10，我们可以将窗口向下滚动若干行。查看规格[这里](https://stanislavs.org/helppc/int_10.html)。请注意，寄存器AH和AL分别指的是16位寄存器AX的最高字节和最低字节。因此，我们可以通过简单地将16位值推送到AX来同时更新它们的值，不过，我们选择更清晰的方法，逐个更新每个1字节子寄存器。

如果查看规格，你会看到我们需要将AH设置为0x07，AL设置为0x00。寄存器BH的值是BIOS颜色属性，对于我们的目的，将是黑色背景（0x0）和浅灰色（0x7）文本，所以我们必须将BH设置为0x07。寄存器CX和DX指的是我们要清除的屏幕的子部分。标准字符行/列数是25/80，因此我们将CH和CL设置为0x00，将屏幕的左上角设置为(0,0)，将DH设置为0x18（24），将DL设置为0x4f（79）。将所有这些组合在一起，我们得到以下代码片段。

```assembly
clearscreen:
    push bp
    mov bp, sp
    pusha

    mov ah, 0x07        ; 告诉BIOS向下滚动窗口
    mov al, 0x00        ; 清除整个窗口
    mov bh, 0x07        ; 白色文本，黑色背景
    mov cx, 0x00        ; 指定屏幕左上角为(0,0)
    mov dh, 0x18        ; 18h = 24个字符行
    mov dl, 0x4f        ; 4fh = 79个字符列
    int 0x10            ; 调用视频中断

    popa
    mov sp, bp
    pop bp
    ret
```

子程序开头和结尾的开销允许我们遵守调用者和被调用者之间的标准调用约定。`pusha`和`popa`在堆栈上推送和弹出所有通用寄存器。我们保存了调用者的基指针（4字节），并使用新堆栈指针更新基指针。最后，我们基本上反映了这个过程。

不错。现在让我们编写一个子程序，将光标移动到屏幕上的任意（行，列）位置。Int 10/AH=02h可以很好地完成这个任务。这个子程序将稍微不同，因为我们需要向它传递一个参数。根据规格，我们必须将寄存器DX设置为一个两字节值，第一个表示所需的行，第二个表示所需的列。AH必须是0x02，BH表示我们想要将光标移动到的页号。这个参数与BIOS允许你绘制到屏幕外的页面有关，以便在向用户显示屏幕内容之前渲染屏幕外内容以实现更流畅的视觉过渡。这被称为多重缓冲或双缓冲。然而，我们并不太关心这个，所以我们将只使用默认的页面0。

将所有这些组合在一起，我们得到以下子程序。

```assembly
movecursor:
    push bp
    mov bp, sp
    pusha

    mov dx, [bp+4]      ; 从堆栈获取参数。 |bp| = 2, |arg| = 2
    mov ah, 0x02        ; 设置光标位置
    mov bh, 0x00        ; 页号0 - 不重要，我们不使用双缓冲
    int 0x10            ; 调用视频中断

    popa
    mov sp, bp
    pop bp
    ret
```

唯一看起来可能不寻常的是`mov dx, [bp+4]`。这将参数从堆栈传递到DX寄存器中。之所以要偏移4是因为堆栈上的bp占用2个字节，而参数占用2个字节，所以我们必须总共偏移4个字节

，从bp的实际地址开始计算。还要注意，调用者在被调用者返回后负责清除堆栈，这涉及将堆栈指针向上移动以删除堆栈顶部的参数。

我们要编写的最后一个子程序只是一个根据字符串开头的指针，在当前光标位置开始将该字符串打印到屏幕的子程序。使用AH=0Eh的视频中断代码可以很好地完成这项任务。首先，我们可以定义一些数据并使用类似以下方式存储指向其起始地址的指针。

```assembly
msg:    db "Oh boy do I sure love assembly!", 0
```

末尾的0是使用空字符终止字符串，以便我们知道字符串何时结束。我们可以使用`msg`来引用这个字符串的地址。然后，其余部分基本上与我们刚刚在`movecursor`中看到的类似。我们使用一些标签和条件跳转，但为了避免过于冗长，我将理解代码的任务留给读者；)。

```assembly
print:
    push bp
    mov bp, sp
    pusha
    mov si, [bp+4]      ; 获取数据的指针
    mov bh, 0x00        ; 页号，再次为0
    mov bl, 0x00        ; 前景颜色，与文本模式无关
    mov ah, 0x0E        ; 将字符打印到TTY
.char:
    mov al, [si]        ; 从我们的指针位置获取当前字符
    add si, 1           ; 保持增加si，直到我们看到空字符
    or al, 0
    je .return          ; 如果字符串结束了，就结束
    int 0x10            ; 如果还没结束，就打印字符
    jmp .char           ; 继续循环
.return:
    popa
    mov sp, bp
    pop bp
    ret
```

那就差不多了，伙计们。将我们到目前为止所拥有的一切组合起来，我们得到以下真实的引导加载程序。

```assembly
bits 16

mov ax, 0x07C0
mov ds, ax
mov ax, 0x07E0      ; 07E0h = (07C00h+200h)/10h, 堆栈段的开始。
mov ss, ax
mov sp, 0x2000      ; 8K的堆栈空间。

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
    mov dh, 0x18        ; 18h = 24个字符行
    mov dl, 0x4f        ; 4fh = 79个字符列
    int 0x10            ; 调用视频中断

    popa
    mov sp, bp
    pop bp
    ret

movecursor:
    push bp
    mov bp, sp
    pusha

    mov dx, [bp+4]      ; 从堆栈获取参数。 |bp| = 2, |arg| = 2
    mov ah, 0x02        ; 设置光标位置
    mov bh, 0x00        ; 页号0 - 不重要，我们不使用双缓冲
    int 0x10            ; 调用视频中断

    popa
    mov sp, bp
    pop bp
    ret

print:
    push bp
    mov bp, sp
    pusha
    mov si, [bp+4]      ; 获取数据的指针
    mov bh, 0x00        ; 页号，0再次
    mov bl, 0x00        ; 前景颜色，与文本模式无关
    mov ah, 0x0E        ; 将字符打印到TTY
.char:
    mov al, [si]        ; 从

我们的指针位置获取当前字符
    add si, 1           ; 保持增加si，直到我们看到空字符
    or al, 0
    je .return          ; 如果字符串结束了，就结束
    int 0x10            ; 如果还没结束，就打印字符
    jmp .char           ; 继续循环
.return:
    popa
    mov sp, bp
    pop bp
    ret

msg:    db "Oh boy do I sure love assembly!", 0
```

有许多改进的空间，但在学习过程中，我发现这个小程序很有趣。此程序用NASM汇编编写，然后使用Bochs模拟器进行测试。你可以将它写入USB驱动器并在启动计算机时将其引导。有了这个基础，你可以进一步发展自己的引导加载程序或深入研究操作系统开发。如果你想进一步了解NASM、x86汇编语言或引导加载程序的编写，可以查阅相关文档和教程。继续学习，开启更多有趣的探索之旅！

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
