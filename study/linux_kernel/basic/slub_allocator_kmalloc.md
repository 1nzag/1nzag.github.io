---
layout: simple
title: SLAB Allocator & kmalloc()
---

## **1. kmalloc**

일반적인 elf 에서 `malloc()` 과 같은 메모리 동적 할당기가 있듯이, 커널에서도 동적 메모리 할당기인 `kmalloc()` 함수가 존재한다. 

kmalloc 함수의 사용법은 다음과 같다. 

```c
#include <linux/slab.h>

void *kmalloc(size_t size, gfp_t flags);
```

+ **size**: 할당할 사이즈
+ **flags**: 메모리 할당의 우선순위나 할당 방법을 지정해 주는 flag

kmalloc 에서 사용하는 flag의 종류는 다음과 같다. 

+ **GFP_KERNEL**: 일반적인 커널 할당을 위해 사용하는 flag.
    + 메모리가 부족할 때, I/O 작업 및 swip 을 통해 추가 메모리를 확보하려고 시도한다. 
+ **GFP_ATOMIC**: 메모리 할당이 즉시 이루어지지 않으면 할당에 실패한다. 
    + I/O 작업, swip 등 추가적인 메모리 확보를 위한 행동을 하지 않는다. 
+ **GFP_USER**: 사용자 공간에서 페이지 할당을 시도한다. 
+ **GFP_HIGHUSER**: 사용자 공간에서 높은 메모리 영역을 할당한다. 
+ **GFP_NOIO / GFP_NORETRY / GFP_NOFS**: 각각 메모리 할당 시  I/O 작업, 재시도, 파일 시스템 호출을 방지한다. 
+ **GFP_DMA / GFP_DMA32**: 특정 DMA(Direct Memory Access) 영역에 대한 메모리 할당을 위해 사용된다. 

## **2. SLAB Allocator**

kmalloc에서 메모리를 할당할 때, 효율적으로 할당하기 위해서 kernel 은 `SLAB Allocator` 를 사용한다. 

예를 들어 작은 크기의 영역을 할당 할 때, 페이지 하나만큼의 크기를 할당 해 메모리 누수가 나는 것을 방지한다. 

커널을 빌드 할 때 설정에 따라 `SLAB`, `SLOB`, `SLUB` allocator 를 지정해 줄 수 있지만, defulat 설정은 `SLUB` allocator 를 사용하도록 되어있다. 

보통은 세가지중 하나로 빌드하기 떄문에 모든걸 `SLAB` 이라고 부르는 경우도 있는 것 같다. 

`SLUB` 은 여러개의 크기를 가진 `SLAB object` 와 그 object 들이 모인 `SLAB Page (Slabs)` 로 구성된다. 

![SLAB Objects](/assets/img/study/slub_allocator_kmalloc/slub_objects.png)

각각의 object 는 alloc 또는 free 된 상태로 존재하며, free 된 object 들은 `free list`로 관리된다.

`SLAB Page` 에는 동일한 크기의 `slab object`로 구성되어 있으며, 이러한 `SLAB page` 들이 여러개 모여 `slab cache`를 구성한다. 

![SLAB Page](/assets/img/study/slub_allocator_kmalloc/slub_page.png)

`kmalloc` 이 호출 될 때, 해당 함수는 `SLAB Cache`에 존재하는 `SLAB object`를 가져와 할당한다. 

당연히 효율을 위해 `SLAB cache`가 가지고 있는 `SLAB object`크기는 cache 마다 다양하며, 각각 다음과 같은 크기를 가진다. 

+ mm/slab_common.c

