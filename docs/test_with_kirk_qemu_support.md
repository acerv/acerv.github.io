# Testing Linux kernel using kirk's qemu support

## Scenario

Let's suppose we have a testing suite which runs on Linux and it might lead
into system crash during its execution, which is the typical case of testing
suites triggering specific syscalls and stress kernel features. We want to keep
track of our execution, to eventually recover system when it's needed. We also
want to test our features on specific kernel version.

[kirk] provides a robust support for this scenario, thanks to the [Qemu]
integration that recognize kernel panics, oops, timeouts, reboots VM and it
keeps track of current running tests. The same result can be reached via SSH,
but we won't cover this option here.

In this guide we will take a look at the described scenario and we will learn
how to use [kirk] in order to run a testing framework on linux kernel.

## Testing framework

We will consider using [LTP], as we know it could crash our kernel.
First of all, each [kirk] framework must implement
[Framework](https://github.com/acerv/kirk/blob/master/libkirk/framework.py)
class and its definition must be placed inside the
[libkirk](https://github.com/acerv/kirk/blob/master/libkirk) folder.
Framework implementation for [LTP] can be found
[here](https://github.com/acerv/kirk/blob/master/libkirk/ltp.py).

A typical [LTP] execution *on host* via [kirk] can be done as following:

    kirk -f ltp:root=/opt/ltp -r syscalls

But it doesn't provide any control over Kernel panic, oops, freeze, etc.
For this reason we need to build fs/kernel images in order to run tests via
[Qemu].

## Build image with buildroot

[Buildroot] project provides an easy and powerful way to build images supported
by [Qemu]. Our minimum requirement is `gcc >= 8`, then we can start building
our images with the following commands:

    cd $BUILDROOT
    make qemu_x86_64_defconfig
    make menuconfig

We need to install LTP inside the buildroot image (notice that we need to give
enough space to FS image, since we have LTP installed):

    Host utilities ->
        host qemu -> yes
        Virtual filesystem support -> yes

    Target packages ->
        Debugging, profiling and benchmark ->
        ltp-testsuite -> yes

    Filesystem images ->
        exact size -> 500MB

Then we can build the images:

    make -j16

Sometimes `gcc-11` compile causes some errors. In this case, `gcc-12`
can be used:

    Toolchain ->
        GCC compiler Version (gcc 12.x) ->
        gcc 12.x -> yes

## Executing kirk

Now that we have both FS and kernel images, we can use [kirk] to run commands
inside virtual machine.

    export IMAGE=$BUILDROOT/output/images/rootfs.ext2
    export KERNEL=$BUILDROOT/output/images/bzImage

    kirk -s qemu:image=$IMAGE:kernel=$KERNEL:user=root:prompt='# ':options='-append "rootwait root=/dev/vda console=ttyS0" -net nic,model=virtio -net user' \
         -c "ls -la /" \
         -v

Notice that we specified `prompt='# '`, since the first busybox prompt string is
`# `. This is really important because first prompt string is used by [kirk] to
recognize if we logged in or not.

The option `-append "rootwait root=/dev/vda console=ttyS0"` is needed to tell
kernel what is our root device and what is our serial console,
`-net nic,model=virtio -net user` is needed in order to communicate with
network.

Command output will be the following:

    ls -la /

    total 25
    drwxr-xr-x   18 root     root      1024 May 31 12:08 .
    drwxr-xr-x   18 root     root      1024 May 31 12:08 ..
    drwxr-xr-x    2 root     root      2048 May 31 12:08 bin
    drwxr-xr-x    8 root     root     12280 May 31 12:09 dev
    drwxr-xr-x    5 root     root      1024 May 31 12:08 etc
    drwxr-xr-x    3 root     root      1024 May 31 12:08 lib
    lrwxrwxrwx    1 root     root         3 May 31 11:43 lib64 -> lib
    lrwxrwxrwx    1 root     root        11 May 31 11:59 linuxrc -> bin/busybox
    drwx------    2 root     root     12288 May 31 12:08 lost+found
    drwxr-xr-x    2 root     root      1024 May  9 20:38 media
    drwxr-xr-x    2 root     root      1024 May  9 20:38 mnt
    drwxr-xr-x    2 root     root      1024 May  9 20:38 opt
    dr-xr-xr-x   99 root     root         0 May 31 12:09 proc
    drwx------    2 root     root      1024 May 31 12:09 root
    drwxr-xr-x    4 root     root       180 May 31 12:09 run
    drwxr-xr-x    2 root     root      1024 May 31 11:59 sbin
    dr-xr-xr-x   12 root     root         0 May 31 12:09 sys
    drwxrwxrwt    2 root     root        80 May 31 12:09 tmp
    drwxr-xr-x    6 root     root      1024 May 31 12:08 usr
    drwxr-xr-x    4 root     root      1024 May 31 12:08 var
    0-pQlWoxTSyF
    Exit code: 0

Now we are ready to run an [LTP] testing suite using kirk:

    export IMAGE=$BUILDROOT/output/images/rootfs.ext2
    export KERNEL=$BUILDROOT/output/images/bzImage

    kirk -f ltp:root=/usr/lib/ltp-testsuite \
         -s qemu:image=$IMAGE:kernel=$KERNEL:user=root:prompt='# ':options='-append "rootwait root=/dev/vda console=ttyS0" -net nic,model=virtio -net user' \
         -r syscalls \
         -v

If everything is fine, we will see the running [LTP] syscalls testing suite.

[kirk]: https://github.com/acerv/kirk
[LTP]: https://github.com/linux-test-project/ltp
[Buildroot]: https://buildroot.org
[Qemu]: https://www.qemu.org
