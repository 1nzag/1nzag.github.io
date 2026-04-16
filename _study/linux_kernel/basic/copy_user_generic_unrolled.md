---
layout: simple
title:  copy_user_generic_unrolled() function
---

## **What is copy_user_generic_unrolled() function?**

이 함수는 유저 공간과 커널 공간의 메모리를 상호간에 복사 하기 위해 사용하는 함수이다. 

사용자 공간에서 커널 공간으로 데이터를 복사 할 수 있다.

반대로 커널 공간에서 사용자 공간으로의 데이터 복사도 가능하다. 

함수의 정의는 다음과 같다. 

```c
copy_user_generic_unrolled(void * to, const void * from, unsigned long n)
```

## **Vulnerabilities**

이 함수를 쓸 떄 가장 유의해야 할 점은 **복사 되는 데이터 공간과 복사할 데이터 공간을 검증하지 않는다**는 것이다.

즉, `to` 의 메모리 공간이 커널 영역인지, 유저영역인지 구분을 하지 않는다는 것이다. 반대로 `from` 도 마찬가지이다. 

또한, 함수를 호출 할때 메모리 예외를 발생시키지 않는다는 특성이 있다. 

만약 `to` 와 `from` 을 공격자가 컨트롤 할 수 있으면, AAR, AAW 등의 상황이 가능 할 수 있다. 

## **Conclusion**
 
이 함수는 `K3RN3L CTF 2021` 과 `LINE CTF 2021`에서 코드를 분석하다가 알게 된 함수이다. 

실제 커널 개발 환경에서 이러한 함수를 쓸지는 모르겠지만, 알아두는게 좋을 것 같아 포스트한다. 

## **References**

+ <https://github.com/theori-io/ctf/blob/master/2021/linectf/LINE%20CTF%202021%20Write%20Up%20-%20The%20Duck.pdf>
