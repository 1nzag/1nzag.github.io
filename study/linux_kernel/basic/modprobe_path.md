---
layout: simple
title: Kernel Exploit Technique - modprobe_path
---

## **1. Why use modprobe?**

`modprobe`란 커널 모듈들을 추가하거나 제거하기 위해 사용하는 사용자 영역의 프로그램이다. 

`modprobe`는 사용되지 않는 모듈이 존재 할 때, 사용할 수 있도록 로드 해 주는 역할 역시 해줄 수 있다. 

`modprobe`를 사용하는 대표적인 것은 `execve` 이다. 

`execve`의 `modprobe`를 사용하는 과정은 다음과 같다.

1. `execve` 로 특정 바이너리 실행을 요구할 때, 먼저 해당 바이너리에 맞는 바이너리 로더를 찾는다. 
    + 예를 들어, 헤더에 `ELF` 가 있다면, elf 실행파일의 로더를 커널에서 찾는다. 
2. 만약 바이너리가 식별되지 않는 바이너리 일 때, 적합한 바이너리 로더가 있는지 찾는다. 
    + 바이너리 로더 이름은 `binfmt-AABBCCDD` 형식이며, 뒤에 붙은 `AABBCCDD`는 바이너리 앞 4바이트 헤더의 hex값이다. 
3. 해당 바이너리 로더가 있으면, 커널은 `modprobe`를 통해 바이너리 로더를 로드하려고 시도한다.

여기서 `modprobe`가 실행하는 절대 경로가 담긴 변수가 바로 `modprobe_path` 이다. 

이 때 실행할 때의 권한은 `root` 권한이다. 

## **2. Dive in to execve**

그럼 커널의 `execve` 가 어떻게 사용되는지 직접 코드를 통해 살펴보자.

아래 코드는 `execve` syscall이 호출되었을때, 진행되는 루틴이다. 

+ fs/exec.c

```c

SYSCALL_DEFINE3(execve,
		const char __user *, filename,
		const char __user *const __user *, argv,
		const char __user *const __user *, envp)
{
	return do_execve(getname(filename), argv, envp);
}

/*...*/

static int do_execve(struct filename *filename,
	const char __user *const __user *__argv,
	const char __user *const __user *__envp)
{
	struct user_arg_ptr argv = { .ptr.native = __argv };
	struct user_arg_ptr envp = { .ptr.native = __envp };
	return do_execveat_common(AT_FDCWD, filename, argv, envp, 0);
}

/*...*/

tatic int do_execveat_common(int fd, struct filename *filename,
			      struct user_arg_ptr argv,
			      struct user_arg_ptr envp,
			      int flags)
{
	struct linux_binprm *bprm;
	int retval;

	if (IS_ERR(filename))
		return PTR_ERR(filename);

	/*
	 * We move the actual failure in case of RLIMIT_NPROC excess from
	 * set*uid() to execve() because too many poorly written programs
	 * don't check setuid() return code.  Here we additionally recheck
	 * whether NPROC limit is still exceeded.
	 */
	if ((current->flags & PF_NPROC_EXCEEDED) &&
	    is_rlimit_overlimit(current_ucounts(), UCOUNT_RLIMIT_NPROC, rlimit(RLIMIT_NPROC))) {
		retval = -EAGAIN;
		goto out_ret;
	}

	/* We're below the limit (still or again), so we don't want to make
	 * further execve() calls fail. */
	current->flags &= ~PF_NPROC_EXCEEDED;

	bprm = alloc_bprm(fd, filename);
	if (IS_ERR(bprm)) {
		retval = PTR_ERR(bprm);
		goto out_ret;
	}

	retval = count(argv, MAX_ARG_STRINGS);
	if (retval == 0)
		pr_warn_once("process '%s' launched '%s' with NULL argv: empty string added\n",
			     current->comm, bprm->filename);
	if (retval < 0)
		goto out_free;
	bprm->argc = retval;

	retval = count(envp, MAX_ARG_STRINGS);
	if (retval < 0)
		goto out_free;
	bprm->envc = retval;

	retval = bprm_stack_limits(bprm);
	if (retval < 0)
		goto out_free;

	retval = copy_string_kernel(bprm->filename, bprm);
	if (retval < 0)
		goto out_free;
	bprm->exec = bprm->p;

	retval = copy_strings(bprm->envc, envp, bprm);
	if (retval < 0)
		goto out_free;

	retval = copy_strings(bprm->argc, argv, bprm);
	if (retval < 0)
		goto out_free;

	/*
	 * When argv is empty, add an empty string ("") as argv[0] to
	 * ensure confused userspace programs that start processing
	 * from argv[1] won't end up walking envp. See also
	 * bprm_stack_limits().
	 */
	if (bprm->argc == 0) {
		retval = copy_string_kernel("", bprm);
		if (retval < 0)
			goto out_free;
		bprm->argc = 1;
	}

	retval = bprm_execve(bprm, fd, filename, flags);
out_free:
	free_bprm(bprm);

out_ret:
	putname(filename);
	return retval;
}

```

