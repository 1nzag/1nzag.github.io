---
layout: simple
title: Android kernel debugging
---

## **Indrodution**

안드로이드 커널을 디버깅 하는 방법은 실기기에서 디버깅 하는 방법, 가상화를 이용한 방법 등이 있겠으나, 이 포스트에서는 가상화를 이용한 방법으로 커널 디버깅을 하겠다.
환경은 `x86_64`, `linux(Ubuntu)` 이다. 

## **1. Android Emulator**

먼저 안드로이드 커널을 가상화 하여 실행시키려면 에뮬레이터가 필요하다. 

일반적으로 리눅스 커널을 가상화하여 실행시킬때는 `qemu` 를 사용하지만, 안드로이드는 `android emulator` 를 사용한다. 

`android emulator`는 `android studio` 를 설치하면 같이 얻을 수 있다. 

`android studio` 를 설치할 땐, NDK 를 같이 설치할 수 있도록 하자. 

`android studio` 는 공식 홈페이지에서 설치 할 수 있다. 


+ ## ✅**[Android Studio 설치 페이지](https://developer.android.com/studio?gclid=CjwKCAiAu9yqBhBmEiwAHTx5p6wo7nDsHeJ3Or6l9-4Gb9jFKWXseeo9sMyBUWs32qtJ_GETlL0gQBoCQC8QAvD_BwE&gclsrc=aw.ds&hl=ko)**


`android studio` 를 설치하면 일반적으로 홈에 설치 디렉토리가 생긴다. 

```bash
cutebear@cutebear:~$ ls -al ~
total 560
drwxr-x--- 27 cutebear cutebear   4096 11월 18 09:07 .
drwxr-xr-x  3 root     root       4096  8월 24 10:34 ..
drwxrwxr-x  4 cutebear cutebear   4096 11월 18 09:07 .android
drwxrwxr-x  3 cutebear cutebear   4096  8월 24 10:41 Android
drwxrwxr-x  3 cutebear cutebear   4096  8월 24 10:44 AndroidStudioProjects
```

여기에서 명령줄을 사용하여 에뮬레이팅 및 디버깅을 할 수 있도록 `android emulator` 및 `platform-tools` 를 환경변수에 추가해 줘야 한다. 

보통 나는 `.bashrc` 에 `export` 명령어로 추가를 하는 편이다. 

```bash
echo "export PATH=$PATH:/home/cutebear/Android/emulator" >> ~/.bashrc
echo "export PATH=$PATH:/home/cutebear/Android/platform-tools" >> ~/.bashrc
```

이제 `emulator` 명령어로 안드로이드 커널을 에뮬레이팅 할 수 있다. 

안드로이드를 에뮬레이팅 하려면 먼저 가상화 기기를 추가해 주어야 한다. 

`AndroidStudio` 를 실행시키고, 빈 프로젝트를 아무거나 하나 생성한다음, 우측 상단의 디바이스 모양을 클릭하면 가상화 기기를 가져올 수 있다. 

**이 때, 안드로이드 기기의 버전은 가상화 할 커널의 버전과 일칭해야 한다.**

버전이 일치하지 않으면, 부트로더에서 커널 이미지 추출이 제대로 되지 않아 가상화 기기가 부팅되지 않는다. 

가상화 기기를 추가해 주었다면, 다음의 명령어를 통해 가상화 기기를 실행해 볼 수 있다. 

```bash
emulator -avd (virtual machine name)
```

`-avd` 옵션을 통해 가상화 기기의 이름으로 해당 가상화 기기를 실행시킬 수 있다. 

## **2. Boot Android Kernel**

가상화 기기에서 자신이 빌드한 커널을 실행시킬 필요가 있다. (옛날 버전의 커널을 빌드한다던지, KASAN을 적용하여 빌드한다던지 등)

따라서 가상화 기기를 통해 우리가 직접 빌드한 커널을 올린다. 

준비물은 다음과 같다. 

+ **bzImage**: 부트로더와 커널 이미지를 포함한 이미지. 커널 빌드시 제공받는다. 
+ **vmlinux**: 커널 이미지. 커널 빌드시 제공받는다. 

그럼 bzImage를 이용해 빌드한 커널을 실행시켜 보자. 

```bash
emulator -show-kernel -wipe-data -no-snapshot -avd (virtual machine name) -kernel (bzImage path)
```

여기서 몇가지 옵션을 준 걸 볼 수 있는데, 각각의 옵션은 다음과 같다. 

