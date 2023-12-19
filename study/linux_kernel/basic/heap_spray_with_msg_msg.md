---
layout: simple
title:  Kernel heap spray with msg_msg
---

## **Intro**

커널 익스플로잇, 특히 SLUB 을 이용해 익스플로잇을 하게 되면 힙 스프레이를 해야 되는 경우가 생길 것이다. 

이 포스트는 `msg_msg`를 이용해 커널 힙 스프레이를 하는 방법에 대해 설명한다. 

## **What is msg_msg**

`msg_msg` 구조체는 `sys_msgsnd` syscall 에서 사용하는 메시지 구조체이다. `sys_msgsnd` 함수는 프로세스 통신간에 사용하는 메시지 큐에 메시지를 보내는데 사용되는 시스템 콜이다. 


`sys_msgsnd` syscall 을 한번 살펴보자.

```c
int sys_msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
```

+ **msqid**: 메시지를 보낼 메시지 큐의 id
+ **msgp**: 메시지 포인터
    + 보통 다음과 같은 구조체 포인터를 사용함.
        ```c
        struct msgbuf {
            long mtype;
            char mtext[1];
        }
        ```
+ **msgsz**: `mtext` 필드의 길이
+ **msgflg**: 메시지 전송 옵션

## **Dive in to sys_msgsnd**

직접 `sys_msgsnd`가 어떻게 `msg_msg` 구조체를 사용하는지 살펴보자.

`sys_msgsnd` 시스템 콜을 호출하면, 커널 내부에서는 `ksys_msgsnd()` 함수가 호출된다. 

+ ipc/msg.c

```c
long ksys_msgsnd(int msqid, struct msgbuf __user *msgp, size_t msgsz,
		 int msgflg)
{
	long mtype;

	if (get_user(mtype, &msgp->mtype))
		return -EFAULT;
	return do_msgsnd(msqid, mtype, msgp->mtext, msgsz, msgflg);
}

/*...*/

static long do_msgsnd(int msqid, long mtype, void __user *mtext,
		size_t msgsz, int msgflg)
{
	struct msg_queue *msq;
	struct msg_msg *msg;
	int err;
	struct ipc_namespace *ns;
	DEFINE_WAKE_Q(wake_q);

	ns = current->nsproxy->ipc_ns;

	if (msgsz > ns->msg_ctlmax || (long) msgsz < 0 || msqid < 0)
		return -EINVAL;
	if (mtype < 1)
		return -EINVAL;

	msg = load_msg(mtext, msgsz);
	if (IS_ERR(msg))
		return PTR_ERR(msg);

	msg->m_type = mtype;
	msg->m_ts = msgsz;

	rcu_read_lock();
	msq = msq_obtain_object_check(ns, msqid);
	if (IS_ERR(msq)) {
		err = PTR_ERR(msq);
		goto out_unlock1;
	}

	ipc_lock_object(&msq->q_perm);

	for (;;) {
		struct msg_sender s;

		err = -EACCES;
		if (ipcperms(ns, &msq->q_perm, S_IWUGO))
			goto out_unlock0;

		/* raced with RMID? */
		if (!ipc_valid_object(&msq->q_perm)) {
			err = -EIDRM;
			goto out_unlock0;
		}

		err = security_msg_queue_msgsnd(&msq->q_perm, msg, msgflg);
		if (err)
			goto out_unlock0;

		if (msg_fits_inqueue(msq, msgsz))
			break;

		/* queue full, wait: */
		if (msgflg & IPC_NOWAIT) {
			err = -EAGAIN;
			goto out_unlock0;
		}

		/* enqueue the sender and prepare to block */
		ss_add(msq, &s, msgsz);

		if (!ipc_rcu_getref(&msq->q_perm)) {
			err = -EIDRM;
			goto out_unlock0;
		}

		ipc_unlock_object(&msq->q_perm);
		rcu_read_unlock();
		schedule();

		rcu_read_lock();
		ipc_lock_object(&msq->q_perm);

		ipc_rcu_putref(&msq->q_perm, msg_rcu_free);
		/* raced with RMID? */
		if (!ipc_valid_object(&msq->q_perm)) {
			err = -EIDRM;
			goto out_unlock0;
		}
		ss_del(&s);

		if (signal_pending(current)) {
			err = -ERESTARTNOHAND;
			goto out_unlock0;
		}

	}
}
```