먼저 `execve` syscall 이 호출되면, `do_execve()` 함수를 호출하고, `do_execve()` 함수는 그대로 `do_execveat_common()` 함수를 호출한다. 

`do_exeveat_common()` 함수는 `linux_binprm` 구조체인 `bprm`을 생성, 구조체 값을 초기화 한 후 `bprm_excve()`를 호출한다.

`linux_binprm` 구조체는 리눅스 커널 내에서 실행 가능한 파일의 정보를 담고 있는 구조체이다. 

+ include/linux/binfmts.h

```c
struct linux_binprm {
#ifdef CONFIG_MMU
	struct vm_area_struct *vma;
	unsigned long vma_pages;
#else
# define MAX_ARG_PAGES	32
	struct page *page[MAX_ARG_PAGES];
#endif
	struct mm_struct *mm;
	unsigned long p; /* current top of mem */
	unsigned long argmin; /* rlimit marker for copy_strings() */
	unsigned int
		/* Should an execfd be passed to userspace? */
		have_execfd:1,

		/* Use the creds of a script (see binfmt_misc) */
		execfd_creds:1,
		/*
		 * Set by bprm_creds_for_exec hook to indicate a
		 * privilege-gaining exec has happened. Used to set
		 * AT_SECURE auxv for glibc.
		 */
		secureexec:1,
		/*
		 * Set when errors can no longer be returned to the
		 * original userspace.
		 */
		point_of_no_return:1;
	struct file *executable; /* Executable to pass to the interpreter */
	struct file *interpreter;
	struct file *file;
	struct cred *cred;	/* new credentials */
	int unsafe;		/* how unsafe this exec is (mask of LSM_UNSAFE_*) */
	unsigned int per_clear;	/* bits to clear in current->personality */
	int argc, envc;
	const char *filename;	/* Name of binary as seen by procps */
	const char *interp;	/* Name of the binary really executed. Most
				   of the time same as filename, but could be
				   different for binfmt_{misc,script} */
	const char *fdpath;	/* generated filename for execveat */
	unsigned interp_flags;
	int execfd;		/* File descriptor of the executable */
	unsigned long loader, exec;

	struct rlimit rlim_stack; /* Saved RLIMIT_STACK used during exec. */

	char buf[BINPRM_BUF_SIZE];
} __randomize_layout;
```

구조체에 대한 자세한 설명은 지금은 딱히 필요 없으므로 넘어가자. 

그럼 다음으로 호출되는 `bprm_execve()` 함수를 살펴보자.

+ fs/exec.c