+ **-show-kernel**: 커널 메시지를 프린트 한다. 
+ **-wipe-data**: 안드로이드 에뮬레이터는 가상머신을 종료 할 때 종료하기 전 상태와 디스크 데이터를 `userdata-qemu.img` 에 저장한다. 해당 데이터를 지우는 옵션이다.
+ **-no-snapshot**: 위의 종료하기 전 스냅샷을 찍는 행위를 하지 않는다. 
+ **-kernel**: 부팅하는 데 사용할 커널 이미지를 지정한다. 

## **3. debug android kernel**

이제 `gdb`를 통해 안드로이드 커널을 디버깅해보자. 

`andriod emulator` 는 `qemu` 기반으로 작동하기 때문에  `qemu` 인자를 직접 전달해 줄 수 있다. 

`-qemu (args)` 인자를 주면 `qemu` 에 `args` 인자를 그대로 줄 수 있는데, 이때 `qemu`의 인자는 일반적으로 리눅스 커널을 디버깅 할 떄 사용하는 `qemu` 의 인자를 주면 된다. 

+ **-S**: gdb가 연결 될 때 까지 대기한다.
+ **-gdb**: remote gdb 서버를 연다. 예를 들어 `-gdb tcp:5055` 라는 옵션을 주면 5055 tcp 포트에 디버거 서버를 연다.  
+ **-s**:  `-gdb tcp:1234` 의 줄임 명령어이다. 1234 포트에 remote gdb 서버를 연다.  

따라서 다음과 같은 명령어를 주면 에뮬레이터에서 gdb 가 붙을 때 까지 커널 부팅을 대기 시킬 수 있다. 

```bash
emulator -show-kernel -wipe-data -no-snapshot -avd (virtual machine name) -kernel (bzImage path) -qemu -s -S
```

이제 다음 한편으로 터미널을 하나 더 열어 gdb를 연결시켜주면 안드로이드 커널 디버깅을 할 수 있다.

```bash
gdb -q (vmlinux path) -ex "target remote :1234"
```

이때 destinamtion 으로 `vmlinux` 를 주는 이유는 리눅스 커널의 심볼을 gdb에 인식시켜주기 위함이다. 

```bash
cutebear@cutebear:~/repo$ gdb -q vmlinux -ex "target remote :1234"
GEF for linux ready, type `gef' to start, `gef config' to configure
89 commands loaded and 5 functions added for GDB 12.1 in 0.00ms using Python engine 3.10
Reading symbols from vmlinux...
[!] Using `target remote` with GEF does not work, use `gef-remote` instead. You've been warned.
Remote debugging using :1234
warning: while parsing target description (at line 1): Could not load XML document "i386-64bit.xml"
warning: Could not load XML target description; ignoring
[*] .gdbinit-gef.py:L445 'AARCH64' is deprecated and will be removed in a feature release. use `Elf.Abi.AARCH64`
[*] .gdbinit-gef.py:L445 'ARM' is deprecated and will be removed in a feature release. use `Elf.Abi.ARM`
[*] .gdbinit-gef.py:L445 'MIPS' is deprecated and will be removed in a feature release. use `Elf.Abi.MIPS`
[*] .gdbinit-gef.py:L445 'POWERPC' is deprecated and will be removed in a feature release. use `Elf.Abi.POWERPC`
[*] .gdbinit-gef.py:L445 'POWERPC64' is deprecated and will be removed in a feature release. use `Elf.Abi.POWERPC64`
[*] .gdbinit-gef.py:L445 'RISCV' is deprecated and will be removed in a feature release. use `Elf.Abi.RISCV`
[*] .gdbinit-gef.py:L445 'SPARC' is deprecated and will be removed in a feature release. use `Elf.Abi.SPARC`
[*] .gdbinit-gef.py:L445 'SPARC64' is deprecated and will be removed in a feature release. use `Elf.Abi.SPARC64`
[*] .gdbinit-gef.py:L445 'X86_32' is deprecated and will be removed in a feature release. use `Elf.Abi.X86_32`
[*] .gdbinit-gef.py:L445 'X86_64' is deprecated and will be removed in a feature release. use `Elf.Abi.X86_64`
Display various information of current execution context
Usage:
    context [reg,code,stack,all] [code/stack length]