```c

#define INIT_KMALLOC_INFO(__size, __short_size)			\
{								\
	.name[KMALLOC_NORMAL]  = "kmalloc-" #__short_size,	\
	KMALLOC_RCL_NAME(__short_size)				\
	KMALLOC_CGROUP_NAME(__short_size)			\
	KMALLOC_DMA_NAME(__short_size)				\
	KMALLOC_RANDOM_NAME(RANDOM_KMALLOC_CACHES_NR, __short_size)	\
	.size = __size,						\
}

const struct kmalloc_info_struct kmalloc_info[] __initconst = {
	INIT_KMALLOC_INFO(0, 0),
	INIT_KMALLOC_INFO(96, 96),
	INIT_KMALLOC_INFO(192, 192),
	INIT_KMALLOC_INFO(8, 8),
	INIT_KMALLOC_INFO(16, 16),
	INIT_KMALLOC_INFO(32, 32),
	INIT_KMALLOC_INFO(64, 64),
	INIT_KMALLOC_INFO(128, 128),
	INIT_KMALLOC_INFO(256, 256),
	INIT_KMALLOC_INFO(512, 512),
	INIT_KMALLOC_INFO(1024, 1k),
	INIT_KMALLOC_INFO(2048, 2k),
	INIT_KMALLOC_INFO(4096, 4k),
	INIT_KMALLOC_INFO(8192, 8k),
	INIT_KMALLOC_INFO(16384, 16k),
	INIT_KMALLOC_INFO(32768, 32k),
	INIT_KMALLOC_INFO(65536, 64k),
	INIT_KMALLOC_INFO(131072, 128k),
	INIT_KMALLOC_INFO(262144, 256k),
	INIT_KMALLOC_INFO(524288, 512k),
	INIT_KMALLOC_INFO(1048576, 1M),
	INIT_KMALLOC_INFO(2097152, 2M)
};
```

하나의 cache 는 모두 동일한 크기의 object 를 가진다. 

## **3. Dive in to kmalloc()**

이제 직접 커널의 `kmalloc()` 함수를 보면서 어떻게 메모리를 할당하는지 확인해보자. 

다음은 `kmalloc()` 함수의 코드이다. 

+ include/linux/slab.h[^1]

```c
static __always_inline __alloc_size(1) void *kmalloc(size_t size, gfp_t flags)
{
	if (__builtin_constant_p(size) && size) {
		unsigned int index;

		if (size > KMALLOC_MAX_CACHE_SIZE)
			return kmalloc_large(size, flags);

		index = kmalloc_index(size);
		return kmalloc_trace(
				kmalloc_caches[kmalloc_type(flags, _RET_IP_)][index],
				flags, size);
	}
	return __kmalloc(size, flags);
}
```

[^1]: `__builtin_constant_p()` 매크로는 해당 변수가 컴파일 타임에 상수값이었는지 확인하는 매크로인 것을 참고하자.

size가 상수값일 때, 위로 분기하고, 아닌경우에는 아래로 분기한다. 

size 가 상수 값일 경우, 요청한 사이즈가 최대 캐시의 사이즈 보다 큰 지 검사하고, 크다면, `kmalloc_large()` 를 통해 메모리를 할당하고, 그렇지 않은 경우엔 `kmalloc_index()` 함수를 통해 

index를 할당받고, `kmalloc_trace()` 함수를 통해 동적 메모리를 할당하는 것을 볼 수 있다. 

앞서 설명했듯이, `SLUB` 은 작은 크기의 메모리를 할당할 때 효율을 올리기 위한 매커니즘이므로, `__kmalloc()` 함수 및 `kmalloc_trace()`를 집중적으로 볼 필요가 있다. 

그렇다면 `kmalloc_index()`는 무슨 함수이며, `kmalloc_trace()` 는 어떤함수일까. 

+ include/slab.h