```c
static int bprm_execve(struct linux_binprm *bprm,
		       int fd, struct filename *filename, int flags)
{
	struct file *file;
	int retval;

	retval = prepare_bprm_creds(bprm);
	if (retval)
		return retval;

	/*
	 * Check for unsafe execution states before exec_binprm(), which
	 * will call back into begin_new_exec(), into bprm_creds_from_file(),
	 * where setuid-ness is evaluated.
	 */
	check_unsafe_exec(bprm);
	current->in_execve = 1;
	sched_mm_cid_before_execve(current);

	file = do_open_execat(fd, filename, flags);
	retval = PTR_ERR(file);
	if (IS_ERR(file))
		goto out_unmark;

	sched_exec();

	bprm->file = file;
	/*
	 * Record that a name derived from an O_CLOEXEC fd will be
	 * inaccessible after exec.  This allows the code in exec to
	 * choose to fail when the executable is not mmaped into the
	 * interpreter and an open file descriptor is not passed to
	 * the interpreter.  This makes for a better user experience
	 * than having the interpreter start and then immediately fail
	 * when it finds the executable is inaccessible.
	 */
	if (bprm->fdpath && get_close_on_exec(fd))
		bprm->interp_flags |= BINPRM_FLAGS_PATH_INACCESSIBLE;

	/* Set the unchanging part of bprm->cred */
	retval = security_bprm_creds_for_exec(bprm);
	if (retval)
		goto out;

	retval = exec_binprm(bprm);
	if (retval < 0)
		goto out;

	sched_mm_cid_after_execve(current);
	/* execve succeeded */
	current->fs->in_exec = 0;
	current->in_execve = 0;
	rseq_execve(current);
	user_events_execve(current);
	acct_update_integrals(current);
	task_numa_free(current, false);
	return retval;

out:
	/*
	 * If past the point of no return ensure the code never
	 * returns to the userspace process.  Use an existing fatal
	 * signal if present otherwise terminate the process with
	 * SIGSEGV.
	 */
	if (bprm->point_of_no_return && !fatal_signal_pending(current))
		force_fatal_sig(SIGSEGV);

out_unmark:
	sched_mm_cid_after_execve(current);
	current->fs->in_exec = 0;
	current->in_execve = 0;

	return retval;
}
```

먼저 bprm 구조체를 이용해 프로그램을 실행시키기 전에 권한을 부여하고, 안전한 검사인지 체크, 그리고 커널 스케줄링을 업데이트 한다. 

스케줄링과 관련된 부분은 해당 포스트에서 중요한 부분이 아니니 넘어가자

그렇게 프로그램을 실행시키기 위한 과정을 마치고, `exec_binprm()` 함수를 호출한다. 

다음 함수를 살펴보자 

+ fs/exec.c

```c
static int exec_binprm(struct linux_binprm *bprm)
{
	pid_t old_pid, old_vpid;
	int ret, depth;

	/* Need to fetch pid before load_binary changes it */
	old_pid = current->pid;
	rcu_read_lock();
	old_vpid = task_pid_nr_ns(current, task_active_pid_ns(current->parent));
	rcu_read_unlock();

	/* This allows 4 levels of binfmt rewrites before failing hard. */
	for (depth = 0;; depth++) {
		struct file *exec;
		if (depth > 5)
			return -ELOOP;

		ret = search_binary_handler(bprm);
		if (ret < 0)
			return ret;
		if (!bprm->interpreter)
			break;

		exec = bprm->file;
		bprm->file = bprm->interpreter;
		bprm->interpreter = NULL;

		allow_write_access(exec);
		if (unlikely(bprm->have_execfd)) {
			if (bprm->executable) {
				fput(exec);
				return -ENOEXEC;
			}
			bprm->executable = exec;
		} else
			fput(exec);
	}

	audit_bprm(bprm);
	trace_sched_process_exec(current, old_pid, bprm);
	ptrace_event(PTRACE_EVENT_EXEC, old_vpid);
	proc_exec_connector(current);
	return 0;
}
```
먼저 `task_pid_nr_ns()` 함수의 호출로 고유한 pid 를 할당받고, 그 다음 일련의 루프문을 거치고, 실행 파일의 로딩 및 이벤트를 처리한다. 

일련의 루프문에서는 어떠한 행동을 할까. 

