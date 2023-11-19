---
layout: simple
title: Linux Kernel Debugging
---

## **1. Introdution**

리눅스 커널의 취약점을 분석하기 위해서는 디버깅이 필요할 때가 있다. 

가령, 익스플로잇을 작성하거나, 퍼징을 할때 디버깅이 필요하다. 

이 포스트에서는 가상 환경에서 리눅스 커널을 디버깅 하는 방법을 소개한다. 

리눅스 커널 디버깅을 하는 방법은 3가지가 존재한다. 

+ **1.kernel debugging**: serial 포트 또는 네트워크 포트 등을 통해 실제 컴퓨터에서 작동하는 커널을 디버깅 하는 방법이다. 
    + 커널과 디버거가 같은 메모리에서 동작하므로, 디버거 메모리가 변조될 위험성이 있다. 
+ **2. hardware debugging**: JTAG 등의 하드웨어 포트를 통해 해당 보드 위에서 작동하는 커널을 디버깅 하는 방법이다. 
+ **3. virtual machine debugging**: 커널을 가상 머신 위에서 구동하여 디버깅 하는 방법이다. 별도의 디버깅 장비가 필요하지 않으며, 사용이 간편하다. 

해당 포스트에서는 3번 VM debugging 을 하는 방법을 소개한다. 

## **2. Prepare**

가상 머신 환경에서 리눅스 커널 디버깅을 하기 위해선 다음과 같은 준비물이 필요하다. 

+ **bzImage**: 부트로더와 커널이미지가 함께 빌드 된 이미지이다. 해당 이미지는 커널을 구동할때 필요한 필수 이미지이다. 
+ **vmlinux**: 커널 이미지이다. 디버거가 커널의 심볼을 확인할때 쓰일 수 있다. `bzImage`에서 커널 이미지를 추출 할 수 있다. 
+ **rootfs**: 커널이 사용할 파일 시스템이다. 
+ **VM(qemu)**: 커널을 구동할 가상 머신이다. 해당 포스트에선 `qemu`를 쓴다. 
+ **gdb**: 커널을 디버깅 할 디버거이다. 

## **3. Extract vmlinux**

본 포스트에서 디버깅할 커널의 이미지는 `CCE 2023 Quals`에서 배포한 `baby_kernel` 문제에서 제공되는 이미지이다. 

해당 문제에는 다음과 같은 파일들이 첨부되어 있다. 

```bash
.
├── Dockerfile
├── docker-compose.yml
└── share
    ├── _bzImage.extracted
    ├── babykernel.ko
    ├── bzImage
    ├── local_run.sh
    ├── pow.py
    ├── pow_client.py
    ├── rootfs.img.gz
    ├── server_run.sh
    └── ynetd
```

먼저 첫번째 준비물인 `bzImage`가 첨부되어 있다. 하지만 `vmlinux`가 없다. 

부트로더 이미지는 먼저 부트로더가 부팅을 하고, 압축해 놓은 커널 이미지인 `vmlinux`를 압축 해제 하여 메모리에 적재한다. 그 후 메모리의 커널 엔트리 포인트로 점프하여 커널을 실행한다. 

![BootImage](/assets/img/study/linux_kernel_debugging/boot_image.png)

즉, `bzImage` 안에는 압축된 커널 이미지인 `vmlinux`가 있다. 


따라서 `bzImage`에서 `vmlinux`를 추출하자.

아주 좋게도 `bzImage`에서 `vmlinux` 를 추출하는 스크립트가 존재한다. 