```c
/*
 * Figure out which kmalloc slab an allocation of a certain size
 * belongs to.
 * 0 = zero alloc
 * 1 =  65 .. 96 bytes
 * 2 = 129 .. 192 bytes
 * n = 2^(n-1)+1 .. 2^n
 *
 * Note: __kmalloc_index() is compile-time optimized, and not runtime optimized;
 * typical usage is via kmalloc_index() and therefore evaluated at compile-time.
 * Callers where !size_is_constant should only be test modules, where runtime
 * overheads of __kmalloc_index() can be tolerated.  Also see kmalloc_slab().
 */
static __always_inline unsigned int __kmalloc_index(size_t size,
						    bool size_is_constant)
{
	if (!size)
		return 0;

	if (size <= KMALLOC_MIN_SIZE)
		return KMALLOC_SHIFT_LOW;

	if (KMALLOC_MIN_SIZE <= 32 && size > 64 && size <= 96)
		return 1;
	if (KMALLOC_MIN_SIZE <= 64 && size > 128 && size <= 192)
		return 2;
	if (size <=          8) return 3;
	if (size <=         16) return 4;
	if (size <=         32) return 5;
	if (size <=         64) return 6;
	if (size <=        128) return 7;
	if (size <=        256) return 8;
	if (size <=        512) return 9;
	if (size <=       1024) return 10;
	if (size <=   2 * 1024) return 11;
	if (size <=   4 * 1024) return 12;
	if (size <=   8 * 1024) return 13;
	if (size <=  16 * 1024) return 14;
	if (size <=  32 * 1024) return 15;
	if (size <=  64 * 1024) return 16;
	if (size <= 128 * 1024) return 17;
	if (size <= 256 * 1024) return 18;
	if (size <= 512 * 1024) return 19;
	if (size <= 1024 * 1024) return 20;
	if (size <=  2 * 1024 * 1024) return 21;

	if (!IS_ENABLED(CONFIG_PROFILE_ALL_BRANCHES) && size_is_constant)
		BUILD_BUG_ON_MSG(1, "unexpected size in kmalloc_index()");
	else
		BUG();

	/* Will never be reached. Needed because the compiler may complain */
	return -1;
}
static_assert(PAGE_SHIFT <= 20);
#define kmalloc_index(s) __kmalloc_index(s, true)
```

`kmalloc_index()` 함수는 입력받은 size에 따라 각각의 정해진 index 를 반환한다. 

그럼 다음 구문으로 오는 코드를 보자

```c
index = kmalloc_index(size);	
return kmalloc_trace(
		kmalloc_caches[kmalloc_type(flags, _RET_IP_)][index],
		flags, size);
```

`kmalloc_index()` 함수에서 전달받은 index로 `kmalloc_caches` 배열에 접근하여 그 포인터를 `kmalloc_trace()` 함수로 전달한다. 

`kmalloc_caches` 배열은 다음과 같이 구조체 포인터 배열로 선언되어있다. 

+ mm/slab_common.c

```c
struct kmem_cache *
kmalloc_caches[NR_KMALLOC_TYPES][KMALLOC_SHIFT_HIGH + 1] __ro_after_init =
{ /* initialization for https://bugs.llvm.org/show_bug.cgi?id=42570 */ };
EXPORT_SYMBOL(kmalloc_caches);
```

이제 `kmalloc_trace()` 함수를 살펴보자

+ mm/slab_common.c

```c
void *kmalloc_trace(struct kmem_cache *s, gfp_t gfpflags, size_t size)
{
	void *ret = __kmem_cache_alloc_node(s, gfpflags, NUMA_NO_NODE,
					    size, _RET_IP_);

	trace_kmalloc(_RET_IP_, ret, size, s->size, gfpflags, NUMA_NO_NODE);

	ret = kasan_kmalloc(s, ret, size, gfpflags);
	return ret;
}

```
+ mm/slab.c[^2]