바로 주어진 `bprm` 구조체를 처리할 수 있는 파일 핸들러를 찾는다. `search_binary_handler()`가 바로 적절한 실행 파일 핸들러를 찾는 함수이다. 위의 1절에서 설명한 2번과정 (실행 파일 로더를 찾는 과정) 이라고 볼 수 있다. 

그럼 `search_binary_handler()` 함수의 로직을 살펴보자.

+ fs/exec.c

```c
#define printable(c) (((c)=='\t') || ((c)=='\n') || (0x20<=(c) && (c)<=0x7e))

/*...*/

static int search_binary_handler(struct linux_binprm *bprm)
{
	bool need_retry = IS_ENABLED(CONFIG_MODULES);
	struct linux_binfmt *fmt;
	int retval;

	retval = prepare_binprm(bprm);
	if (retval < 0)
		return retval;

	retval = security_bprm_check(bprm);
	if (retval)
		return retval;

	retval = -ENOENT;
 retry:
	read_lock(&binfmt_lock);
	list_for_each_entry(fmt, &formats, lh) {
		if (!try_module_get(fmt->module))
			continue;
		read_unlock(&binfmt_lock);

		retval = fmt->load_binary(bprm);

		read_lock(&binfmt_lock);
		put_binfmt(fmt);
		if (bprm->point_of_no_return || (retval != -ENOEXEC)) {
			read_unlock(&binfmt_lock);
			return retval;
		}
	}
	read_unlock(&binfmt_lock);

	if (need_retry) {
		if (printable(bprm->buf[0]) && printable(bprm->buf[1]) &&
		    printable(bprm->buf[2]) && printable(bprm->buf[3]))
			return retval;
		if (request_module("binfmt-%04x", *(ushort *)(bprm->buf + 2)) < 0)
			return retval;
		need_retry = false;
		goto retry;
	}

	return retval;
}
```

여기서 우리가 집중적으로 봐야 할 곳은 다음부분이다. 

```c
if (printable(bprm->buf[0]) && printable(bprm->buf[1]) &&
    printable(bprm->buf[2]) && printable(bprm->buf[3]))
    return retval;
if (request_module("binfmt-%04x", *(ushort *)(bprm->buf + 2)) < 0)
    return retval;
```

파일의 처음 헤더 4바이트가 printable하면 바로 리턴하지만, printable하지 않으면 `request_module()` 함수를 통해 모듈을 찾는다. 이때 이름은 `binfmt-AABBCCDD` 형식이다. 

그럼 마지막으로 `request_module()` 함수가 어떻게 모듈을 찾는지 살펴보자.

+ include/linux/kmod.h

```c
int __request_module(bool wait, const char *name, ...);
#define request_module(mod...) __request_module(true, mod)

int __request_module(bool wait, const char *fmt, ...)
{
	va_list args;
	char module_name[MODULE_NAME_LEN];
	int ret, dup_ret;

	/*
	 * We don't allow synchronous module loading from async.  Module
	 * init may invoke async_synchronize_full() which will end up
	 * waiting for this task which already is waiting for the module
	 * loading to complete, leading to a deadlock.
	 */
	WARN_ON_ONCE(wait && current_is_async());

	if (!modprobe_path[0])
		return -ENOENT;

	va_start(args, fmt);
	ret = vsnprintf(module_name, MODULE_NAME_LEN, fmt, args);
	va_end(args);
	if (ret >= MODULE_NAME_LEN)
		return -ENAMETOOLONG;

	ret = security_kernel_module_request(module_name);
	if (ret)
		return ret;

	ret = down_timeout(&kmod_concurrent_max, MAX_KMOD_ALL_BUSY_TIMEOUT * HZ);
	if (ret) {
		pr_warn_ratelimited("request_module: modprobe %s cannot be processed, kmod busy with %d threads for more than %d seconds now",
				    module_name, MAX_KMOD_CONCURRENT, MAX_KMOD_ALL_BUSY_TIMEOUT);
		return ret;
	}

	trace_module_request(module_name, wait, _RET_IP_);

	if (kmod_dup_request_exists_wait(module_name, wait, &dup_ret)) {
		ret = dup_ret;
		goto out;
	}

	ret = call_modprobe(module_name, wait ? UMH_WAIT_PROC : UMH_WAIT_EXEC);

out:
	up(&kmod_concurrent_max);

	return ret;
}
EXPORT_SYMBOL(__request_module);

```

