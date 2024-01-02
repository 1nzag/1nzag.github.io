---
layout: simple
title: struct pipe_buffer
---

## **Intro**

`pipe_buffer` 구조체는 `sys_pipe` syscall 호출 시 사용되는 구조체이다. 만약 힙 취약점을 `kmalloc-1024` 에서 수행할 수 있을 때 유용하게 쓸 수 있는 victim object 이다. 

## **Simple dive in to sys_pipe**

그럼 `sys_pipe` syscall 을 호출 할 때 커널 내에서 어떻게 `pipe_buffer`를 할당하는지 확인해보자. 

`sys_pipe` syscall 을 호출하면 다음과 같은 코드가 호출된다.

+ fs/pipe.c

```c

/*
 * sys_pipe() is the normal C calling standard for creating
 * a pipe. It's not the way Unix traditionally does this, though.
 */
static int do_pipe2(int __user *fildes, int flags)
{
	struct file *files[2];
	int fd[2];
	int error;

	error = __do_pipe_flags(fd, files, flags);
	if (!error) {
		if (unlikely(copy_to_user(fildes, fd, sizeof(fd)))) {
			fput(files[0]);
			fput(files[1]);
			put_unused_fd(fd[0]);
			put_unused_fd(fd[1]);
			error = -EFAULT;
		} else {
			fd_install(fd[0], files[0]);
			fd_install(fd[1], files[1]);
		}
	}
	return error;
}

/* ... */

SYSCALL_DEFINE1(pipe, int __user *, fildes)
{
	return do_pipe2(fildes, 0);
}
```

`pipe` syscall 핸들러는 `do_pipe2()` 함수를 호출하고, `do_pipe2()` 함수는 `__do_pipe_flags()` 함수를 호출한다. 

+ fs/pipe.c

```c
static int __do_pipe_flags(int *fd, struct file **files, int flags)
{
	int error;
	int fdw, fdr;

	if (flags & ~(O_CLOEXEC | O_NONBLOCK | O_DIRECT | O_NOTIFICATION_PIPE))
		return -EINVAL;

	error = create_pipe_files(files, flags);
	if (error)
		return error;

	error = get_unused_fd_flags(flags);
	if (error < 0)
		goto err_read_pipe;
	fdr = error;

	error = get_unused_fd_flags(flags);
	if (error < 0)
		goto err_fdr;
	fdw = error;

	audit_fd_pair(fdr, fdw);
	fd[0] = fdr;
	fd[1] = fdw;
	return 0;

 err_fdr:
	put_unused_fd(fdr);
 err_read_pipe:
	fput(files[0]);
	fput(files[1]);
	return error;
}
```

`__do_pipe_flags()` 함수는 `create_pipe_files()` 함수를 통해 pipe 를 생성한다. 

+ fs/pipe.c

```c
int create_pipe_files(struct file **res, int flags)
{
	struct inode *inode = get_pipe_inode();
	struct file *f;
	int error;

	if (!inode)
		return -ENFILE;

	if (flags & O_NOTIFICATION_PIPE) {
		error = watch_queue_init(inode->i_pipe);
		if (error) {
			free_pipe_info(inode->i_pipe);
			iput(inode);
			return error;
		}
	}

	f = alloc_file_pseudo(inode, pipe_mnt, "",
				O_WRONLY | (flags & (O_NONBLOCK | O_DIRECT)),
				&pipefifo_fops);
	if (IS_ERR(f)) {
		free_pipe_info(inode->i_pipe);
		iput(inode);
		return PTR_ERR(f);
	}

	f->private_data = inode->i_pipe;

	res[0] = alloc_file_clone(f, O_RDONLY | (flags & O_NONBLOCK),
				  &pipefifo_fops);
	if (IS_ERR(res[0])) {
		put_pipe_info(inode, inode->i_pipe);
		fput(f);
		return PTR_ERR(res[0]);
	}
	res[0]->private_data = inode->i_pipe;
	res[1] = f;
	stream_open(inode, res[0]);
	stream_open(inode, res[1]);
	return 0;
}
```

`create_pipe_files()` 함수는 `get_pipe_inode()` 함수를 호출하여 `inode` 구조체를 초기화 한다. 

+ fs/pipe.c

