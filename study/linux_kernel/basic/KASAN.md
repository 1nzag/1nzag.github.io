---
layout: simple
title: KASAN (kernel Address SANitizer)
---

## **1. What is KASAN**

KASAN (Kernel Address SANitizer) 란 kernel 동적 메모리 오류 탐지기로, UAF (Use-After-Free) 및 OOB(Out-Of-Bounds) 오류를 탐지할 수 있는 도구다.

kernel 에서 메모리 오류 탐지를 해야 하기 때문에 사용하려면 KASAN 도구를 kernel 빌드 시에 같이 넣어서 빌드해주어야 한다. 

현재는 `x86_64` 와 `arm64` 아키텍처만 지원한다. 


## **2. Build kernel with KASAN**

일반적으로 커널을 빌드할 때 다음과 같은 옵션을 추가해 주면 KASAN 을 kernel 과 같이 빌드 할 수 있다. 

```bash
CONFIG_KASAN = y
```

또는 `make menuconfig` 를 통해 KASAN을 빌드 할 수 있다. 

+ `make menuconfig` 명령어 사용
+ 구성 메뉴에서 "Kernel hacking" 섹션 이동
+ "Memory Debugging" 옵션 이동
+ "Use 'kernel address sanitizer' to detect use-after-free and out-of-bounds bugs" 옵션 활성화

kasan 이 제대로 같이 빌드되었는지 다음과 같이 확인 할 수 있다.

+ 생성된 kernel image 에서 kasan 관련 심볼 확인
```bash
user@user:~/repo$ strings vmlinux | grep kasan
0kasan: CONFIG_KASAN_INLINE enabled
0kasan: GPF could be caused by NULL-ptr deref or user memory access
6kasan: KernelAddressSanitizer initialized
/goldfish/mm/kasan/common.c
/goldfish/mm/kasan/init.c
kasan: bad access detected
/goldfish/mm/kasan/generic.c
kasan_kmalloc
kasan_check_write
kasan_check_read
kasan_restore_multi_shot
kasan_save_enable_multi_shot
kasan_multi_shot
/out/kasan/goldfish
kasan_depth
kasan_early_init
kasan_unpoison_task_stack
kasan_init
kasan_unpoison_stack_above_sp_to
kasan_module_alloc
kasan_populate_pud
kasan_populate_early_shadow
kasan_early_shadow_p4d
kasan_mem_to_shadow
kasan_map_early_shadow
kasan_die_handler
kasan_populate_shadow
kasan_populate_p4d
kasan_early_shadow_page
kasan_early_p4d_populate
kasan_early_shadow_pmd
kasan_die_notifier
kasan_early_shadow_pte
kasan_early_shadow_pud
kasan_populate_pgd
kasan_populate_pmd
kasan_poison_kfree
kasan_poison_element
kasan_unpoison_element
kasan_unpoison_slab
kasan_alloc_pages
kasan_free_pages
kasan_info
kasan_cache
[...]
```
+ 커널 부팅 시 kasan 메시지 확인
```
[    0.000000] kasan: KernelAddressSanitizer initialized
```

## **3. KASAN functions**

config 파일을 이용하면 KASAN 이 가지고 있는 여러 기능들을 선택적으로만 추가 할 수 있다. 

KASAN 의 기능들은 다음과 같다. 

+ **CONFIG_KASAN**: Kernel Address Sanitizer 활성화 옵션
+ **CONFIG_KASAN_INLINE**: KASAN의 인라인 모드 활성화 옵션
    + KASAN 체크를 코드에 직접 삽입하는 방식
+ **CONFIG_TEST_KASAN**: KASAN 테스트 코드를 활성화 옵션
+ **CONFIG_KCOV**: 커널 코드 커버리지 측정 도구 활성화 옵션
+ **CONFIG_SLUB, CONFIG_SLUB_DEBUG, CONFIG_SLUB_DEBUG_ON**: SLUB 할당자와 관련된 디버깅 옵션
+ **CONFIG_SLUB_DEBUG_PANIC_ON**: SLUB 디버거가 패닉 상태를 유발하는 설정
+ **CONFIG_KASAN_OUTLINE**: KASAN의 아웃라인(Outline) 모드 
    + 이 모드는 체크 로직을 별도의 함수로 분리한다.
+ **CONFIG_KERNEL_LZ4, CONFIG_RANDOMIZE_BASE**: 커널 압축 방식과 주소 공간 레이아웃 무작위화와 관련된 설정

이러한 기능을 선택적으로 추가하는 방법은 다음과 같다. 

다음은 android10 (goldfish) 를 빌드할 때 KASAN을 적용하는 config 파일 샘플이다. 

```bash
ARCH=x86_64
BRANCH=kasan

CC=clang
CLANG_PREBUILT_BIN=prebuilts-master/clang/host/linux-x86/clang-r377782b/bin
BUILDTOOLS_PREBUILT_BIN=build/build-tools/path/linux-x86
CLANG_TRIPLE=x86_64-linux-gnu-
CROSS_COMPILE=x86_64-linux-androidkernel-
LINUX_GCC_CROSS_COMPILE_PREBUILTS_BIN=prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.9/bin

KERNEL_DIR=goldfish
EXTRA_CMDS=''
STOP_SHIP_TRACEPRINTK=1

FILES="
arch/x86/boot/bzImage
vmlinux
System.map
"

DEFCONFIG=x86_64_ranchu_defconfig
POST_DEFCONFIG_CMDS="check_defconfig && update_kasan_config"

function update_kasan_config() {
    ${KERNEL_DIR}/scripts/config --file ${OUT_DIR}/.config \
         -e CONFIG_KASAN \
         -e CONFIG_KASAN_INLINE \
         -e CONFIG_TEST_KASAN \
         -e CONFIG_KCOV \
         -e CONFIG_SLUB \
         -e CONFIG_SLUB_DEBUG \
         -e CONFIG_SLUB_DEBUG_ON \
         -d CONFIG_SLUB_DEBUG_PANIC_ON \
         -d CONFIG_KASAN_OUTLINE \
         -d CONFIG_KERNEL_LZ4 \
         -d CONFIG_RANDOMIZE_BASE
    (cd ${OUT_DIR} && \
     make O=${OUT_DIR} $archsubarch CROSS_COMPILE=${CROSS_COMPILE} olddefconfig)
}
```

`function update_kasan_config` 를 살펴보면 KASAN의 각 기능을 설정할 수 있는 것을 볼 수 있다. 

각 기능에 `-e` 옵션을 붙이면 활성화 한다는 의미이고, `-d` 옵션을 붙이면 비활성화 한다는 의미이다. 

해당 config 파일을 빌드할때 다음과 같이 적용해 주면 KASAN의 선택적 기능을 같이 빌드 할 수 있다. 

```bash
BUILD_CONFIG=[kasan config file path] build/build.sh
```

## **Conclusion**

KASAN을 이용하면 kernel 디버깅을 할 때 많은 도움을 받을 수 있다. 

KASAN 의 기능을 공부하고, 실제 연구에 적용하는 것에 익숙해진다면 커널 버그를 찾는 효율이 증가할 것으로 생각한다. 

## **References**

+ https://cloudfuzz.github.io/android-kernel-exploitation/chapters/vulnerability-trigger.html#build-kernel-with-kasan
+ https://www.kernel.org/doc/html/v4.14/dev-tools/kasan.html