`ksys_msgsnd()` 는 `do_msgsnd()` 함수를 호출하고, `do_msgsnd()` 함수는 `mtext`영역의 데이터와 `msgsz` 를 받아 `load_msg()` 함수에 넘겨준다. 

+ ipc/msgutil.c

```c
struct msg_msg *load_msg(const void __user *src, size_t len)
{
	struct msg_msg *msg;
	struct msg_msgseg *seg;
	int err = -EFAULT;
	size_t alen;

	msg = alloc_msg(len);
	if (msg == NULL)
		return ERR_PTR(-ENOMEM);

	alen = min(len, DATALEN_MSG);
	if (copy_from_user(msg + 1, src, alen))
		goto out_err;

	for (seg = msg->next; seg != NULL; seg = seg->next) {
		len -= alen;
		src = (char __user *)src + alen;
		alen = min(len, DATALEN_SEG);
		if (copy_from_user(seg + 1, src, alen))
			goto out_err;
	}

	err = security_msg_msg_alloc(msg);
	if (err)
		goto out_err;

	return msg;

out_err:
	free_msg(msg);
	return ERR_PTR(err);
}
```

`load_msg()` 함수는 `struct msg_msg` 구조체를 리턴하는 것을 알 수 있다!

먼저 `alloc_msg()` 함수를 통해 `len` 만큼 크기의 `msg`를 할당받고, `copy_from_user()` 함수를 통해 `sizeof(struct msg_msg)` 뒤에 유저가 보낸 `src`를 복사한다. 

+ ipc/msgutil.c

```c
static struct msg_msg *alloc_msg(size_t len)
{
	struct msg_msg *msg;
	struct msg_msgseg **pseg;
	size_t alen;

	alen = min(len, DATALEN_MSG);
	msg = kmalloc(sizeof(*msg) + alen, GFP_KERNEL_ACCOUNT);
	if (msg == NULL)
		return NULL;

	msg->next = NULL;
	msg->security = NULL;

	len -= alen;
	pseg = &msg->next;
	while (len > 0) {
		struct msg_msgseg *seg;

		cond_resched();

		alen = min(len, DATALEN_SEG);
		seg = kmalloc(sizeof(*seg) + alen, GFP_KERNEL_ACCOUNT);
		if (seg == NULL)
			goto out_err;
		*pseg = seg;
		seg->next = NULL;
		pseg = &seg->next;
		len -= alen;
	}

	return msg;

out_err:
	free_msg(msg);
	return NULL;
}
```

`alloc_msg()` 함수는 `kmalloc()` 함수를 이용해 `sizeof(*msg) + alen` 만큼의 공간을 할당 하는 것을 볼 수 있다.

즉, `sys_msgsnd` 시스템 콜을 사용하여 우리가 지정한 버퍼를 `msg_msg` 구조체를 이용해 커널의 힙 부분에 쓸 수 있다. 

## **Why Use msg_msg?**

유저가 힙 영역에 메모리를 allocate 하는 방법은 간접적으로 이루어져야 한다. 유저영역에서 직접 `kmalloc()`을 호출 할 수는 없기 때문이다. 

이 때 유저가 원하는 데이터를 원하는 크기만큼 힙 영역에 할당시킬 수 있는 방법은 여러가지가 있을것이다. 

하지만 대부분은 **익스플로잇을 하는 동안 커널 힙 영역에 메모리 할당을 유지하는 조건**을 달성하기가 힘들다. 

`sys_msgsnd`를 사용하여 `msg_msg` 구조체를 메시지 큐에 넣는다면, `sys_msgrcv`를 통해 메시지 큐에서 해당 메모리를 팝 하지 않는 이상, 커널 힙 메모리에 메모리가 계속 할당된 채로 유지한다! 

따라서 `sys_msgsnd` 를 이용해 `msg_msg` 구조체를 할당하는 방법은 커널 힙 스프레이에 유용하다.

## **How to use msg_msg**

그럼 직접 `msg_msg`를 사용해 heap spray를 해보자.

