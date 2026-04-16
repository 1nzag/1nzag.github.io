---
layout: simple
title: SMEP & SMAP
---

## **1. What is SMEP(Supervisor Mode Execution Prevention)?**

linux kenel 을 exploit 할 때 만약 `rip` 를 컨트롤 할 수 있는 기회가 생겼다고 가정하자.

아주 단순하게 생각하면 우리는 사용자공간 (userspace) 에서 `mmap` 함수 등을 통해 실행 권한이 있는 메모리를 할당하고, 그 코드로 `rip`를 점프시키면 우리가 원하는 악성 행위를 정말 쉽게 할 수 있을것이다. 

SMEP 은 이러한 익스플로잇 행위를 방지하기 위해 만들어진 보호기법이다. 

SMEP 은 intel processor 의 보안 기능으로, kernel mode 에서 userspace 의 코드 실행을 차단한다. 

즉 쉽게 표현하면 우리가 커널 드라이버에 ioctl 등의 요청을 통해 kernel mode로 진입했을 때,  userspace 의 모든 메모리 영역이 NX 가 걸렸다고 생각하면 될 것 같다. 

SMEP 은 우리가 일반적으로 NX를 우회 할 때 사용하는 ROP (Return Oriented Programming) 을 통해 우회 할 수 있다. 

kernel mode 에서도 kernel의 모든 코드는 실행 가능하기 때문에 kernelspace 상에 존재하는 코드 가젯을 통해 원하는 행위를 수행하면 되는 것이다.

## **2. What is SMAP(Supervisor Mode Access Prevention)?**

SMAP은 SMEP 보다 더 강화된 보호조치로, 이 역시 인텔 프로세서에서 지원한다. 

SMAP은 kernelspace에서 userspace 로의 접근 자체를 차단한다. 

예를 들어 kernelspace에서 `rsp` 값을 변조시켜 우리가 userspace에 `mmap`등으로 만든 가짜 스텍을 이용하여 ROP를 수행한다고 가정하자.

만약 SMEP만 적용되어있는 커널이라면 userspace 에 우리가 지정한 값이 있는 커널 코드 ROP 스텍이 있기 때문에 우리가 원하는 악성 행위를 수행하는 것이 가능하다. 

하지만 SMAP 이 적용되어 있는 상태라면 `rsp` 값 자체를 변조시키는 것이 불가능하다. userspace 로의 접근 자체가 불가능하기 때문에 `mov rsp`, `xchg rsp` 등의 방법 자체가 모두 차단되기 때문이다. 

SMAP도 우회할 방법이 없는 것은 아니다. 바로 kernel 메모리 영역에 우리가 원하는 값을 쓸 수 있으면, 해당 공간에 우리가 원하는 데이터를 쓰고 그곳으로 jump 하거나, ROP 체인을 수행하면 된다. 

대표적으로 많이 쓰이는 공간은 커널 힙 메모리 영역이다. 만약 커널 힙 스프레이가 가능한 상황이라면, 커널 힙에 ROP 스텍을 써놓고, rsp 를 조작하여 ROP 를 수행하면 된다. 

## **References**

+ <https://core-research-team.github.io/2020-05-01/memory#8-kaslr--smep--smap>
+ <https://github.com/pr0cf5/kernel-exploit-practice/blob/master/bypass-smap/README.md>