```bash
#!/bin/sh
# SPDX-License-Identifier: GPL-2.0-only
# ----------------------------------------------------------------------
# extract-vmlinux - Extract uncompressed vmlinux from a kernel image
#
# Inspired from extract-ikconfig
# (c) 2009,2010 Dick Streefland <dick@streefland.net>
#
# (c) 2011      Corentin Chary <corentin.chary@gmail.com>
#
# ----------------------------------------------------------------------

check_vmlinux()
{
	# Use readelf to check if it's a valid ELF
	# TODO: find a better to way to check that it's really vmlinux
	#       and not just an elf
	readelf -h $1 > /dev/null 2>&1 || return 1

	cat $1
	exit 0
}

try_decompress()
{
	# The obscure use of the "tr" filter is to work around older versions of
	# "grep" that report the byte offset of the line instead of the pattern.

	# Try to find the header ($1) and decompress from here
	for	pos in `tr "$1\n$2" "\n$2=" < "$img" | grep -abo "^$2"`
	do
		pos=${pos%%:*}
		tail -c+$pos "$img" | $3 > $tmp 2> /dev/null
		check_vmlinux $tmp
	done
}

# Check invocation:
me=${0##*/}
img=$1
if	[ $# -ne 1 -o ! -s "$img" ]
then
	echo "Usage: $me <kernel-image>" >&2
	exit 2
fi

# Prepare temp files:
tmp=$(mktemp /tmp/vmlinux-XXX)
trap "rm -f $tmp" 0

# That didn't work, so retry after decompression.
try_decompress '\037\213\010' xy    gunzip
try_decompress '\3757zXZ\000' abcde unxz
try_decompress 'BZh'          xy    bunzip2
try_decompress '\135\0\0\0'   xxx   unlzma
try_decompress '\211\114\132' xy    'lzop -d'
try_decompress '\002!L\030'   xxx   'lz4 -d'
try_decompress '(\265/\375'   xxx   unzstd

# Finally check for uncompressed images or objects:
check_vmlinux $img

# Bail out:
echo "$me: Cannot find vmlinux." >&2
```

다음 스크립트를 이용해 vmlinux를 추출하자

```bash
graypanda@graypanda-inzag:~/baby_kernel/share$ ./extract-vmlinux.sh ./bzImage > ./vmlinux
graypanda@graypanda-inzag:~/baby_kernel/share$ file vmlinux 
vmlinux: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=76c48c00bc16c56e0b401595ebebeb48871da4a6, stripped
```

## **4. QEMU**

앞서 말했듯 우리는 가상머신 (`qemu`)를 이용하여 커널을 디버깅 할 것이다. 

문제에는 `qemu`를 구동시켜주는 스크립트가 존재한다. 

```bash
graypanda@graypanda-inzag:~/baby_kernel/share$ cat server_run.sh 
#!/bin/sh
qemu-system-x86_64 \
    -m 128M \
    -cpu kvm64,+smep \
    -kernel bzImage \
    -initrd $1 \
    -snapshot \
    -nographic \
    -monitor /dev/null \
    -no-reboot \
    -append "console=ttyS0 kaslr kpti=1 quiet panic=1"
```

여기서 qemu에 관련된 옵션 여러개를 볼 수 있는데, 각 옵션은 다음과 같다. 

+ **-m**: 가상머신에 할당되는 메모리 양이다. 
+ **-cpu**: 가상머신에 사용될 CPU 모델과 기능이다. 
    + 위의 파일에서 `kvm64`는 일반적으로 많이 사용되는 모델이고, `smep` 은 **Supervisor Mode Execution Protection** 이라는 mitigation을 적용한다는 뜻이다. 
+ **-kernel**: 사용할 커널 이미지이다. 
+ **-initrd**: 초기 램 디스크(initrd)를 지정하는 옵션이다. 여기서는 루트 파일시스템을 주는 옵션이라고 까지만 알고 있자. 
+ **-snapshot**: 스냅샷 모드에서 실행한다. 이 모드에서 실행하면 가상 디스크에 대한 변경사항을 없애고, 가상머신 종료시 가상 디스크를 지운다. 
+ **-nographic**: 그래픽 모드를 사용하지 않고 출력을 콘솔로 리다이렉션 한다. 
+ **monitor**: QEMU 모니터 를 출력하는 디스크립터 설정이다. 
    + 해당 파일에서는 `/dev/null`로 주었기 때문에 사실상 모니터 출력을 제외한다고 보면 된다. 
+ **-no-reboot**: 가상머신이 재부팅 되는 것을 방지한다. 패닉이 발생하면, 가상머신은 재부팅 되지 않고 종료된다. 
+ **-append**: 커널 명령 줄 매개변수이다. (kernel commands)
    + **console**: 콘솔 출력을 지정한다.
    + **kaslr**: 커널 mitigation 중 하나긴 kaslr을 활성화 한다. 
    + **kpti**: cpu 취약점을 완화 하기 위한 mitigation인 Kernel Page Table Isolation 기법을 지정한다. 
    + **quiet**: 커널의 출력을 제한한다. 
    + **panic**: 커널 패닉 이후 재시작 하는 시간을 지정한다. 

위의 쉘 스크립트는 파일시스템이란 인자를 받아 그것으로 `bzImage`및 여러 옵션들과 함께 가상 머신을 부팅하는 스크립트라는 것을 알 수 있다. 

