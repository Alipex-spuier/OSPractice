# EN
[Writing a Tiny x86 Bootloader](https://www.joe-bergeron.com/posts/Writing%20a%20Tiny%20x86%20Bootloader/)

# CN
## 编写一个微小的x86引导程序
#### 2016年12月27日

本文中的所有代码/文件都可以在我的[Github](https://github.com/Jophish/tiny-bootstrap)上找到。

编辑：在[good people on Hacker News](https://news.ycombinator.com/item?id=13268781)上的一些讨论后，已经明确，在16位实模式下，最好使用2字节寄存器bp和sp，而不是4字节的esb和esp。文章和代码已经进行了更新，以反映这一点。

___

可能是因为被困在家里，没有休息时可做的事情，或者可能是出于对低级系统设计的真正兴趣，但我已经决定自己学习有关操作系统实施的更多知识，首先从引导程序开始。因此，让我们开始吧。虽然在网上已经存在各种其他地方的所有这些信息，但通过教授来学习无疑是更好的方法，对吧？无论如何，这篇文章应该作为对引导程序到底是什么以及如何实现一个相对简单引导程序的入门指南（与像[GRUB](https://en.wikipedia.org/wiki/GNU_GRUB)这样的庞然大物相比，它显然是一个独立的小型操作系统）。
