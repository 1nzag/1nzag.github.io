---
layout: simple
title: Kernel Exploit Technique - commit_creds & prepare_kernel_creds
---

## **1. task_struct**

linux kernel 은 스레드를 구현하는 방식으로 LWP(Light-weight Process) 를 사용한다.

이를 통해 리눅스 시스템에서 병렬 처리를 효율적으로 만들 수 있다고 한다. 

linux 는 프로세스 또는 LWP 를 나타내는 구조체로 `task_struct` 를 사용한다.

즉, task_struct 에는 linux 가 process 를 관리하는 정보를 모두 가지고 있다. 

이 때 task_struct 의 일부분을 보자 

+ include/linux/sched.h

```c
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
	/*
	 * For reasons of header soup (see current_thread_info()), this
	 * must be the first element of task_struct.
	 */
	struct thread_info		thread_info;
#endif
	/* -1 unrunnable, 0 runnable, >0 stopped: */
	volatile long			state;
/*.....*/
	/* Process credentials: */

	/* Tracer's credentials at attach: */
	const struct cred __rcu		*ptracer_cred;

	/* Objective and real subjective task credentials (COW): */
	const struct cred __rcu		*real_cred;

	/* Effective (overridable) subjective task credentials (COW): */
	const struct cred __rcu		*cred;
/*.....*/
#ifdef CONFIG_SECURITY
	/* Used by LSM modules for access restriction: */
	void				*security;
#endif

	/*
	 * New fields for task_struct should be added above here, so that
	 * they are included in the randomized portion of task_struct.
	 */
	randomized_struct_fields_end

	/* CPU-specific state of this task: */
	struct thread_struct		thread;

	/*
	 * WARNING: on x86, 'thread_struct' contains a variable-sized
	 * structure.  It *MUST* be at the end of 'task_struct'.
	 *
	 * Do not put anything below here!
	 */
};
```

이 부분에서 우리는 `task_structure` 내부에 `cred` 필드가 있는 것을 확인할 수 있다. 
이 `cred` 필드가 바로 프로세스가 가지고 있는 권한에 관련된 필드이다. 

## **2. cred**

그럼 리눅스 프로세스와관련된 `cred` 구조체에 대해서 살펴보자

+ include/linux/cred.h

```c
struct cred {
	atomic_t	usage;
#ifdef CONFIG_DEBUG_CREDENTIALS
	atomic_t	subscribers;	/* number of processes subscribed */
	void		*put_addr;
	unsigned	magic;
#define CRED_MAGIC	0x43736564
#define CRED_MAGIC_DEAD	0x44656144
#endif
	kuid_t		uid;		/* real UID of the task */
	kgid_t		gid;		/* real GID of the task */
	kuid_t		suid;		/* saved UID of the task */
	kgid_t		sgid;		/* saved GID of the task */
	kuid_t		euid;		/* effective UID of the task */
	kgid_t		egid;		/* effective GID of the task */
	kuid_t		fsuid;		/* UID for VFS ops */
	kgid_t		fsgid;		/* GID for VFS ops */
	unsigned	securebits;	/* SUID-less security management */
	kernel_cap_t	cap_inheritable; /* caps our children can inherit */
	kernel_cap_t	cap_permitted;	/* caps we're permitted */
	kernel_cap_t	cap_effective;	/* caps we can actually use */
	kernel_cap_t	cap_bset;	/* capability bounding set */
	kernel_cap_t	cap_ambient;	/* Ambient capability set */
#ifdef CONFIG_KEYS
	unsigned char	jit_keyring;	/* default keyring to attach requested
					 * keys to */
	struct key __rcu *session_keyring; /* keyring inherited over fork */
	struct key	*process_keyring; /* keyring private to this process */
	struct key	*thread_keyring; /* keyring private to this thread */
	struct key	*request_key_auth; /* assumed request_key authority */
#endif
#ifdef CONFIG_SECURITY
	void		*security;	/* subjective LSM security */
#endif
	struct user_struct *user;	/* real user ID subscription */
	struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
	struct group_info *group_info;	/* supplementary groups for euid/fsgid */
	/* RCU deletion */
	union {
		int non_rcu;			/* Can we skip RCU deletion? */
		struct rcu_head	rcu;		/* RCU deletion hook */
	};
} __randomize_layout;
```

