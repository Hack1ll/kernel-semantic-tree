# slab allocator

## 한 문장 정의

slab allocator는 page를 작은 object slot으로 나누어 `task_struct`, `file`, socket, driver object 같은 kernel heap object를 빠르게 할당하고 재사용하는 계층이다.

## 왜 중요한가

커널은 작은 객체를 매우 자주 만들고 없앤다. 매번 page allocator에서 page를 직접 받고 object 배치를 새로 계산하면 비용이 크다. slab allocator는 같은 크기나 같은 타입의 객체를 cache 단위로 관리해 빠르게 빌려주고 다시 받는다.

취약점 분석에서 slab allocator가 중요한 이유는 해제된 object slot이 다시 다른 object로 재사용될 수 있기 때문이다.

```text
kmalloc()
    -> size class 선택
    -> free object slot 반환
    -> caller가 구조체로 사용
    -> kfree()
    -> slot이 cache로 돌아감
    -> 이후 allocation에서 재사용 가능
```

## page allocator와의 차이

page allocator는 page 단위로 memory를 준다. slab allocator는 그 page 안을 object slot으로 나눈다.

```text
page allocator
    -> page 단위
    -> 물리 메모리 연속성과 GFP 조건이 핵심

slab allocator
    -> object 단위
    -> size class, cache, reuse가 핵심
```

`kmalloc(64)`를 호출하면 일반적으로 64 byte에 맞는 cache에서 object slot을 받는다. 전용 cache를 쓰는 타입은 `kmem_cache_alloc()` 경로로 할당될 수 있다.

## 일반 API

자주 만나는 API는 다음과 같다.

- `kmalloc(size, gfp)`: 초기화되지 않은 kernel heap memory를 할당한다.
- `kzalloc(size, gfp)`: 0으로 초기화된 memory를 할당한다.
- `kcalloc(n, size, gfp)`: 배열 크기 overflow를 고려해 0 초기화 memory를 할당한다.
- `kfree(ptr)`: object slot을 allocator에 반환한다.
- `kmem_cache_create()`: 특정 타입을 위한 전용 cache를 만든다.
- `kmem_cache_alloc()`: 전용 cache에서 object를 할당한다.

`kmalloc()`으로 받은 memory가 항상 0이라고 가정하면 안 된다. 초기화가 필요하면 `kzalloc()`이나 명시적 초기화가 필요하다.

## object reuse

slab allocator의 핵심은 reuse다. `kfree()`가 호출되면 pointer 값은 여전히 변수에 남아 있을 수 있지만, 그 memory는 더 이상 caller의 소유가 아니다.

이후 같은 cache나 같은 size class에서 새 allocation이 일어나면 같은 주소가 다른 객체로 쓰일 수 있다.

```text
object A 할당
    -> pointer p 저장
    -> object A 해제
    -> object B가 같은 slot 재사용
    -> p로 접근하면 B memory를 A로 해석
```

이 구조 때문에 use-after-free, type confusion, stale pointer 문제가 slab에서 자주 보인다.

## size와 overflow

allocation size가 user input에서 오면 계산 과정을 봐야 한다.

- `count * sizeof(struct item)`이 overflow되지 않는가?
- header 크기와 payload 크기를 더할 때 overflow되지 않는가?
- 할당한 크기보다 copy size가 크지 않은가?
- flexible array member를 포함한 구조체 크기를 정확히 계산하는가?
- object 끝의 padding이나 redzone을 data 영역으로 착각하지 않는가?

작은 계산 실수가 heap overflow나 out-of-bounds read/write로 이어진다.

## cache와 타입

전용 slab cache는 특정 타입의 생성과 해제를 모아 관리한다. 일반 `kmalloc` cache는 크기별로 객체를 섞어 관리할 수 있다.

취약점 분석에서 확인할 점은 다음과 같다.

- 해제된 slot이 같은 타입으로만 재사용되는가?
- 다른 타입이 같은 size class를 통해 같은 주소에 올 수 있는가?
- object 생성 시 모든 필드가 초기화되는가?
- destroy path가 list, timer, callback, refcount를 먼저 끊는가?

## 코드에서 확인할 것

1. 할당 크기는 고정인가, user input으로 결정되는가?
2. allocation API가 zero initialization을 보장하는가?
3. `kfree()` 이후 pointer를 다시 사용하지 않는가?
4. error path에서 같은 pointer를 두 번 free하지 않는가?
5. object가 list나 callback에 남아 있는 상태에서 free되지 않는가?
6. 같은 slot이 어떤 타입으로 재사용될 수 있는가?

## 보안 관점

slab allocator 주변에서 자주 보는 취약점은 다음과 같다.

- use-after-free: 해제된 object pointer를 계속 사용한다.
- double free: 같은 object slot을 두 번 반환한다.
- heap overflow: 할당된 object 경계를 넘어 쓴다.
- uninitialized memory leak: 초기화되지 않은 field를 user space로 복사한다.
- type confusion: 재사용된 memory를 이전 타입으로 해석한다.
- refcount bug: 마지막 reference가 아닌데 free하거나, 마지막 reference 뒤에도 접근한다.

## 다른 문서와의 연결

- [page allocator](page-allocator.md): slab cache가 backing page를 얻는 계층
- [user/kernel memory separation](user-kernel-memory-separation.md): slab object 내용이 user space로 복사되는 경계
- [4. Object Lifetime](../4-object-lifetime/README.md): refcount, ownership, cleanup path
- [5. Concurrency](../5-concurrency/README.md): 동시에 같은 object를 만지는 race

## 기억할 문장

slab allocator를 볼 때 핵심은 “이 주소가 지금 누구의 object인가”와 “해제된 slot이 언제 무엇으로 재사용되는가”다.
