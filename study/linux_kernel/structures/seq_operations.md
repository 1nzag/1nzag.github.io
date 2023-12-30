---
layout: simple
title: struct seq_operations
---

## **Intro**

`seq_operations` 구조체는 `seq_read()` 함수에서 사용하는 구조체이다. 만약 UAF 를 `kmalloc-32` 에서 수행 할 수 있을 때, 해당 구조체가 유용하게 쓰일 수 있다. 

## **Simple dive in to seq_read()**

`seq_read()` 함수는 single_open 을 사용하는 파일에 read 를 하면서 발생하는 함수이다. 대표적으로 `/proc/self/stat` 함수를 open 하고, 거기에 read 할 때 트리거 된다. 

`seq_read()` 함수가 어떤 함수인지 한번 파해쳐 보자. 

+ fs/seq_file.c

```c
/**
 *	seq_read -	->read() method for sequential files.
 *	@file: the file to read from
 *	@buf: the buffer to read to
 *	@size: the maximum number of bytes to read
 *	@ppos: the current position in the file
 *
 *	Ready-made ->f_op->read()
 */
ssize_t seq_read(struct file *file, char __user *buf, size_t size, loff_t *ppos)
{
	struct iovec iov = { .iov_base = buf, .iov_len = size};
	struct kiocb kiocb;
	struct iov_iter iter;
	ssize_t ret;

	init_sync_kiocb(&kiocb, file);
	iov_iter_init(&iter, READ, &iov, 1, size);

	kiocb.ki_pos = *ppos;
	ret = seq_read_iter(&kiocb, &iter);
	*ppos = kiocb.ki_pos;
	return ret;
}
EXPORT_SYMBOL(seq_read);
```
먼저 `init_sync_kiocb()` 함수를 통해 `struct kiocb` 구조체로 선언된 `kiocb` 변수를 초기화 한다. 

+ include/linux/fs.h

```c
static inline void init_sync_kiocb(struct kiocb *kiocb, struct file *filp)
{
	*kiocb = (struct kiocb) {
		.ki_filp = filp,
		.ki_flags = iocb_flags(filp),
		.ki_hint = ki_hint_validate(file_write_hint(filp)),
		.ki_ioprio = get_current_ioprio(),
	};
}
```

`kiocb` 구조체를 다음과 같이 초기화 하는 것을 볼 수 있는데, 다른건 제치고 우리는 `ki_filp` 만 볼 것이다. 

`ki_filp` 필드는 인자로 주어진 `filp` 인 것을 알 수 있는데, 이는 `seq_read()` 함수의 인자로 주어졌던 `struct file` 구조체인 `file`이다. 

`struct file` 구조체를 살펴보자.

+ include/linux/fs.h

```c
struct file {
	union {
		struct llist_node	fu_llist;
		struct rcu_head 	fu_rcuhead;
	} f_u;
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	/*
	 * Protects f_ep, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	enum rw_hint		f_write_hint;
	atomic_long_t		f_count;
	unsigned int 		f_flags;
	fmode_t			f_mode;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct hlist_head	*f_ep;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
	errseq_t		f_wb_err;
	errseq_t		f_sb_err; /* for syncfs */
} __randomize_layout
  __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */
```

우리는 여기서 `private_data` 필드를 볼 것이다. 그 이유는 아래 다음 코드에서 볼 수 있다.

`seq_read()` 함수는 `seq_read_iter()` 함수를 통해 실질적인 read 를 수행한다. 

+ fs/seq_file.c