```c
void *__kmem_cache_alloc_node(struct kmem_cache *cachep, gfp_t flags,
			     int nodeid, size_t orig_size,
			     unsigned long caller)
{
	return slab_alloc_node(cachep, NULL, flags, nodeid,
			       orig_size, caller);
}

/*...*/

static __always_inline void *
slab_alloc_node(struct kmem_cache *cachep, struct list_lru *lru, gfp_t flags,
		int nodeid, size_t orig_size, unsigned long caller)
{
	unsigned long save_flags;
	void *objp;
	struct obj_cgroup *objcg = NULL;
	bool init = false;

	flags &= gfp_allowed_mask;
	cachep = slab_pre_alloc_hook(cachep, lru, &objcg, 1, flags);
	if (unlikely(!cachep))
		return NULL;

	objp = kfence_alloc(cachep, orig_size, flags);
	if (unlikely(objp))
		goto out;

	local_irq_save(save_flags);
	objp = __do_cache_alloc(cachep, flags, nodeid);
	local_irq_restore(save_flags);
	objp = cache_alloc_debugcheck_after(cachep, flags, objp, caller);
	prefetchw(objp);
	init = slab_want_init_on_alloc(flags, cachep);

out:
	slab_post_alloc_hook(cachep, objcg, flags, 1, &objp, init,
				cachep->object_size);
	return objp;
}

```
[^2]: `unlikely()` 매크로는 리눅스 커널 상에서 거의 일어나지 않는 현상의 힌트로 준 것으로, 주로 예외처리를 할 때 쓰는 매크로인 것을 참고하자.


`__kmem_cache_alloc_node()` 함수를 호출하고, 그 값을 반환하는 것을 알 수 있다. 나머지 두개의 함수 호출은 디버깅 / trace 를 위한 호출이므로 크게 고려할 필요 없다. 

`__kmem_cache_alloc_node()` 함수는 바로 `slab_alloc_node()` 함수를 호출하고, `__do_cache_alloc()` 함수를 통해 할당받은 포인터를 반환한다. 

+ mm/slab.c

```c
static __always_inline void *
__do_cache_alloc(struct kmem_cache *cachep, gfp_t flags, int nodeid __maybe_unused)
{
	return ____cache_alloc(cachep, flags);
}

/*...*/

static inline void *____cache_alloc(struct kmem_cache *cachep, gfp_t flags)
{
	void *objp;
	struct array_cache *ac;

	check_irq_off();

	ac = cpu_cache_get(cachep);
	if (likely(ac->avail)) {
		ac->touched = 1;
		objp = ac->entry[--ac->avail];

		STATS_INC_ALLOCHIT(cachep);
		goto out;
	}

	STATS_INC_ALLOCMISS(cachep);
	objp = cache_alloc_refill(cachep, flags);
	/*
	 * the 'ac' may be updated by cache_alloc_refill(),
	 * and kmemleak_erase() requires its correct value.
	 */
	ac = cpu_cache_get(cachep);

out:
	/*
	 * To avoid a false negative, if an object that is in one of the
	 * per-CPU caches is leaked, we need to make sure kmemleak doesn't
	 * treat the array pointers as a reference to the object.
	 */
	if (objp)
		kmemleak_erase(&ac->entry[ac->avail]);
	return objp;
}

```
마지막으로 `cache_alloc_refill()` 에서 cache 의 정보를 통해 할당한 메모리의 주소를 반환한다.[^3]

[^3]: `cache_alloc_refill()` 함수까지 분석하는 것은 포스팅이 매우 길어질 것 같아 생략한다. 

즉, `kmalloc_trace()`는 `kmalloc_index()` 함수를 통해 사이즈별로 정해진 cache 를 받아 해당 cache에서 메모리를 할당 받아 반환하는 함수 라는 것을 알 수 있다. 

리눅스 버전별로 `kmalloc_trace()` 는 이름이 비슷한 다른 함수로도 표현이 되어있는 것 같다. 

대표적으로는 `kmem_cache_alloc_trace()` 함수 등이 있겠다. 

다음으로 인자가 상수가 아닐 때 호출되는 `__kmalloc()` 함수에 대해서 살펴보자. 

+ mm/slab_common.c

