---
layout: simple
title: SLUB Allocator & kmalloc()
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

## **2. SLUB Allocator**

kmalloc에서 메모리를 할당할 때, 효율적으로 할당하기 위해서 kernel 은 `SLUB Allocator` 를 사용한다. 

예를 들어 작은 크기의 영역을 할당 할 때, 페이지 하나만큼의 크기를 할당 해 메모리 누수가 나는 것을 방지한다. 

커널을 빌드 할 때 설정에 따라 `SLAB`, `SLOB`, `SLUB` allocator 를 지정해 줄 수 있지만, defulat 설정은 `SLUB` allocator 를 사용하도록 되어있다. 

`SLUB` 은 여러개의 크기를 가진 `SLUB object` 와 그 object 들이 모인 `SLUB Page (Slabs)` 로 구성된다. 

![SLUB Objects](/assets/img/study/slub_allocator_kmalloc/slub_objects.png)

각각의 object 는 alloc 또는 free 된 상태로 존재하며, free 된 object 들은 `free list`로 관리된다.

`SLUB Page` 에는 동일한 크기의 `slub object`로 구성되어 있으며, 이러한 `SLUB page` 들이 여러개 모여 `slub cache`를 구성한다. 

![SLUB Page](/assets/img/study/slub_allocator_kmalloc/slub_page.png)

`kmalloc` 이 호출 될 때, 해당 함수는 `SLUB Cache`에 존재하는 `SLUB object`를 가져와 할당한다. 

당연히 효율을 위해 `SLUB cache`가 가지고 있는 `SLUB object`크기는 cache 마다 다양하며, 각각 다음과 같은 크기를 가진다. 

+ mm/slab_common.c
```c
const struct kmalloc_info_struct kmalloc_info[] __initconst = {
	{NULL,                      0},		{"kmalloc-96",             96},
	{"kmalloc-192",           192},		{"kmalloc-8",               8},
	{"kmalloc-16",             16},		{"kmalloc-32",             32},
	{"kmalloc-64",             64},		{"kmalloc-128",           128},
	{"kmalloc-256",           256},		{"kmalloc-512",           512},
	{"kmalloc-1024",         1024},		{"kmalloc-2048",         2048},
	{"kmalloc-4096",         4096},		{"kmalloc-8192",         8192},
	{"kmalloc-16384",       16384},		{"kmalloc-32768",       32768},
	{"kmalloc-65536",       65536},		{"kmalloc-131072",     131072},
	{"kmalloc-262144",     262144},		{"kmalloc-524288",     524288},
	{"kmalloc-1048576",   1048576},		{"kmalloc-2097152",   2097152},
	{"kmalloc-4194304",   4194304},		{"kmalloc-8388608",   8388608},
	{"kmalloc-16777216", 16777216},		{"kmalloc-33554432", 33554432},
	{"kmalloc-67108864", 67108864}
};
```

하나의 cache 는 모두 동일한 크기의 object 를 가진다. 

## **3. Dive in to kmalloc()**

이제 직접 커널의 `kmalloc()` 함수를 보면서 어떻게 메모리를 할당하는지 확인해보자. 

다음은 `kmalloc()` 함수의 코드이다. 

+ inclde/slab.h
```c
static __always_inline void *kmalloc(size_t size, gfp_t flags)
{
	if (__builtin_constant_p(size)) {
		if (size > KMALLOC_MAX_CACHE_SIZE)
			return kmalloc_large(size, flags);
#ifndef CONFIG_SLOB
		if (!(flags & GFP_DMA)) {
			int index = kmalloc_index(size);

			if (!index)
				return ZERO_SIZE_PTR;

			return kmem_cache_alloc_trace(kmalloc_caches[index],
					flags, size);
		}
#endif
	}
	return __kmalloc(size, flags);
}
```

우리는 defualt config 인 `SLUB` 에 대해서 분석할 것이므로, `SLOB` 과 관련된 정의 부분은 제외하고 분석한다. 

먼저 요청한 사이즈가 최대 캐시의 사이즈 보다 큰 지 검사하고, 크다면, `kmalloc_large()` 를 통해 메모리를 할당하고, 그렇지 않은 경우엔 `__kmalloc()` 을 통해 메모리를 할당한다. 

앞서 배웠듯이, `SLUB` 은 작은 크기의 메모리를 할당할 때 효율을 올리기 위한 매커니즘이므로, `__kmalloc()` 함수를 볼 집중적으로 볼 필요가 있다. 

+ include/slab.c[^1]

```c

/**
 * __do_kmalloc - allocate memory
 * @size: how many bytes of memory are required.
 * @flags: the type of memory to allocate (see kmalloc).
 * @caller: function caller for debug tracking of the caller
 */
static __always_inline void *__do_kmalloc(size_t size, gfp_t flags,
					  unsigned long caller)
{
	struct kmem_cache *cachep;
	void *ret;

	if (unlikely(size > KMALLOC_MAX_CACHE_SIZE))
		return NULL;
	cachep = kmalloc_slab(size, flags);
	if (unlikely(ZERO_OR_NULL_PTR(cachep)))
		return cachep;
	ret = slab_alloc(cachep, flags, caller);

	ret = kasan_kmalloc(cachep, ret, size, flags);
	trace_kmalloc(caller, ret,
		      size, cachep->size, flags);

	return ret;
}

void *__kmalloc(size_t size, gfp_t flags)
{
	return __do_kmalloc(size, flags, _RET_IP_);
}
EXPORT_SYMBOL(__kmalloc);
```

[^1]: 코드에는 `unlikely()` 매크로가 보이는데, 리눅스 커널 상에서 거의 일어나지 않는 현상의 힌트로 준 것으로, 주로 예외처리를 할 때 쓰는 매크로인 것을 참고하자.


바로 `__do_kmalloc()` 을 호출 하는 것을 알 수 있고, `__do_kmalloc()` 함수에서는 `kmalloc_slab()` 을 통해 포인터를 할당 받고, 할당받은 포인터를 `slab_alloc()` 함수에 전달한다. 

그렇다면 `kmalloc_slab()` 함수는 무엇을 하고, `slab_alloc()` 함수는 무엇을 할까.

먼저 `kmalloc_slab()` 함수를 보자

```c
struct kmem_cache *kmalloc_caches[KMALLOC_SHIFT_HIGH + 1];
EXPORT_SYMBOL(kmalloc_caches);

/*...*/

static inline int size_index_elem(size_t bytes)
{
	return (bytes - 1) / 8;
}

/*
 * Find the kmem_cache structure that serves a given size of
 * allocation
 */
struct kmem_cache *kmalloc_slab(size_t size, gfp_t flags)
{
	int index;

	if (size <= 192) {
		if (!size)
			return ZERO_SIZE_PTR;

		index = size_index[size_index_elem(size)];
	} else {
		if (unlikely(size > KMALLOC_MAX_CACHE_SIZE)) {
			WARN_ON(1);
			return NULL;
		}
		index = fls(size - 1);
	}

#ifdef CONFIG_ZONE_DMA
	if (unlikely((flags & GFP_DMA)))
		return kmalloc_dma_caches[index];

#endif
	return kmalloc_caches[index];
}

```

요청한 사이즈 별로 `kmalloc_caches` 에서 캐시를 가져오는 함수 임을 알 수 있다. 
이때 반환하는 값은 `kmem_cache` 구조체이다. 



## **References**

+ <https://jiravvit.tistory.com/entry/linux-kernel-4-%EC%8A%AC%EB%9E%A9%ED%95%A0%EB%8B%B9%EC%9E%90>

## comments