[*] .gdbinit-gef.py:L445 'AARCH64' is deprecated and will be removed in a feature release. use `Elf.Abi.AARCH64`
[*] .gdbinit-gef.py:L445 'ARM' is deprecated and will be removed in a feature release. use `Elf.Abi.ARM`
[*] .gdbinit-gef.py:L445 'MIPS' is deprecated and will be removed in a feature release. use `Elf.Abi.MIPS`
[*] .gdbinit-gef.py:L445 'POWERPC' is deprecated and will be removed in a feature release. use `Elf.Abi.POWERPC`
[*] .gdbinit-gef.py:L445 'POWERPC64' is deprecated and will be removed in a feature release. use `Elf.Abi.POWERPC64`
[*] .gdbinit-gef.py:L445 'RISCV' is deprecated and will be removed in a feature release. use `Elf.Abi.RISCV`
[*] .gdbinit-gef.py:L445 'SPARC' is deprecated and will be removed in a feature release. use `Elf.Abi.SPARC`
[*] .gdbinit-gef.py:L445 'SPARC64' is deprecated and will be removed in a feature release. use `Elf.Abi.SPARC64`
[*] .gdbinit-gef.py:L445 'X86_32' is deprecated and will be removed in a feature release. use `Elf.Abi.X86_32`
[*] .gdbinit-gef.py:L445 'X86_64' is deprecated and will be removed in a feature release. use `Elf.Abi.X86_64`
Save/restore a working gdb session to file as a script
Usage:
    session save [filename]
    session restore [filename]

0x000000000000fff0 in exception_stacks ()

[ Legend: Modified register | Code | Heap | Stack | String ]
────────────────────────────────────────────────────────────────────────────────────────────────── registers ────
$rax   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$rbx   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$rcx   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$rdx   : 0x0000000000000663  →  0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$rsp   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$rbp   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$rsi   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$rdi   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$rip   : 0x000000000000fff0  →  0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$r8    : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$r9    : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$r10   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$r11   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$r12   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$r13   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$r14   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$r15   : 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
$eflags: [zero carry parity adjust sign trap interrupt direction overflow resume virtualx86 identification]
$cs: 0xf000 $ss: 0x00 $ds: 0x00 $es: 0x00 $fs: 0x00 $gs: 0x00 
────────────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0x0000000000000000│+0x0000: 0x0000000000000000  →  [loop detected]	 ← $rax, $rbx, $rcx, $rsp, $rbp, $rsi, $rdi, $r8, $r9, $r10, $r11, $r12, $r13, $r14, $r15, $ss, $ds, $es, $fs, $gs
0x0000000000000008│+0x0008: 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
0x0000000000000010│+0x0010: 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
0x0000000000000018│+0x0018: 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
0x0000000000000020│+0x0020: 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
0x0000000000000028│+0x0028: 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
0x0000000000000030│+0x0030: 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
0x0000000000000038│+0x0038: 0x0000000000000000  →  0x0000000000000000  →  [loop detected]
──────────────────────────────────────────────────────────────────────────────────────────────── code:x86:64 ────
       0xffea <exception_stacks+12266> add    BYTE PTR [rax], al
       0xffec <exception_stacks+12268> add    BYTE PTR [rax], al
       0xffee <exception_stacks+12270> add    BYTE PTR [rax], al
 →     0xfff0 <exception_stacks+12272> add    BYTE PTR [rax], al
       0xfff2 <exception_stacks+12274> add    BYTE PTR [rax], al
       0xfff4 <exception_stacks+12276> add    BYTE PTR [rax], al
       0xfff6 <exception_stacks+12278> add    BYTE PTR [rax], al
       0xfff8 <exception_stacks+12280> add    BYTE PTR [rax], al
       0xfffa <exception_stacks+12282> add    BYTE PTR [rax], al
──────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, stopped 0xfff0 in exception_stacks (), reason: SIGTRAP
[#1] Id 2, stopped 0xfff0 in exception_stacks (), reason: SIGTRAP
[#2] Id 3, stopped 0xfff0 in exception_stacks (), reason: SIGTRAP
[#3] Id 4, stopped 0xfff0 in exception_stacks (), reason: SIGTRAP
────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0xfff0 → exception_stacks()
[#1] 0x0 →  <irq_stack_union+0> add BYTE PTR [rax], al
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  c
Continuing.
```

## **Conclusion**

간략하게 안드로이드 디버깅을 할 수 있는 방법을 소개했다. `-qemu` 옵션을 통해 non-aslr 등의 편의적인 옵션을 주거나 `KASAN`을 적용한 커널을 디버깅 하면 더 편하게 디버깅을 할 수 있을것이다. 

## **References**

+ <https://cloudfuzz.github.io/android-kernel-exploitation/chapters/scripted-privilege-escalation.html#kernel-debugging>
+ <https://developer.android.com/studio/run/emulator-commandline?hl=ko>
+ <https://developer.android.com/studio/run/managing-avds?hl=ko>