```c
/*
 * Ready-made ->f_op->read_iter()
 */
ssize_t seq_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
	struct seq_file *m = iocb->ki_filp->private_data;
	size_t copied = 0;
	size_t n;
	void *p;
	int err = 0;

	if (!iov_iter_count(iter))
		return 0;

	mutex_lock(&m->lock);

	/*
	 * if request is to read from zero offset, reset iterator to first
	 * record as it might have been already advanced by previous requests
	 */
	if (iocb->ki_pos == 0) {
		m->index = 0;
		m->count = 0;
	}

	/* Don't assume ki_pos is where we left it */
	if (unlikely(iocb->ki_pos != m->read_pos)) {
		while ((err = traverse(m, iocb->ki_pos)) == -EAGAIN)
			;
		if (err) {
			/* With prejudice... */
			m->read_pos = 0;
			m->index = 0;
			m->count = 0;
			goto Done;
		} else {
			m->read_pos = iocb->ki_pos;
		}
	}

	/* grab buffer if we didn't have one */
	if (!m->buf) {
		m->buf = seq_buf_alloc(m->size = PAGE_SIZE);
		if (!m->buf)
			goto Enomem;
	}
	// something left in the buffer - copy it out first
	if (m->count) {
		n = copy_to_iter(m->buf + m->from, m->count, iter);
		m->count -= n;
		m->from += n;
		copied += n;
		if (m->count)	// hadn't managed to copy everything
			goto Done;
	}
	// get a non-empty record in the buffer
	m->from = 0;
	p = m->op->start(m, &m->index);
	while (1) {
		err = PTR_ERR(p);
		if (!p || IS_ERR(p))	// EOF or an error
			break;
		err = m->op->show(m, p);
		if (err < 0)		// hard error
			break;
		if (unlikely(err))	// ->show() says "skip it"
			m->count = 0;
		if (unlikely(!m->count)) { // empty record
			p = m->op->next(m, p, &m->index);
			continue;
		}
		if (!seq_has_overflowed(m)) // got it
			goto Fill;
		// need a bigger buffer
		m->op->stop(m, p);
		kvfree(m->buf);
		m->count = 0;
		m->buf = seq_buf_alloc(m->size <<= 1);
		if (!m->buf)
			goto Enomem;
		p = m->op->start(m, &m->index);
	}
	// EOF or an error
	m->op->stop(m, p);
	m->count = 0;
	goto Done;
Fill:
	// one non-empty record is in the buffer; if they want more,
	// try to fit more in, but in any case we need to advance
	// the iterator once for every record shown.
	while (1) {
		size_t offs = m->count;
		loff_t pos = m->index;

		p = m->op->next(m, p, &m->index);
		if (pos == m->index) {
			pr_info_ratelimited("buggy .next function %ps did not update position index\n",
					    m->op->next);
			m->index++;
		}
		if (!p || IS_ERR(p))	// no next record for us
			break;
		if (m->count >= iov_iter_count(iter))
			break;
		err = m->op->show(m, p);
		if (err > 0) {		// ->show() says "skip it"
			m->count = offs;
		} else if (err || seq_has_overflowed(m)) {
			m->count = offs;
			break;
		}
	}
	m->op->stop(m, p);
	n = copy_to_iter(m->buf, m->count, iter);
	copied += n;
	m->count -= n;
	m->from = n;
Done:
	if (unlikely(!copied)) {
		copied = m->count ? -EFAULT : err;
	} else {
		iocb->ki_pos += copied;
		m->read_pos += copied;
	}
	mutex_unlock(&m->lock);
	return copied;
Enomem:
	err = -ENOMEM;
	goto Done;
}
EXPORT_SYMBOL(seq_read_iter);
```

코드를 보면 `struct seq_operations` 의 구조체를 `m` 이라는 이름의 변수로 할당 하고 `iocb->ki_filp->private_data` 로 초기화를 하는 것을 알 수 있다. 

`struct seq_file` 의 구조는 다음과 같다. 

+ include/linux/seq_file.h

```c
struct seq_file {
	char *buf;
	size_t size;
	size_t from;
	size_t count;
	size_t pad_until;
	loff_t index;
	loff_t read_pos;
	struct mutex lock;
	const struct seq_operations *op;
	int poll_event;
	const struct file *file;
	void *private;
};
```

보다시피 우리가 봐야 할 `struct seq_operaitons` 구조체가 있는 필드가 `op` 라는 이름으로 지정되어 있고, 아래부분에 파일 읽기를 수행 할 때 `op` 의 맴버 함수를 호출 하는 것을 알 수 있다. 


그러면 해당 `private_data`는 어디서 초기화가 되는 것일까?

그것은 바로 read 를 하기 전 파일을 open 할 떄 초기화가 된다.

+ fs/seq_file.c

