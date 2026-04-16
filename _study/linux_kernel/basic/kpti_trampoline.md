---
layout: simple
title: KPTI (Kernel Page Table Isolation)
---

## **1.What is KPTI?**

`KPTI(Kernel Page Table Isolation)` 이란 커널 공간과 사용자 공간을 메모리 상에서 분리를 하는 것으로, `Meltdown` 및 `spectre`와 같은 cpu cache side channel attack 을 방지 하기 위해 고안된 보호기법이다. 

`KPTI` 를 적용한 상태에선 커널 공간과 사용자 공간이 서로 격리되어있기 때문에 사용자 공간이 커널 공간에 접근 할 수 없다. `KPTI` 가 적용 되지 않아도 가상 메모리 등으로 커널 공간과 사용자 공간은 서로 분리되어 있지만, `KPTI`를 적용한 상테에서는 사용자 공간과 커널 공간의 페이지 테이블을 분리한다. 

근본 자체가 사이드 체널을 막기 위해 고안된 방어기법이라 Memory Curruption 공격에는 크게 영향을 주지 않지만, 커널 영역에서 익스플로잇을 진행 한 후, 다시 유저 모드로 복귀할 때 `Segmentation Fault`를 발생시킨다. 

따라서 커널 모드에서 익스플로잇을 한 후 사용자 모드에서 쉘을 얻기 위해 우리는 `KPTI` 로 야기되는 문제를 해결 할 필요가 있다. 

## **2. KPTI trampoline**

`KPTI` 로 발생하는 문제를 우회하는 방법에는 크게 두가지가 있다. 

첫번째로는 `signal handler`를 사용하여 `Segmentation Fault` 가 발생했을 때 원하는 함수를 실행할 수 있게 하는 방법이다. 

하지만 굳이 핸들러를 쓰지 않고 한번에 유저모드로 복귀시키는 방법이 있는데, 이것이 바로 **KPTI trampoline** 기법이다. 

우리가 일반적으로 API 를 이용하여 커널에 작동을 요청 할 때, 사용자 모드에서 커널 모드로 상태가 바뀐 후, 다시 커널 모드에서 사용자 모드로 돌아온다. 

이때 커널 모드에서 사용자 모드로 돌아오는데 사용하는 커널의 기능이 바로 `swapgs_restore_regs_and_return_to_usermode` 이다. 

KPTI trampoline은 해당 기능의 context switch 기능을 사용해 유저모드로 안전하게 돌아오는 기법이다.


## **3. Simple dive to swapgs_restore_regs_and_return_to_usermode**

`swapgs_restore_regs_and_return_to_usermode`는 사용자 모드일 때의 레지스터를 복원하고, 사용자 모드로 복귀시키는 함수이다. 

실제 `swapgs_restore_regs_and_return_to_usermode`를 살펴보자.

+ `swapgs_restore_regs_and_return_to_usermode` 심볼 확인

```bash
~ # cat /proc/kallsyms | grep swapgs_restore_regs_and_return_to_usermode
ffffffff820010f0 T swapgs_restore_regs_and_return_to_usermode
```

+ `swapgs_restore_regs_and_return_to_usermode` 루틴

```c
0xffffffff820010f0:	jmp    0xffffffff8200110b
0xffffffff820010f2:	mov    ecx,0x48
0xffffffff820010f7:	mov    rdx,QWORD PTR gs:0x1fbc8
0xffffffff82001100:	and    edx,0xfffffffe
0xffffffff82001103:	mov    eax,edx
0xffffffff82001105:	shr    rdx,0x20
0xffffffff82001109:	wrmsr  
0xffffffff8200110b:	nop    DWORD PTR [rax+rax*1+0x0]
0xffffffff82001110:	pop    r15
0xffffffff82001112:	pop    r14
0xffffffff82001114:	pop    r13
0xffffffff82001116:	pop    r12
0xffffffff82001118:	pop    rbp
0xffffffff82001119:	pop    rbx
0xffffffff8200111a:	pop    r11
0xffffffff8200111c:	pop    r10
0xffffffff8200111e:	pop    r9
0xffffffff82001120:	pop    r8
0xffffffff82001122:	pop    rax
0xffffffff82001123:	pop    rcx
0xffffffff82001124:	pop    rdx
0xffffffff82001125:	pop    rsi
0xffffffff82001126:	mov    rdi,rsp
0xffffffff82001129:	mov    rsp,QWORD PTR gs:0x6004
0xffffffff82001132:	push   QWORD PTR [rdi+0x30]
0xffffffff82001135:	push   QWORD PTR [rdi+0x28]
0xffffffff82001138:	push   QWORD PTR [rdi+0x20]
0xffffffff8200113b:	push   QWORD PTR [rdi+0x18]
0xffffffff8200113e:	push   QWORD PTR [rdi+0x10]
0xffffffff82001141:	push   QWORD PTR [rdi]
0xffffffff82001143:	push   rax
0xffffffff82001144:	xchg   ax,ax
0xffffffff82001146:	mov    rdi,cr3
0xffffffff82001149:	jmp    0xffffffff8200117f
0xffffffff8200114b:	mov    rax,rdi
0xffffffff8200114e:	and    rdi,0x7ff
0xffffffff82001155:	bt     QWORD PTR gs:0x30b96,rdi
0xffffffff8200115f:	jae    0xffffffff82001170
0xffffffff82001161:	btr    QWORD PTR gs:0x30b96,rdi
0xffffffff8200116b:	mov    rdi,rax
0xffffffff8200116e:	jmp    0xffffffff82001178
0xffffffff82001170:	mov    rdi,rax
0xffffffff82001173:	bts    rdi,0x3f
0xffffffff82001178:	or     rdi,0x800
0xffffffff8200117f:	or     rdi,0x1000
0xffffffff82001186:	mov    cr3,rdi
0xffffffff82001189:	pop    rax
0xffffffff8200118a:	pop    rdi
0xffffffff8200118b:	swapgs 
0xffffffff8200118e:	jmp    0xffffffff820011b0
0xffffffff82001190:	pop    r15
0xffffffff82001192:	pop    r14
0xffffffff82001194:	pop    r13
0xffffffff82001196:	pop    r12
0xffffffff82001198:	pop    rbp
0xffffffff82001199:	pop    rbx
0xffffffff8200119a:	pop    r11
0xffffffff8200119c:	pop    r10
0xffffffff8200119e:	pop    r9
0xffffffff820011a0:	pop    r8
0xffffffff820011a2:	pop    rax
0xffffffff820011a3:	pop    rcx
0xffffffff820011a4:	pop    rdx
0xffffffff820011a5:	pop    rsi
0xffffffff820011a6:	pop    rdi
0xffffffff820011a7:	add    rsp,0x8
0xffffffff820011ab:	jmp    0xffffffff820011b0
0xffffffff820011b0:	test   BYTE PTR [rsp+0x20],0x4
0xffffffff820011b5:	jne    0xffffffff820011b9
0xffffffff820011b7:	iretq  
```