```c
static struct inode * get_pipe_inode(void)
{
	struct inode *inode = new_inode_pseudo(pipe_mnt->mnt_sb);
	struct pipe_inode_info *pipe;

	if (!inode)
		goto fail_inode;

	inode->i_ino = get_next_ino();

	pipe = alloc_pipe_info();
	if (!pipe)
		goto fail_iput;

	inode->i_pipe = pipe;
	pipe->files = 2;
	pipe->readers = pipe->writers = 1;
	inode->i_fop = &pipefifo_fops;

	/*
	 * Mark the inode dirty from the very beginning,
	 * that way it will never be moved to the dirty
	 * list because "mark_inode_dirty()" will think
	 * that it already _is_ on the dirty list.
	 */
	inode->i_state = I_DIRTY;
	inode->i_mode = S_IFIFO | S_IRUSR | S_IWUSR;
	inode->i_uid = current_fsuid();
	inode->i_gid = current_fsgid();
	inode->i_atime = inode->i_mtime = inode->i_ctime = current_time(inode);

	return inode;

fail_iput:
	iput(inode);

fail_inode:
	return NULL;
}
```

그리고 `get_pipe_inode()` 함수는 `alloc_pipe_info()` 함수를 통해 `struct pipe_inode_info`를 초기화 하는 것을 알 수 있다. 

`alloc_pipe_info()` 함수를 보자.

+ fs/pipe.c

```c
struct pipe_inode_info *alloc_pipe_info(void)
{
	struct pipe_inode_info *pipe;
	unsigned long pipe_bufs = PIPE_DEF_BUFFERS;
	struct user_struct *user = get_current_user();
	unsigned long user_bufs;
	unsigned int max_size = READ_ONCE(pipe_max_size);

	pipe = kzalloc(sizeof(struct pipe_inode_info), GFP_KERNEL_ACCOUNT);
	if (pipe == NULL)
		goto out_free_uid;

	if (pipe_bufs * PAGE_SIZE > max_size && !capable(CAP_SYS_RESOURCE))
		pipe_bufs = max_size >> PAGE_SHIFT;

	user_bufs = account_pipe_buffers(user, 0, pipe_bufs);

	if (too_many_pipe_buffers_soft(user_bufs) && pipe_is_unprivileged_user()) {
		user_bufs = account_pipe_buffers(user, pipe_bufs, PIPE_MIN_DEF_BUFFERS);
		pipe_bufs = PIPE_MIN_DEF_BUFFERS;
	}

	if (too_many_pipe_buffers_hard(user_bufs) && pipe_is_unprivileged_user())
		goto out_revert_acct;

	pipe->bufs = kcalloc(pipe_bufs, sizeof(struct pipe_buffer),
			     GFP_KERNEL_ACCOUNT);

	if (pipe->bufs) {
		init_waitqueue_head(&pipe->rd_wait);
		init_waitqueue_head(&pipe->wr_wait);
		pipe->r_counter = pipe->w_counter = 1;
		pipe->max_usage = pipe_bufs;
		pipe->ring_size = pipe_bufs;
		pipe->nr_accounted = pipe_bufs;
		pipe->user = user;
		mutex_init(&pipe->mutex);
		return pipe;
	}

out_revert_acct:
	(void) account_pipe_buffers(user, pipe_bufs, 0);
	kfree(pipe);
out_free_uid:
	free_uid(user);
	return NULL;
}
```

여기서 `struct pipe_inode_info` 를 먼저 `kzalloc()`으로 할당하고, 그 이후에 `pipe->bufs` 를 `kcalloc()`으로 할당하는 것을 볼 수 있다. 

이 때 `kcalloc` 으로 `struct pipe_buffer` 의 크기 * `PIPE_DEF_BUFFERS` 만큼 하는 것을 알 수 있다. 

`struct pipe_buffer`의 크기는 40 이고, `PIPE_DEF_BUFFERS`는 16으로 정의되어 있으므로 , 요청하는 할당 크기는 40 * 16 = 640이므로 `kmalloc-1024`에 해당 구조체가 할당이 되는 것을 볼 수 있다. 

+ include/linux/pipe_fs_i.h

```c
struct pipe_buffer {
	struct page *page;
	unsigned int offset, len;
	const struct pipe_buf_operations *ops;
	unsigned int flags;
	unsigned long private;
};

/* ... */

#define PIPE_DEF_BUFFERS	16
```

그럼 `struct pipe_buffer`에서 어떤 필드를 우리가 활용할 수 있을까. 

우선 `struct pipe_buffer`가 어디서 어떻게 초기화 되는지를 볼 필요가 있다. 

`pipe_buffer`는 생성된 pipe 에 write 할때 초기화 된다. pipe에 write를 하게 되면 `pipe_write()` 함수가 호출된다. 

+ fs/pipe.c

```c
const struct file_operations pipefifo_fops = {
	.open		= fifo_open,
	.llseek		= no_llseek,
	.read_iter	= pipe_read,
	.write_iter	= pipe_write,
	.poll		= pipe_poll,
	.unlocked_ioctl	= pipe_ioctl,
	.release	= pipe_release,
	.fasync		= pipe_fasync,
	.splice_write	= iter_file_splice_write,
};
```