```c
int single_open(struct file *file, int (*show)(struct seq_file *, void *),
		void *data)
{
	struct seq_operations *op = kmalloc(sizeof(*op), GFP_KERNEL_ACCOUNT);
	int res = -ENOMEM;

	if (op) {
		op->start = single_start;
		op->next = single_next;
		op->stop = single_stop;
		op->show = show;
		res = seq_open(file, op);
		if (!res)
			((struct seq_file *)file->private_data)->private = data;
		else
			kfree(op);
	}
	return res;
}
EXPORT_SYMBOL(single_open);

/*...*/

int seq_open(struct file *file, const struct seq_operations *op)
{
	struct seq_file *p;

	WARN_ON(file->private_data);

	p = kmem_cache_zalloc(seq_file_cache, GFP_KERNEL);
	if (!p)
		return -ENOMEM;

	file->private_data = p;

	mutex_init(&p->lock);
	p->op = op;

	// No refcounting: the lifetime of 'p' is constrained
	// to the lifetime of the file.
	p->file = file;

	/*
	 * seq_files support lseek() and pread().  They do not implement
	 * write() at all, but we clear FMODE_PWRITE here for historical
	 * reasons.
	 *
	 * If a client of seq_files a) implements file.write() and b) wishes to
	 * support pwrite() then that client will need to implement its own
	 * file.open() which calls seq_open() and then sets FMODE_PWRITE.
	 */
	file->f_mode &= ~FMODE_PWRITE;
	return 0;
}
EXPORT_SYMBOL(seq_open);
```

앞서 설명한 대로 `single_open()` 을 통해 파일을 open 할 때 `op`는 `struct seq_operations` 구조체로 할당이 되고, `kmalloc`을 통해 힙영역에 해당 구조체를 할당 하는 것을 알 수 있다. 그 후 해당 `op` 의 맴버를 여러 함수 포인터로 초기화 하는 것을 알 수 있다. 

그리고 `seq_open()` 함수를 호출 하는데, 해당 함수는 `struct file`의 `private_data` 필드를 해당 `op`로 초기화 하는 것을 볼 수 있다. 

그럼 이제 다시 `seq_read()`의 코드를 보자. 

```c
/* ... */
	p = m->op->start(m, &m->index);
	while (1) {
		err = PTR_ERR(p);
		if (!p || IS_ERR(p))	// EOF or an error
			break;
		err = m->op->show(m, p);
		if (err < 0)		// hard error
			break;
		if (unlikely(err))	// ->show() says "skip it"
			m->count = 0;
		if (unlikely(!m->count)) { // empty record
			p = m->op->next(m, p, &m->index);
			continue;
		}
		if (!seq_has_overflowed(m)) // got it
			goto Fill;
		// need a bigger buffer
		m->op->stop(m, p);
		kvfree(m->buf);
		m->count = 0;
		m->buf = seq_buf_alloc(m->size <<= 1);
		if (!m->buf)
			goto Enomem;
		p = m->op->start(m, &m->index);
	}
	// EOF or an error
	m->op->stop(m, p);
	m->count = 0;
	goto Done;
/* ... */
```

`seq_read()` 함수에서는 해당 구조체의 함수 포인터를 호출하는 것을 볼 수 있다.

## **Structure of seq_operations**

그럼 이런 함수 포인터를 담고 있는 `struct seq_opertaions` 구조체의 구조를 보자. 

+ include/linux/seq_file.h

```c
struct seq_operations {
	void * (*start) (struct seq_file *m, loff_t *pos);
	void (*stop) (struct seq_file *m, void *v);
	void * (*next) (struct seq_file *m, void *v, loff_t *pos);
	int (*show) (struct seq_file *m, void *v);
};
```
총 4개의 함수포인터가 있고, 크기는 64비트에서 32 바이트라는 것을 알 수 있다. 즉, 이 구조체는 `kmalloc-32`에 할당이 되는 것을 알 수 있다.


## **Conclusion**

만약 우리가 `kmalloc-32` 에서 OOB 나 UAF를 일으킬 수 있다고 가정하면, 우리는 `single_open()` 을 통해 `kmalloc-32`에 `seq_operations` 데이터를 스프레이 하고, 취약점을 트리거 하여 해당 함수 포인터를 변조한 뒤, `seq_read()` 함수를 호출하여 RIP를 변조 할 수 있을것이다.

즉 이 포스트에서 소개하는 구조체는 `kmalloc-32` 익스플로잇에 활용할 수 있는 유용한 구조체중 하나이다. 

연습문제를 푸는데, `kmalloc-32`에서 익스하는 방법을 찾아보다 알게 된 구조체이다. 하지만, 내가 푸는 문제에는 쓸모가 없었다 ;;

나중에 쓸 기회가 되면 이 포스트에 사용하는 방법까지 자세하게 포스트할 생각이다. 

## **References

+ <https://github.com/esanfelix/r2con2019-ctf-kernel/blob/master/solution/README.md>
+ <https://blog.kakaocdn.net/dn/cw8D0r/btqFtKLLWSj/adwRN8BVxfOZwByjd0qN0K/%5B%EC%A0%95%EC%9E%AC%EC%98%81%5D%20ASIS%20CTF%202020%20shared_house.html?attach=1&knm=tfile.html>