`uid`, `gid`, `guid` 등 권한에 관련된 여러 필드 들이 있는 것을 알 수 있다. 

## **3. commit_creds**

`commit_creds` 함수는 linux에서 프로세스의 권한을 변경할 떄 커널 내부에서 호출 되는 함수이다. 

예를 들어 `setuid` 시스템 콜을 호출하면, `commit_creds` 함수를 호출하여 권한 변경이 될 수 있다. 

`commit_creds` 함수의 코드를 살펴보자

+ kernel/cred.c

```c
int commit_creds(struct cred *new)
{
	struct task_struct *task = current;
	const struct cred *old = task->real_cred;

	kdebug("commit_creds(%p(%d,%d))", new,
	       atomic_read(&new->usage),
	       read_cred_subscribers(new));

	BUG_ON(task->cred != old);
#ifdef CONFIG_DEBUG_CREDENTIALS
	BUG_ON(read_cred_subscribers(old) < 2);
	validate_creds(old);
	validate_creds(new);
#endif
	BUG_ON(atomic_read(&new->usage) < 1);

	get_cred(new); /* we will require a ref for the subj creds too */

	/* dumpability changes */
	if (!uid_eq(old->euid, new->euid) ||
	    !gid_eq(old->egid, new->egid) ||
	    !uid_eq(old->fsuid, new->fsuid) ||
	    !gid_eq(old->fsgid, new->fsgid) ||
	    !cred_cap_issubset(old, new)) {
		if (task->mm)
			set_dumpable(task->mm, suid_dumpable);
		task->pdeath_signal = 0;
		/*
		 * If a task drops privileges and becomes nondumpable,
		 * the dumpability change must become visible before
		 * the credential change; otherwise, a __ptrace_may_access()
		 * racing with this change may be able to attach to a task it
		 * shouldn't be able to attach to (as if the task had dropped
		 * privileges without becoming nondumpable).
		 * Pairs with a read barrier in __ptrace_may_access().
		 */
		smp_wmb();
	}

	/* alter the thread keyring */
	if (!uid_eq(new->fsuid, old->fsuid))
		key_fsuid_changed(task);
	if (!gid_eq(new->fsgid, old->fsgid))
		key_fsgid_changed(task);

	/* do it
	 * RLIMIT_NPROC limits on user->processes have already been checked
	 * in set_user().
	 */
	alter_cred_subscribers(new, 2);
	if (new->user != old->user)
		atomic_inc(&new->user->processes);
	rcu_assign_pointer(task->real_cred, new);
	rcu_assign_pointer(task->cred, new);
	if (new->user != old->user)
		atomic_dec(&old->user->processes);
	alter_cred_subscribers(old, -2);

	/* send notifications */
	if (!uid_eq(new->uid,   old->uid)  ||
	    !uid_eq(new->euid,  old->euid) ||
	    !uid_eq(new->suid,  old->suid) ||
	    !uid_eq(new->fsuid, old->fsuid))
		proc_id_connector(task, PROC_EVENT_UID);

	if (!gid_eq(new->gid,   old->gid)  ||
	    !gid_eq(new->egid,  old->egid) ||
	    !gid_eq(new->sgid,  old->sgid) ||
	    !gid_eq(new->fsgid, old->fsgid))
		proc_id_connector(task, PROC_EVENT_GID);

	/* release the old obj and subj refs both */
	put_cred(old);
	put_cred(old);
	return 0;
}
EXPORT_SYMBOL(commit_creds);
```

다음과 같이 기존 cred 값 `rcu_assign_pointer(task->real_cred, new)` 및 `rcu_assign_pointer(task->cred, new)` 를 통해 새로운 `cred` 값을 덮어씌우는 것을 볼 수 있다. 
+ rcu_assign_pointer 는 첫번째 인자로 주어인 포인터를 두번째 인자로 주어진 포인터 값으로 할당해 주는 함수이다. 

