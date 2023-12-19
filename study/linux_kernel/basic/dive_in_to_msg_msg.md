---
layout: simple
title: Dive into msg_msg
---

[이전의 포스트에서](/study/linux_kernel/basic/heap_spray_with_msg_msg) `msg_msg` 를 이용해 힙 스프레이를 하는 방법을 포스트했다.

하지만 단지 힙 스프레이만을 통해서 내가 원하는 데이터를 넣는다고 `msg_msg`를 쓰는게 아니다. 앞서 기술했듯 `msgsnd()` 함수를 통해 힙에 데이터를 할당 했을 때, 앞의 48바이트는 `msg_msg` 구조체로 할당이 된다. 

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