## ucore Lab1 实验报告

### 练习1：理解通过make生成执行文件的过程

#### 1. 操作系统镜像文件`ucore.img`是如何一步一步生成的？

执行`make V=`，结果如下

    + cc kern/init/init.c
    i386-elf-gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
    kern/init/init.c:95:1: warning: 'lab1_switch_test' defined but not used [-Wunused-function]
     lab1_switch_test(void) {
     ^
    + cc kern/libs/readline.c
    i386-elf-gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/readline.c -o obj/kern/libs/readline.o
    ...
    + ld bin/bootblock
    i386-elf-ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
    'obj/bootblock.out' size: 472 bytes
    build 512 bytes boot sector: 'bin/bootblock' success!
    dd if=/dev/zero of=bin/ucore.img count=10000
    10000+0 records in
    10000+0 records out
    5120000 bytes transferred in 0.030496 secs (167891520 bytes/sec)
    dd if=bin/bootblock of=bin/ucore.img conv=notrunc
    1+0 records in
    1+0 records out
    512 bytes transferred in 0.000051 secs (10034970 bytes/sec)
    dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
    138+1 records in
    138+1 records out
    70759 bytes transferred in 0.000212 secs (333841121 bytes/sec)

大致步骤为：首先调用`i386-elf-gcc`编译源码生成一系列`.o`文件，然后调用`i386-elf-ld`把它们链接起来，最后通过`dd`生成镜像。

<!-- TODO: in detail -->

#### 2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

根据`sign.c`中的检查逻辑

    if (st.st_size > 510) {
        fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
        return -1;
    }
    ...
    buf[510] = 0x55;
    buf[511] = 0xAA;

一个被系统认为是符合规范的硬盘主引导扇区的特征有：

- 镜像实际内容不大于510字节，
- 总大小为512字节，且
- 最后两个字节的内容分别为`0x55`与`0xAA`。

### 练习2：使用qemu执行并调试lab1中的软件

首先启动`qemu`

    qemu-system-i386 -S -s -d in_asm -D bin/q.log -monitor stdio -hda bin/ucore.img -serial null

然后启动`gdb`，在`gdb`终端执行

    file obj/bootblock.o
    target remote :1234
    break bootmain
    continue