## **4. prepare_kernel_cred**

`prepare_kernel_cred` 함수는 cred 구조체를 받아서 새로운 자격증명을 생성하는 함수이다. 

다음 그 함수의 코드를 보자. 

+ kernel/cred.c

```c
struct cred *prepare_kernel_cred(struct task_struct *daemon)
{
	const struct cred *old;
	struct cred *new;

	new = kmem_cache_alloc(cred_jar, GFP_KERNEL);
	if (!new)
		return NULL;

	kdebug("prepare_kernel_cred() alloc %p", new);

	if (daemon)
		old = get_task_cred(daemon);
	else
		old = get_cred(&init_cred);

	validate_creds(old);

	*new = *old;
	new->non_rcu = 0;
	atomic_set(&new->usage, 1);
	set_cred_subscribers(new, 0);
	get_uid(new->user);
	get_user_ns(new->user_ns);
	get_group_info(new->group_info);

#ifdef CONFIG_KEYS
	new->session_keyring = NULL;
	new->process_keyring = NULL;
	new->thread_keyring = NULL;
	new->request_key_auth = NULL;
	new->jit_keyring = KEY_REQKEY_DEFL_THREAD_KEYRING;
#endif

#ifdef CONFIG_SECURITY
	new->security = NULL;
#endif
	if (security_prepare_creds(new, old, GFP_KERNEL) < 0)
		goto error;

	put_cred(old);
	validate_creds(new);
	return new;

error:
	put_cred(new);
	put_cred(old);
	return NULL;
}
EXPORT_SYMBOL(prepare_kernel_cred);
```

다음 함수에서 유의할 부분은 다음이다. 

```c
if (daemon)
    old = get_task_cred(daemon);
else
    old = get_cred(&init_cred);
```

인자로 주어진 `daemon` 값이 `NULL` 이라면, `init_cred` 라는 구조체 값을 받아 새로운 자격 증명을 생성한다. 

그렇다면 `init_cred`는 어떤 값으로 초기화 되어 있을까?

+ kernel/cred.c

```c
struct cred init_cred = {
	.usage			= ATOMIC_INIT(4),
#ifdef CONFIG_DEBUG_CREDENTIALS
	.subscribers		= ATOMIC_INIT(2),
	.magic			= CRED_MAGIC,
#endif
	.uid			= GLOBAL_ROOT_UID,
	.gid			= GLOBAL_ROOT_GID,
	.suid			= GLOBAL_ROOT_UID,
	.sgid			= GLOBAL_ROOT_GID,
	.euid			= GLOBAL_ROOT_UID,
	.egid			= GLOBAL_ROOT_GID,
	.fsuid			= GLOBAL_ROOT_UID,
	.fsgid			= GLOBAL_ROOT_GID,
	.securebits		= SECUREBITS_DEFAULT,
	.cap_inheritable	= CAP_EMPTY_SET,
	.cap_permitted		= CAP_FULL_SET,
	.cap_effective		= CAP_FULL_SET,
	.cap_bset		= CAP_FULL_SET,
	.user			= INIT_USER,
	.user_ns		= &init_user_ns,
	.group_info		= &init_groups,
};
```

`uid` 및 `guid` 값을 보면 이 cred 구조체는 `root` 의 권한을 가진 구조체 라는 것을 알 수 있다!

따라서 `prepare_kernel_cred(NULL)` 을 호출하면 `root` 의 자격 증명을 할당 받을 수 있다는 것을 알 수 있다. 

## **Conclusion**

따라서 linux kernel 에서 code-execution 이 되는 상황이라면, `commit_creds(prepare_kernel_cred(NULL))` 호출을 통해 root 권한을 자신에게 할당 받을 수 있다. 

실제 linux kernel exploit 에서 code-execution premitive 달성시, 이러한 방법으로 루트 권한을 획득한다고 한다. 

## **References**

+ <https://cloudfuzz.github.io/android-kernel-exploitation/chapters/linux-privilege-escalation.html#process-credentials>
+ <https://wogh8732.tistory.com/308>
+ <https://learn.dreamhack.io/61#5>