드디어 우리의 주제인 `modprobe_path`가 눈에 보였다. 실제 모듈을 실행하는 함수인 `call_modprobe()` 함수를 보자

+ fs/exec.c

```c
static int call_modprobe(char *orig_module_name, int wait)
{
	struct subprocess_info *info;
	static char *envp[] = {
		"HOME=/",
		"TERM=linux",
		"PATH=/sbin:/usr/sbin:/bin:/usr/bin",
		NULL
	};
	char *module_name;
	int ret;

	char **argv = kmalloc(sizeof(char *[5]), GFP_KERNEL);
	if (!argv)
		goto out;

	module_name = kstrdup(orig_module_name, GFP_KERNEL);
	if (!module_name)
		goto free_argv;

	argv[0] = modprobe_path;
	argv[1] = "-q";
	argv[2] = "--";
	argv[3] = module_name;	/* check free_modprobe_argv() */
	argv[4] = NULL;

	info = call_usermodehelper_setup(modprobe_path, argv, envp, GFP_KERNEL,
					 NULL, free_modprobe_argv, NULL);
	if (!info)
		goto free_module_name;

	ret = call_usermodehelper_exec(info, wait | UMH_KILLABLE);
	kmod_dup_request_announce(orig_module_name, ret);
	return ret;

free_module_name:
	kfree(module_name);
free_argv:
	kfree(argv);
out:
	kmod_dup_request_announce(orig_module_name, -ENOMEM);
	return -ENOMEM;
}
```

인자로 주어진 `module_name`과 함께 모듈의 인자를 설정해 주고 `call_usermodehelpher_setup()` 을 호출 한 후 `call_usermodehelper_exec()` 함수를 호출한다. 

그리고 `call_usermodehelper_setup()` 함수의 첫번째 인자는 `modprobe_path`인 것을 확인할 수 있다. 

+ kernel/umh.c