`sys_msgsnd` 시스템 콜은 `msgsnd` API 를 통해 할 수 있다. 

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <sys/ipc.h>
#include <sys/msg.h>

#define SPRAY_SIZE

struct msgbuf {
    long mtype;
    char mtext[1];
}

int msg_msg_spray(int spray_count)
{
    int spray_queue;
    struct *msgbuf msg_buf = (struct msgbuf*)malloc(SPRAY_SIZE);
    int i;
    
    if (spray_queue = msgget(IPC_PRIVATE, 0666 | IPC_CREAT) == -1)
    {
        perror("create queue failed");
        return;
    }

    for(i = 0; i < spray_count; i++)
    {
        if (msgsnd(spray_queue, msg_buf, SPRAY_SIZE - sizeof(msgbuf.mtype), 0) < 0)
        {
            perror("msgsnd failed");
        }
    }
}

int main(void)
{
    msg_msg_spray(0x10000);
}
```

먼저 `msgget()` 함수를 이용해 메시지 큐를 생성하고, 해당 큐에 `msgsnd()` 함수를 통해 원하는 만큼의 횟수만큼 메시지를 보내면 된다. 

## **Limitation of msg_msg**

해당 기법에는 단점이 하나 존재한다. 위의 `do_msgsnd()` 함수의 리뷰에서 봤듯이, 할당되는 메모리가 온전히 유저가 지정한 영역이 아니라 `sizeof(msg_msg)`의 크기만큼의 헤더를 가진다는 것이다. 

```
---------------------

   <msg_msg header>
        48byte

---------------------

 <user defined data>


---------------------
```

## **Avaliable size with msg_msg**

`msg_msg`를 통해 할당을 할 때에는 최대 크기가 있다. 다시 `alloc_msg()` 함수의 코드를 보자.

+ ipc/msgutil.c

```c
static struct msg_msg *alloc_msg(size_t len)
{
	struct msg_msg *msg;
	struct msg_msgseg **pseg;
	size_t alen;

	alen = min(len, DATALEN_MSG);
	msg = kmalloc(sizeof(*msg) + alen, GFP_KERNEL_ACCOUNT);
	if (msg == NULL)
		return NULL;

	msg->next = NULL;
	msg->security = NULL;

	len -= alen;
	pseg = &msg->next;
	while (len > 0) {
		struct msg_msgseg *seg;

		cond_resched();

		alen = min(len, DATALEN_SEG);
		seg = kmalloc(sizeof(*seg) + alen, GFP_KERNEL_ACCOUNT);
		if (seg == NULL)
			goto out_err;
		*pseg = seg;
		seg->next = NULL;
		pseg = &seg->next;
		len -= alen;
	}

	return msg;

out_err:
	free_msg(msg);
	return NULL;
}
```

유저가 보낸 메시지가 `DATALEN_MSG` 보다 크면, `DATALEN_MSG` 길이 단위로 끊어서 할당을 하는 것을 볼 수 있다. 

`DATALEN_MSG`의 크기는 다음과 같다. 

+ ipc/msgutil.c

```c
#define DATALEN_MSG	((size_t)PAGE_SIZE-sizeof(struct msg_msg))
```

페이지 사이즈에 `msg_msg` 사이즈를 뺀 만큼인 것을 알 수 있다. 보통 커널의 페이지 사이즈는 4096 바이트이므로, 평균적으로 최대 4048 만큼의 유저 지정 데이터를 한번 할당 할 수 있다는 것을 알 수 있다. 

## **Conclusion**

원데이를 공부하다 `msg_msg`를 이용하여 힙 스프레이를 한다는 말이 정말 많이 나와서 공부하게 됐다. 앞으로 다른 원데이들을 공부하면서 PoC 를 짜는데에 이해가 되면서 도움이 될 수 있을거라 생각한다. 

다음엔 위의 기법을 쓰는 원데이 익스플로잇을 리뷰할 생각이다. 

`msg_msg`에 대해서 각 구조체 필드의 의미와 자료 저장 원리를 알고 싶다면 [이 포스트를 보자](/study/linux_kernel/basic/dive_in_to_msg_msg)

## **References**

+ <https://duasynt.com/blog/linux-kernel-heap-spray>
+ <https://github.com/Bonfee/CVE-2022-0995/blob/main/exploit.c>