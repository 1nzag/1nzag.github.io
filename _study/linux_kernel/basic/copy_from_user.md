---
layout: simple
title: copy_from_user() & copy_to_user() function
---

## 1.**What is copy_from_user() ?**

`copy_from_user()` 함수는 리눅스 커널 내부에서 쓰이는 함수다. 

보통 메모리 공간은 커널 모드와 사용자 모드로 각각 분리되어 격리되고 있고, 사용자 공간에서 실행되는 코드는 커널 공간에 직접 접근하지 못하고, 커널 공간에서 실행되는 코드는 사용자 공간에 직접 접근하지 못한다. 

따라서 커널 공간에서 사용자 공간에 있는 메모리에 안전하게 접근하기 위해 사용하는 함수가 필요한데, 그러한 역할을 해 주는 대표적인 함수가 바로 `copy_from_user()` 함수이다. 

`copy_from_user()` 함수는 **사용자 공간에 있는 데이터를 커널 공간의 메모리로 복사하는 함수이다.**

함수의 정의는 다음과 같다. 

```c
unsigned long copy_from_user (void * to, const void __user * from, unsigned long n);
```

각 인자는 다음과 같다. 

+ **to**: 커널 공간의 대상 메모리 주소
+ **from**: 사용자 공간의 소스 메모리 주소
+ **n**: 복사할 바이트 수

## **What is copy_to_user() ?**

`copy_to_user()` 함수는 반대로 **커널 공간에 존재하는 데이터를 사용자 공간의 메모리로 복사하는 함수이다.**

함수의 정의는 다음과 같다. 

```c
unsigned long copy_to_user (void __user * to, const void * from, unsigned long n);
```

각 인자는 다음과 같다. 

+ **to**: 사용자 공간의 대상 메모리 주소
+ **from**: 커널 공간의 소스 메모리 주소
+ **n** 복사할 바이트 수

## **References**

+ <https://www.cs.bham.ac.uk/~exr/lectures/opsys/12_13/docs/kernelAPI/r4081.html>
+ <https://www.cs.bham.ac.uk/~exr/lectures/opsys/13_14/docs/kernelAPI/r4037.html>