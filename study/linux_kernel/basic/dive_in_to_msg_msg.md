---
layout: simple
title: Dive into msg_msg
---

## **Intro**

[이전의 포스트에서](/study/linux_kernel/basic/heap_spray_with_msg_msg) `msg_msg` 를 이용해 힙 스프레이를 하는 방법을 포스트했다.

하지만 단지 힙 스프레이만을 통해서 내가 원하는 데이터를 넣는다고 `msg_msg`를 쓰는게 아니다. 앞서 기술했듯 `msgsnd()` 함수를 통해 힙에 데이터를 할당 했을 때, 앞의 48바이트는 `msg_msg` 구조체로 할당이 된다. 

![](/assets/img/study/dive_into_msg_msg/msg_alloc.png)

따라서 우리는 힙 스프레이를 한 후, oob 등의 취약점을 사용하기 위해서 `msg_msg` 구조체를 이용 할 수 있어야 한다. 

## **Structure of msg_msg**

먼저 `msg_msg` 구조체의 구조는 다음과 같다. 

+ include/linux/msg.h

```c
/* one msg_msg structure for each message */
struct msg_msg {
	struct list_head m_list;
	long m_type;
	size_t m_ts;		/* message text size */
	struct msg_msgseg *next;
	void *security;
	/* the actual message follows immediately */
};
```
+ include/linux/types.h

```c
struct list_head {
	struct list_head *next, *prev;
};
```

+ ipc/msgutil.c

```c
struct msg_msgseg {
	struct msg_msgseg *next;
	/* the next part of the message follows immediately */
};
```

## **msgsnd() function**

먼저 `msgsnd()` 함수를 사용하여 `msg_msg` 가 어떻게 초기화 되는지 살펴보자.

앞서 기술한 것 처럼 `sys_msgsnd` 시스템 콜을 호출 하면 `do_msg()` 함수가 호출된다. 

```c
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

	ipc_update_pid(&msq->q_lspid, task_tgid(current));
	msq->q_stime = ktime_get_real_seconds();

	if (!pipelined_send(msq, msg, &wake_q)) {
		/* no one is waiting for this message, enqueue it */
		list_add_tail(&msg->m_list, &msq->q_messages);
		msq->q_cbytes += msgsz;
		msq->q_qnum++;
		atomic_add(msgsz, &ns->msg_bytes);
		atomic_inc(&ns->msg_hdrs);
	}

	err = 0;
	msg = NULL;

out_unlock0:
	ipc_unlock_object(&msq->q_perm);
	wake_up_q(&wake_q);
out_unlock1:
	rcu_read_unlock();
	if (msg != NULL)
		free_msg(msg);
	return err;
}
```

먼저 `do_msgsnd()` 힘수는 `load_msg()` 함수를 통해 메시지 객체를 할당하고 초기화 한다. `load_msg()` 함수는 `alloc_msg()` 함수로 메시지 객체를 할당하고, 할당한 객체는 `msg_msg` 구조체와 사용자가 지정한 `m_text` 으로 구성되어 있다. 

보통 요청한 `m_text`크기가 1페이지 이상이면, `msgseg` 를 통해 최대 크기만큼 데이터를 끊어서 객체를 할당한다. 

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

/* ... */

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

이 과정을 그림으로 간단하게 표현하면 다음과 같다. 

![](/assets/img/study/dive_into_msg_msg/allocated_msg.png)