그럼 `pipe_write()` 함수에서 어떻게 `pipe_buffer`를 초기화 하는 지 보자

+ fs/pipe.c

```c
static ssize_t
pipe_write(struct kiocb *iocb, struct iov_iter *from)
{
	struct file *filp = iocb->ki_filp;
	struct pipe_inode_info *pipe = filp->private_data;
	unsigned int head;
	ssize_t ret = 0;
	size_t total_len = iov_iter_count(from);
	ssize_t chars;
	bool was_empty = false;
	bool wake_next_writer = false;

	/* Null write succeeds. */
	if (unlikely(total_len == 0))
		return 0;

	__pipe_lock(pipe);

	if (!pipe->readers) {
		send_sig(SIGPIPE, current, 0);
		ret = -EPIPE;
		goto out;
	}

#ifdef CONFIG_WATCH_QUEUE
	if (pipe->watch_queue) {
		ret = -EXDEV;
		goto out;
	}
#endif

	/*
	 * If it wasn't empty we try to merge new data into
	 * the last buffer.
	 *
	 * That naturally merges small writes, but it also
	 * page-aligns the rest of the writes for large writes
	 * spanning multiple pages.
	 */
	head = pipe->head;
	was_empty = pipe_empty(head, pipe->tail);
	chars = total_len & (PAGE_SIZE-1);
	if (chars && !was_empty) {
		unsigned int mask = pipe->ring_size - 1;
		struct pipe_buffer *buf = &pipe->bufs[(head - 1) & mask];
		int offset = buf->offset + buf->len;

		if ((buf->flags & PIPE_BUF_FLAG_CAN_MERGE) &&
		    offset + chars <= PAGE_SIZE) {
			ret = pipe_buf_confirm(pipe, buf);
			if (ret)
				goto out;

			ret = copy_page_from_iter(buf->page, offset, chars, from);
			if (unlikely(ret < chars)) {
				ret = -EFAULT;
				goto out;
			}

			buf->len += ret;
			if (!iov_iter_count(from))
				goto out;
		}
	}

	for (;;) {
		if (!pipe->readers) {
			send_sig(SIGPIPE, current, 0);
			if (!ret)
				ret = -EPIPE;
			break;
		}

		head = pipe->head;
		if (!pipe_full(head, pipe->tail, pipe->max_usage)) {
			unsigned int mask = pipe->ring_size - 1;
			struct pipe_buffer *buf = &pipe->bufs[head & mask];
			struct page *page = pipe->tmp_page;
			int copied;

			if (!page) {
				page = alloc_page(GFP_HIGHUSER | __GFP_ACCOUNT);
				if (unlikely(!page)) {
					ret = ret ? : -ENOMEM;
					break;
				}
				pipe->tmp_page = page;
			}

			/* Allocate a slot in the ring in advance and attach an
			 * empty buffer.  If we fault or otherwise fail to use
			 * it, either the reader will consume it or it'll still
			 * be there for the next write.
			 */
			spin_lock_irq(&pipe->rd_wait.lock);

			head = pipe->head;
			if (pipe_full(head, pipe->tail, pipe->max_usage)) {
				spin_unlock_irq(&pipe->rd_wait.lock);
				continue;
			}

			pipe->head = head + 1;
			spin_unlock_irq(&pipe->rd_wait.lock);

			/* Insert it into the buffer array */
			buf = &pipe->bufs[head & mask];
			buf->page = page;
			buf->ops = &anon_pipe_buf_ops;
			buf->offset = 0;
			buf->len = 0;
			if (is_packetized(filp))
				buf->flags = PIPE_BUF_FLAG_PACKET;
			else
				buf->flags = PIPE_BUF_FLAG_CAN_MERGE;
			pipe->tmp_page = NULL;

			copied = copy_page_from_iter(page, 0, PAGE_SIZE, from);
			if (unlikely(copied < PAGE_SIZE && iov_iter_count(from))) {
				if (!ret)
					ret = -EFAULT;
				break;
			}
			ret += copied;
			buf->offset = 0;
			buf->len = copied;

			if (!iov_iter_count(from))
				break;
		}

		if (!pipe_full(head, pipe->tail, pipe->max_usage))
			continue;

		/* Wait for buffer space to become available. */
		if (filp->f_flags & O_NONBLOCK) {
			if (!ret)
				ret = -EAGAIN;
			break;
		}
		if (signal_pending(current)) {
			if (!ret)
				ret = -ERESTARTSYS;
			break;
		}

		/*
		 * We're going to release the pipe lock and wait for more
		 * space. We wake up any readers if necessary, and then
		 * after waiting we need to re-check whether the pipe
		 * become empty while we dropped the lock.
		 */
		__pipe_unlock(pipe);
		if (was_empty)
			wake_up_interruptible_sync_poll(&pipe->rd_wait, EPOLLIN | EPOLLRDNORM);
		kill_fasync(&pipe->fasync_readers, SIGIO, POLL_IN);
		wait_event_interruptible_exclusive(pipe->wr_wait, pipe_writable(pipe));
		__pipe_lock(pipe);
		was_empty = pipe_empty(pipe->head, pipe->tail);
		wake_next_writer = true;
	}
out:
	if (pipe_full(pipe->head, pipe->tail, pipe->max_usage))
		wake_next_writer = false;
	__pipe_unlock(pipe);

	/*
	 * If we do do a wakeup event, we do a 'sync' wakeup, because we
	 * want the reader to start processing things asap, rather than
	 * leave the data pending.
	 *
	 * This is particularly important for small writes, because of
	 * how (for example) the GNU make jobserver uses small writes to
	 * wake up pending jobs
	 *
	 * Epoll nonsensically wants a wakeup whether the pipe
	 * was already empty or not.
	 */
	if (was_empty || pipe->poll_usage)
		wake_up_interruptible_sync_poll(&pipe->rd_wait, EPOLLIN | EPOLLRDNORM);
	kill_fasync(&pipe->fasync_readers, SIGIO, POLL_IN);
	if (wake_next_writer)
		wake_up_interruptible_sync_poll(&pipe->wr_wait, EPOLLOUT | EPOLLWRNORM);
	if (ret > 0 && sb_start_write_trylock(file_inode(filp)->i_sb)) {
		int err = file_update_time(filp);
		if (err)
			ret = err;
		sb_end_write(file_inode(filp)->i_sb);
	}
	return ret;
}
```