## **5. Debugging Kernel**

나는 디버깅을 할 것이기 때문에 위의 스크립트에 다음의 옵션을 추가하여 디버깅을 할 수 있게, 디버깅이 더 편할 수 있게 한다. 

+ **-append "nokaslr"**: kaslr 옵션을 끈다. 
+ **-S**: 외부의 gdb가 붙을 때 까지 커널 실행을 대기한다. 
+ **-s**: tcp 1234 번 포트로 디버거 포트를 연다. 
    + 1234번 포트 말고 다른 포트를 지정하고 싶으면 `-gdb tcp:5055`를 주면 5055 번 포트로 디버거 포트를 열 수 있다. 

스크립트를 수정하면 다음과 같다. 

```bash
#!/bin/sh
qemu-system-x86_64 \
    -m 128M \
    -cpu kvm64,+smep \
    -kernel bzImage \
    -initrd $1 \
    -snapshot \
    -nographic \
    -monitor /dev/null \
    -s
    -S
    -no-reboot \
    -append "console=ttyS0 nokaslr kpti=1 quiet panic=1"

```

그릐고 명령을 명령을 실행해보자

```bash
graypanda@graypanda-inzag:~/baby_kernel/share$ ./server_run.sh rootfs.img.gz
```

그럼 qemu가 대기한 상태로 기다린다. 

그럼 다른 터미널을 열어 디버거를 연결해주자

```bash
graypanda@graypanda-inzag:~/baby_kernel/share$ gdb -q vmlinux -ex "target remote :1234"
GEF for linux ready, type `gef' to start, `gef config' to configure
89 commands loaded and 5 functions added for GDB 12.1 in 0.01ms using Python engine 3.10
Reading symbols from vmlinux...
(No debugging symbols found in vmlinux)
[!] Using `target remote` with GEF does not work, use `gef-remote` instead. You've been warned.
Remote debugging using :1234
0x000000000000fff0 in ?? ()

[ Legend: Modified register | Code | Heap | Stack | String ]
──────────────────────────────────────────────────────────────────────────────────── registers ────
[!] Command 'registers' failed to execute properly, reason: [Errno 13] Permission denied: '/proc/1/maps'
──────────────────────────────────────────────────────────────────────────────────────── stack ────
[!] Command 'dereference' failed to execute properly, reason: [Errno 13] Permission denied: '/proc/1/maps'
────────────────────────────────────────────────────────────────────────────────── code:x86:64 ────
       0xffea                  add    BYTE PTR [rax], al
       0xffec                  add    BYTE PTR [rax], al
       0xffee                  add    BYTE PTR [rax], al
 →     0xfff0                  add    BYTE PTR [rax], al
       0xfff2                  add    BYTE PTR [rax], al
       0xfff4                  add    BYTE PTR [rax], al
       0xfff6                  add    BYTE PTR [rax], al
       0xfff8                  add    BYTE PTR [rax], al
       0xfffa                  add    BYTE PTR [rax], al
────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, stopped 0xfff0 in ?? (), reason: SIGTRAP
──────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0xfff0 → add BYTE PTR [rax], al
[#1] 0x0 → add BYTE PTR [rax], al
───────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  c
Continuing.
```

디버거가 연결 된 것을 확인 할 수 있고, 다음과 qemu도 실행되는 것을 볼 수 있다. 

```bash
~ $ ls
babykernel.ko  etc            linuxrc        rootfs.img.gz  usr
bin            flag           proc           sbin
dev            init           root           tmp
~ $ uname -a
Linux (none) 5.19.17 #1 SMP PREEMPT_DYNAMIC Thu May 18 00:44:59 KST 2023 x86_64 GNU/Linux
```

이제 마음껏 디버깅을 할 수 있게 됐다.

## **Conclusion**

qemu 옵션을 보니 여러 커널 mitigation 이 있다는 것을 알게 됐다. 아마 더 많을거고 안드로이드는 더 많을 것이다. 

앞으로 이런 mitigation 들이 뭐고 어떻게 우회 할 수 있는지 공부를 더 해봐야겠다.

문제풀이는 조만간 [ctf](/ctf) 포스트에 올려놓을 예정이다. 

## **References**

+ <https://learn.dreamhack.io/51#3>
+ <https://cce.cstec.kr/>
+ <https://raw.githubusercontent.com/torvalds/linux/master/scripts/extract-vmlinux>