```c
static __always_inline
void *__do_kmalloc_node(size_t size, gfp_t flags, int node, unsigned long caller)
{
	struct kmem_cache *s;
	void *ret;

	if (unlikely(size > KMALLOC_MAX_CACHE_SIZE)) {
		ret = __kmalloc_large_node(size, flags, node);
		trace_kmalloc(caller, ret, size,
			      PAGE_SIZE << get_order(size), flags, node);
		return ret;
	}

	s = kmalloc_slab(size, flags, caller);

	if (unlikely(ZERO_OR_NULL_PTR(s)))
		return s;

	ret = __kmem_cache_alloc_node(s, flags, node, size, caller);
	ret = kasan_kmalloc(s, ret, size, flags);
	trace_kmalloc(caller, ret, size, s->size, flags, node);
	return ret;
}

void *__kmalloc_node(size_t size, gfp_t flags, int node)
{
	return __do_kmalloc_node(size, flags, node, _RET_IP_);
}
EXPORT_SYMBOL(__kmalloc_node);

void *__kmalloc(size_t size, gfp_t flags)
{
	return __do_kmalloc_node(size, flags, NUMA_NO_NODE, _RET_IP_);
}
EXPORT_SYMBOL(__kmalloc);
```

함수를 따라가 추적하다보면 결과적으로 `__do_kmalloc_node()` 함수를 호출하는 것을 알 수 있는데, 해당 함수에서는 사이즈가 최대 캐시 보다 클 경우, `__kmalloc_large_node()` 함수를 통해 

동적 메모리를 할당하고, 아닌 경우  `kmalloc_slab()` 함수로 받은 `s` 인자를 `__kmem_cache_alloc_node` 함수로 전달하고, 그 값을 반환하는 것을 볼 수 있다. 

`__kmem_cache_alloc_node()` 함수는 우리가 앞서 봤던 `kmalloc_trace()` 함수가 쓰는 동일한 함수다. 

역시 우리는 `__kmalloc_large_node()` 호출은 고려하지 않는다. 

먼저 `kmalloc_slab()` 함수를 보자

+ mm/slab_common.c

```c
static inline unsigned int size_index_elem(unsigned int bytes)
{
	return (bytes - 1) / 8;
}

/*
 * Find the kmem_cache structure that serves a given size of
 * allocation
 */
struct kmem_cache *kmalloc_slab(size_t size, gfp_t flags, unsigned long caller)
{
	unsigned int index;

	if (size <= 192) {
		if (!size)
			return ZERO_SIZE_PTR;

		index = size_index[size_index_elem(size)];
	} else {
		if (WARN_ON_ONCE(size > KMALLOC_MAX_CACHE_SIZE))
			return NULL;
		index = fls(size - 1);
	}

	return kmalloc_caches[kmalloc_type(flags, caller)][index];
}

```

위의 `kmalloc_index` 와 같이 사이즈에 적합한 캐시를 가져오는 것을 알 수 있다. 즉, 그 뒤의 매커니즘은 위에서 상술한 것과 비슷하게 흘러가는 것을 알 수 있다. 

## **Conclusion**

커널은 메모리를 동적 할당 할 때, `SLAB cache` 를 활용하여 더 효율적이고 빠른 할당을 수행 할 수 있다. 

아직 `kmem_cache` 구조체에 대해서 제대로 분석하지 않았고, `__kmem_cache_alloc_node()` 함수의 정확한 매커니즘도 완벽하게 이해하지 못했기에, 필요하게 되면 포스트를 보충할 예정이다. 

중간에 나오는 함수의 호출들은 은근 커널 드라이버에서 발견되는 함수들 인 것 같다. 

사실 이 포스트를 쓸 때도 분석하는 커널 드라이버에서 사용하는 `kmem_cache_alloc_trace()` 함수가 무슨 함수인지 궁금해서 이것저것 찾아보다가 쓰게 된 것이다. 

## **References**

+ <https://www.kernel.org/>
+ <https://jiravvit.tistory.com/entry/linux-kernel-4-%EC%8A%AC%EB%9E%A9%ED%95%A0%EB%8B%B9%EC%9E%90>

## comments