`pipe_write()` 함수는 다음과 같이 `pipe_buffer->ops`를 `anon_pipe_buf_ops` 라는 상수형 구조체로 초기화 되는 것을 알 수 있다. 


```c
    /* ... */
			buf->ops = &anon_pipe_buf_ops;
    /* ... */
```

그럼 `anon_pipe_buf_ops` 구조체를 살펴보자.

+ fs/pipe.c

```c
static const struct pipe_buf_operations anon_pipe_buf_ops = {
	.release	= anon_pipe_buf_release,
	.try_steal	= anon_pipe_buf_try_steal,
	.get		= generic_pipe_buf_get,
};
```

각각의 필드에 함수 포인터가 구성되어있는 것을 알 수 있다. 

또한 `anon_pipe_buf_ops` 역시 상수이기 때문에 커널 영역의 `.data`영역에 위치한다. 

즉, 우리는 `pipe_buffer` 구조체의 `ops` 필드를 릭 해 커널 베이스를 알 수 있고, `anon_pipe_buf_ops` 로 되어있는 필드를 조작할 수 있는 경우에 함수를 후킹할 수 있다. (RIP 를 컨트롤 할 수 있다). 

## **How to allocate pipe_buffer**

`struct pipe_buffer` 를 커널 힙 상에 할당하는 방법은 매우 간단하다. `sys_pipe` 를 호출하여 pipe 를 생성하고, 거기에 write 하면 끝이다. 

아래 예제코드는 `kmalloc-1024` 에 `struct pipe_buffer` 를 스프레이 하는 코드의 예시이다. 

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    
    int pipe_fds[1024][2];
    int i;

    for (i = 0; i < 1024; i++)
    {
        if (pipe(pipe_fds[i]) < 0)
        {
            perror("failed create pipe\n");
            exit(0);
        }

        if (write(pipe_fds[i][1], "A", 1) < 0)
        {
            perror("failed write to pipe\n");
            exit(0);
        }
    }

    return 0;
}
```

## **Conclusion**

CVE-2022-0995 취약점을 공부하면서 알게 된 구조체이다. kmalloc-1024 를 이용해 익스플로잇을 짤 때 언젠가 한번 써먹을 수 있지 않을까 한다. 

물론 `kmalloc-1024` 를 이용해 익스플로잇을 할 때 이 구조체를 쓰는 것이 무조건 옳다는 것은 아니지만, 커널 베이스를 릭할 수 있음과 동시에, 함수 포인터를 후킹할 수 있다는 점에서 굉장히 매력적인 구조체임에는 틀림없는것 같다. 

## **References**

+ <https://google.github.io/security-research/pocs/linux/cve-2021-22555/writeup.html>