```c
/**
 * call_usermodehelper_setup - prepare to call a usermode helper
 * @path: path to usermode executable
 * @argv: arg vector for process
 * @envp: environment for process
 * @gfp_mask: gfp mask for memory allocation
 * @init: an init function
 * @cleanup: a cleanup function
 * @data: arbitrary context sensitive data
 *
 * Returns either %NULL on allocation failure, or a subprocess_info
 * structure.  This should be passed to call_usermodehelper_exec to
 * exec the process and free the structure.
 *
 * The init function is used to customize the helper process prior to
 * exec.  A non-zero return code causes the process to error out, exit,
 * and return the failure to the calling process
 *
 * The cleanup function is just before the subprocess_info is about to
 * be freed.  This can be used for freeing the argv and envp.  The
 * Function must be runnable in either a process context or the
 * context in which call_usermodehelper_exec is called.
 */
struct subprocess_info *call_usermodehelper_setup(const char *path, char **argv,
		char **envp, gfp_t gfp_mask,
		int (*init)(struct subprocess_info *info, struct cred *new),
		void (*cleanup)(struct subprocess_info *info),
		void *data)
{
	struct subprocess_info *sub_info;
	sub_info = kzalloc(sizeof(struct subprocess_info), gfp_mask);
	if (!sub_info)
		goto out;

	INIT_WORK(&sub_info->work, call_usermodehelper_exec_work);

#ifdef CONFIG_STATIC_USERMODEHELPER
	sub_info->path = CONFIG_STATIC_USERMODEHELPER_PATH;
#else
	sub_info->path = path;
#endif
	sub_info->argv = argv;
	sub_info->envp = envp;

	sub_info->cleanup = cleanup;
	sub_info->init = init;
	sub_info->data = data;
  out:
	return sub_info;
}
EXPORT_SYMBOL(call_usermodehelper_setup);

/*...*/

/**
 * call_usermodehelper_exec - start a usermode application
 * @sub_info: information about the subprocess
 * @wait: wait for the application to finish and return status.
 *        when UMH_NO_WAIT don't wait at all, but you get no useful error back
 *        when the program couldn't be exec'ed. This makes it safe to call
 *        from interrupt context.
 *
 * Runs a user-space application.  The application is started
 * asynchronously if wait is not set, and runs as a child of system workqueues.
 * (ie. it runs with full root capabilities and optimized affinity).
 *
 * Note: successful return value does not guarantee the helper was called at
 * all. You can't rely on sub_info->{init,cleanup} being called even for
 * UMH_WAIT_* wait modes as STATIC_USERMODEHELPER_PATH="" turns all helpers
 * into a successful no-op.
 */
int call_usermodehelper_exec(struct subprocess_info *sub_info, int wait)
{
	unsigned int state = TASK_UNINTERRUPTIBLE;
	DECLARE_COMPLETION_ONSTACK(done);
	int retval = 0;

	if (!sub_info->path) {
		call_usermodehelper_freeinfo(sub_info);
		return -EINVAL;
	}
	helper_lock();
	if (usermodehelper_disabled) {
		retval = -EBUSY;
		goto out;
	}

	/*
	 * If there is no binary for us to call, then just return and get out of
	 * here.  This allows us to set STATIC_USERMODEHELPER_PATH to "" and
	 * disable all call_usermodehelper() calls.
	 */
	if (strlen(sub_info->path) == 0)
		goto out;

	/*
	 * Set the completion pointer only if there is a waiter.
	 * This makes it possible to use umh_complete to free
	 * the data structure in case of UMH_NO_WAIT.
	 */
	sub_info->complete = (wait == UMH_NO_WAIT) ? NULL : &done;
	sub_info->wait = wait;

	queue_work(system_unbound_wq, &sub_info->work);
	if (wait == UMH_NO_WAIT)	/* task has freed sub_info */
		goto unlock;

	if (wait & UMH_FREEZABLE)
		state |= TASK_FREEZABLE;

	if (wait & UMH_KILLABLE) {
		retval = wait_for_completion_state(&done, state | TASK_KILLABLE);
		if (!retval)
			goto wait_done;

		/* umh_complete() will see NULL and free sub_info */
		if (xchg(&sub_info->complete, NULL))
			goto unlock;

		/*
		 * fallthrough; in case of -ERESTARTSYS now do uninterruptible
		 * wait_for_completion_state(). Since umh_complete() shall call
		 * complete() in a moment if xchg() above returned NULL, this
		 * uninterruptible wait_for_completion_state() will not block
		 * SIGKILL'ed processes for long.
		 */
	}
	wait_for_completion_state(&done, state);

wait_done:
	retval = sub_info->retval;
out:
	call_usermodehelper_freeinfo(sub_info);
unlock:
	helper_unlock();
	return retval;
}
EXPORT_SYMBOL(call_usermodehelper_exec);
```
정확히 `call_usermodehelpher_setup()` 과 `call_usermodehelper_exec()` 를 모두 분석하는 것은 못했다. 
 
`call_usermodehelpher_setup()` 함수의 `init` 인자는 해당 프로세스의 `creds`를 초기화 하는 역할을 하는데, 해당 함수에서는 `NULL` 을 주는 것을 확인할 수 있다. 

usermodehelpher API 의 설명을 보면 `usermodehelpher` 계열 함수는 커널이 유저스페이스 파일의 실행을 도와주는 API 로써, 실행 시킬때의 프로세스 권한의 기본값은 루트라고 되어있다. 

따라서 추정이긴 하지만 `init` 함수가 `NULL` 로 주어졌을 때, 기본값인 루트 권한으로 프로그램을 실행시킨다는 것을 유추 할 수 있다. 