여기서 자세히 봐야할 곳은 다음과 같다. 

### **1. swicth CR3 register's bit**

```c
0xffffffff82001144:	xchg   ax,ax
0xffffffff82001146:	mov    rdi,cr3
0xffffffff82001149:	jmp    0xffffffff8200117f
...
0xffffffff8200117f:	or     rdi,0x1000
0xffffffff82001186:	mov    cr3,rdi
```

코드를 보면 cr3 레지스터 값의 12번째 비트를 1로 만드는 과정을 볼 수 있다. 

cr3 레지스터의 12번째 비트는 페이지 암호화와 관련되어있다. 

12번째 비트가 0이면 암호화 되지 않은 페이지에 접근한다는 의미이고, 12번째비트가 1이면 암호화 되어있는 페이지 테이블에 접근한다는 뜻이다. 

디버깅을 통해 보면 12번째 비트가 0인 것을 확인할 수 있는데, 암호화 되지 않은 커널 페이지에서 암호화된 영역인 유저 페이지에 들어간다는 뜻 인것 같다. 

### **2. swapgs**

```c
0xffffffff8200118b:	swapgs 
0xffffffff8200118e:	jmp    0xffffffff820011b0
```

`swapgs` 명령어는 커널 영역 값으로 되어있는 GS 레지스터를 유저영역의 값으로 바꾸는 명령어이다. 값이 유저영역이면 커널영역으로 바꾼다.  
유저모드로 돌아가기 전에 커널영역으로 되어있는 GS 레지스터 값을 유저영역의 값으로 바꾼다. 

### **3. iretq**

```c
0xffffffff820011b7:	iretq  
```

`iretq` 명령어는 인터럽트나 예외 처리 후에 사용되는 명령어로, 인터럽트나 예외 처리 루틴을 종료하고 원래의 프로그램 실행 흐름으로 돌아가게 해주는 명령어이다. 

`iretq` 명령어는 스텍에서 다음과 같은 요소를 차례대로 pop 한다. 

+ pop rip
+ pop cs 
+ pop rflags
+ pop rsp 
+ pop ss

다음 과 같은 레지스터의 값들은 기존에 유저영역에 있을 때 당시의 레지스터값이다. 

다음과 같이 3개의 과정이 모두 끝나면 커널 영역에서 사용자 영역으로 전환되면서 유저영역으로 안전하게 돌아 올 수 있다. 

## **Conclusion (Jump to KPTI trampoline)**

즉, 커널영역에서 ROP 가 가능한 상황 등 rsp 및 stack 을 조작할 수 있게 된다면, `swapgs_restore_regs_and_return_to_usermode`의 일부로 점프하여 기존 사용자 영역의 레지스터 값을 설정하여 안전하게 사용자 영역으로 돌아올 수 있게 할 수 있다. 

위의 첫번째 과정인 switch cr3 register 가 시작하는 코드로 점프하고, 스텍에 `rip`, `cs`, `rflags`, `rsp`, `ss` 값을 넣어준다면 안전하게 돌아올 수 있을 것이다. 

## **References**

+ <https://lkmidas.github.io/posts/20210128-linux-kernel-pwn-part-2/#adding-kpti>
+ <https://breaking-bits.gitbook.io/breaking-bits/exploit-development/linux-kernel-exploit-development/kernel-page-table-isolation-kpti>
+ <https://en.wikipedia.org/wiki/Control_register>