지금까지 분석해온 결과를 종합하면 다음과 같다. 

1. `execve` syscall을 호출 했을 때, `linux_binprm` 구조체인 `bprm`을 생성, 구조체 값을 초기화 한 후 `bprm_excve()`를 호출
2. `search_binary_handler()` 함수를 통해 실행 파일에 적합한 프로그램 핸들러를 검색
3. 만약 파일의 첫 4바이트가 printable하지 않으면, `modprobe_path` 경로의 이름을 가진 `binfmt-AABBCCDD`포맷으로 된 프로그램 핸들러를 인자로 주어 실행
    + `modprobe_path -q -binfmt-AABBCCDD`
4. 로더가 존재한다면 해당 파일 핸들러를 `root` 권한으로 실행

## **3. How to exploit with modprobe_path**

그래서 `modprobe_path`로 어떻게 익스플로잇을 한다는 것인가?

보통 `modprobe_path`의 기본값은 `/usr/bin/modprobe`이다. 

아래에 보다시피 기본 `modprobe_path` 경로에는 `root` 외에 쓸 수 있는 권한이 없다. 

```bash
graypanda@graypanda-1nzag:~/Desktop/linux-6.6.2$ ls -al /usr/sbin/modprobe
lrwxrwxrwx 1 root root 9 Nov 25 12:03 /usr/sbin/modprobe -> /bin/kmod
graypanda@graypanda-1nzag:~/Desktop/linux-6.6.2$ ls -al /bin/kmod
-rwxr-xr-x 1 root root 170352 Aug 17  2021 /bin/kmod
```
하지만 우리가 `modprobe_path`의 경로를 덮어 쓸 수 있다면, 그 경로에 일반 사용자가 쓸 수 있는 권한의 디렉토리하면?

`root` 권한으로 우리가 작성한 임의의 프로그램을 쓸 수 있다는 것이다!

즉, 우리가 원하는 프로그램을 작성 후 그 프로그램의 경로를 `modprobe_path`에 덮어씌우고, printable하지 않은 파일을 `execve()`로 실행시키려고 시도하면, 우리는 원하는 프로그램을 루트 권한으로 실행시킬 수 있게 된다. 

## **Conclusion**

익스플로잇을 하는 방법은 어떻게 보면 단순하지만, 해당 원리는 조금 복잡했다. 

이 익스플로잇 공격 방법은 어찌보면 저번에 포스트 했던 `commit_creds()` 와 `prepare_kernel_commits()`를 이용한 방법보다 훨씬 효과적이다. 

가령 우리가 임의의 쓰기를 할 수 있는 취약점을 발견했다고 가정하자. 만약 커널에 `SMEP` 이 걸려있다면, 주로 ROP를 이용해 익스플로잇 해야 하는데, 

ROP 만으로 `commit_creds(prepare_kernel_cred(0))` 을 호출하고 정상 리턴을 하는 것은 다소 어렵다. 심지어 요즘 커널은 이 과정을 수행할 수 있는 가젯이 거의 없다!

이때 그냥 `modprobe_path`를 덮어씌우고 우리가 원하는 프로그램을 작성 후 `execve()`로 원하는 프로그램 실행을 트리거 하는게 훨씬 간편하다. 

이번 커널 코드를 분석하면서, 모르지만 넘어간 커널과 관련된 부분이 너무 많았다. 예를 들면 `bprm`이라던지, 스케줄링이라던지, `usermodehelper`라던지 정확하게 이해는 하지 못했다. 

앞으로 커널 소스를 보면서 다음 부분을 추가적으로 더 공부해야겠다는 생각이 들었다. 

추후 이 익스플로잇을 이용해 커널 익스를 하는 것을 [ctf]() 포스트에 올릴 예정이다. 

## **References**

+ <https://sam4k.com/like-techniques-modprobe_path/>
+ <https://github.com/smallkirby/kernelpwn/blob/master/technique/modprobe_path.md>